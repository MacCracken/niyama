# 0012 — pcre keeps its backtrack state on an explicit heap stack

**Status**: Accepted
**Date**: 2026-10-08 (niyama 1.1.0)

## Context

Through 1.0.13, `_pcre_match_run` was a recursive backtracker. Every SPLIT
recursed for its first arm and every SAVE recursed so that it could restore
the slot on the way back, so the native call depth grew with the **subject**,
not with the pattern. The frames were large (measured at ~20 KB in 1.0.9; the
8 MB default stack faulted somewhere past ~400 of them), so 1.0.9 capped the
depth at `PCRE_MAX_DEPTH = 256` and a quantifier that had to cover more than
~250 positions gave up:

```
a*$       over 300 × 'a'          → no match            (correct: match)
(a){200}  over 200 × 'a'          → no match            (correct: match)
.*z       over 300 × 'a' then 'z' → search answers 46   (correct: 0)
a+z       over 300 × 'a' then 'z' → search answers 45   (correct: 0)
```

The search answers were worse than a false negative: a start that gave up was
treated as a start that cannot match, so `niyama_pcre_search_at` moved on and
reported a later start as the leftmost match. 1.0.9 added
`PCRE_E_DEPTH_EXCEEDED = 12` as the mitigation, but the matcher wrote it to
`_pcre_err` and `niyama_pcre_last_error()` read `_pcre_last_err`, which only
compile writes, so no caller ever saw it.

Raising the bound was not a fix — it traded the false negative for a stack
overflow — and the roadmap held the real fix, replacing native recursion with
an explicit stack, for the next minor.

Writing it surfaced a second dependency on the bound. Nothing in the engine
stopped a loop whose body matches the empty string (`(a*)*b`, `(a?)+`, `\b*`,
a quantified lookaround) or a recursion that re-enters itself at the same
position (`(?R)?a`); the depth bound cut both off, and the matcher fell back to
the alternative the deepest frame had left. That fallback gave the right
answer for `(a*)*b` by accident and wrong answers elsewhere — on a two-byte
subject, `(?<!.)*b??` against `"ba"` ended at 1 where PCRE2 ends at 0, because
the empty loop had used up the depth the rest of the pattern needed. Without
the bound, both kinds of loop would run until the step limit and answer
"no match".

## Decision

**The pcre matcher is one loop over an explicit, process-lifetime backtrack
stack. Matching depth no longer depends on the subject; the step limit stays
the only DoS bound; and two explicit rules replace what the depth bound did by
accident.**

### The stack

One 8-byte word per entry, the kind in the low two bits:

| Entry | Layout | Popped while backtracking |
|---|---|---|
| CHOICE | `pc << 2 \| pos << 16` | resume at `(pc, pos)` |
| UNDO | `idx << 2 \| (prev + 1) << 16` | `saves[idx] = prev`, keep popping |
| MARK (3 words) | `[prev mark + 1] [entry pos] [type, continuation pc, previous recursion stop, recursion group]` | the construct's body ran out of alternatives |

SPLIT pushes a CHOICE for its second arm; SAVE pushes an UNDO (none when the
stack is empty, where a failure ends the run anyway). A lookahead, negative
lookahead, lookbehind, negative lookbehind, atomic group or recursion pushes a
MARK and runs its body in place. The body's terminator — `LOOKAHEAD_END`,
`ATOMIC_END`, or reaching the recursion's stop pc — resolves the innermost
MARK:

- a positive lookaround or atomic group **commits**: the CHOICEs above the
  MARK are dropped (nothing backtracks into it) and its UNDOs are compacted
  down over it, so a later backtrack past the construct still restores the
  captures it set; a positive lookaround then resets slot 0 (the 1.0.9 C8 rule
  for `\K`);
- a recursion, and a negative lookaround whose body matched, **unwind**: every
  UNDO above the MARK is applied, which restores exactly the slots the body
  changed — what the 1.0.9 snapshot restore did.

That is the recursive matcher's contract construct for construct: each was a
nested call that returned its first success.

### Bounds

An instruction pushes at most three words, so a match never holds more than
3 × the step limit. Under the default 1,000,000-step limit that is ~24 MB, below
the stack's hard ceiling of 2^24 words (128 MB): the step limit is the bound a
caller meets. The ceiling binds only when a caller raises the step limit past
~5.5M or `alloc()` refuses to grow the stack, and then the operation **fails
closed**: no answer, `PCRE_E_DEPTH_EXCEEDED` (its frozen value, 12) from
`niyama_pcre_last_error()`, and `niyama_pcre_search_at` stops instead of trying
the next start.

### Allocation

`_pcre_lazy_init` allocates 4,096 words; the matcher doubles the stack when it
is full. `alloc()` never frees, so a growth abandons the old buffer, but growth
happens only when a match goes deeper than every earlier one in the process:
everything ever allocated stays under twice the deepest match's stack, and
nothing is allocated per call or per backtrack point (the 1.0.9 C2 class).
Measured: 50 repeated searches of `.*z` over 20,000 bytes allocate 0 bytes
after the first.

### Empty iterations

An iteration of a quantified body that matched the empty string ends the loop,
and matching goes on from the loop's exit with that iteration's captures —
PCRE's rule, and the answer PCRE2 10.49 gives for every case in the test suite.
Only a loop whose body *can* match empty is instrumented, so `.*`, `a+` and
`[a-z]+` cost nothing: at compile time `_pcre_body_nullable` walks the body
(conservatively — a backreference, a recursion or an atomic group counts as
possibly empty), and a nullable loop gets a loop id in the unused `small` field
(bits 8..21) of its loop instructions:

- `X*` and `{n,}`: the head SPLIT records the iteration's start position in a
  loop slot; the back-edge JMP compares and, on no progress, goes to the loop's
  exit instead of the head.
- `X+`: the closing SPLIT (bit 22) compares with the slot and either exits or
  records and loops.

Loop slots sit after the 20 capture slots in the matcher's `saves[]`, so the
same UNDO entry restores them on backtrack. The opcode numbers and the frozen
encodings are unchanged; every relocation helper keeps the low 32 bits of an
instruction word, so the id survives shifts and `{n,m}` copies.

### Recursion loops

A recursion into a group that the nearest open recursion into the same group
entered at the same position can only repeat itself, so that branch fails.
PCRE2 stops the same case with `PCRE2_ERROR_RECURSELOOP`; niyama has no error
code for it, and a new one would extend the frozen surface (ADR 0010), so the
branch fails as the depth bound's fallback made it fail.

## Consequences

- **Positive** — the false negatives and wrong match starts are gone at any
  subject length the step limit allows: `a*$` over 300 bytes, `(a){200}`,
  `^a*z$` over 1,000 bytes, `.*z` over 20,000 bytes, `(?R)` nested 400 deep.
- **Positive** — faster. Same-boot A/B, 5 interleaved rounds, 13 pcre bench
  rows: mean −22.1%, median −27.1%, none slower beyond the ±5% noise margin
  (`docs/benchmarks.md` § v1.1.0). The recursive matcher paid a call and a
  160-byte snapshot copy at every SPLIT.
- **Positive** — a depth failure is now reported and fails closed, as 1.0.9
  documented and never delivered.
- **Negative** — answers change where 1.0.13 depended on the depth bound, by
  design. A differential run of the 1.0.13 matcher against this one (21,000
  random patterns × 16 short subjects, match and search) changed only answers
  that 1.0.13 had given up on at the step limit, answers to patterns PCRE2
  rejects or stops with RECURSELOOP, and answers where the new result equals
  PCRE2's and 1.0.13's did not. Over the 514,804 runs that stayed under 256
  steps — far from the old bound — every step count is identical.
- **Negative** — a frozen capacity row changes: ADR 0010's "pcre depth-limit
  256 (not configurable)" no longer exists. `PCRE_MAX_DEPTH` and
  `_pcre_depth_limit` are removed; no consumer in the ecosystem names either.
  This is why the release is a minor.
- **Neutral** — the backtrack stack can hold up to 128 MB for a caller that
  raises the step limit and feeds a long subject; under the default limit it
  stays under ~24 MB, and 32 KB until a match needs more.
- **Neutral** — a left recursion that PCRE2 reports as an error answers by
  failing the looping branch here. If a consumer needs the distinction, it is a
  post-fold additive error code in cyrius stdlib's `lib/niyama.cyr`.

## Alternatives considered

- **Raise `PCRE_MAX_DEPTH`.** Rejected in 1.0.9 and again here: the frames are
  native, so a higher bound trades a false negative for a stack overflow.
- **Keep native recursion only for the scoped constructs** (lookaround, atomic,
  recursion) and iterate SPLIT and SAVE. Rejected: recursion nests with the
  subject too (`(?R)` over balanced input), so it would keep a depth bound that
  input can reach.
- **A full saves snapshot per CHOICE** instead of UNDO entries. Rejected: 160
  bytes per choice point against 8, and the copy the recursive matcher paid at
  every SPLIT.
- **Fail an empty iteration** (ECMAScript's rule) instead of ending the loop
  with it. Rejected: PCRE ends the loop and keeps the iteration's captures, and
  so did 1.0.13's accidental fallback; `(a?)*b` keeps group 1 empty at 2.
- **New opcodes for the loop checks.** Rejected: ADR 0010 freezes the opcode
  numbering for niyama v1.x; an operand in the unused `small` field changes no
  frozen value.
- **A depth bound on recursion alone.** Rejected: it is the same class of
  limit this ADR removes, and the nearest-same-group check needs no bound.
