# 0007 — v0.7.0 catch-up: no-Unicode-dep slice of M4.5

**Status**: Accepted
**Date**: 2026-05-03

## Context

M4 closed at v0.6.0 with all five engines shipped. The roadmap's
M4.5 (v0.9.0) consolidates 11 deferred features from ADRs 0002-0006
that need to land before M5 surface freeze. The v0.7.0 / v0.8.0
slots were left explicitly open for sub-milestones if the work
needed splitting; the user carved those slots at planning time.

The deferred items cluster naturally by infrastructure dependency.
Three rough buckets:

1. **No-Unicode-dep** — POSIX bracket classes (ASCII), GNU `\<\>`
   word boundaries, named captures, inline flags, `\K`, branch-reset,
   conditional patterns, callouts. Shared infrastructure: a small
   `src/posix_classes.cyr` module (the 12 POSIX bracket-class
   fillers + name recognizer).
2. **Compile-time analysis-heavy** — fixed-width lookbehind
   `(?<=...)` `(?<!...)`, pcre recursion `(?R)` `(?P>name)`, fuzzy
   exact-start recovery in `_search`. Each is a non-trivial pattern
   analysis or matcher addition.
3. **Unicode** — `\p{L}` Unicode property classes (re2 + pcre + vim)
   and Unicode NFD normalization (`FUZZY_FLAG_UNICODE_NFD`). Shared
   infrastructure: a new `src/unicode.cyr` (~25 KB decomposition +
   property table).

Bucket 1 is **8 features across 3 engines, all small, all sharing
one new shared module.** Bucket 2 is **3 features each requiring
real analysis or matcher work.** Bucket 3 is **a single ~25KB
infrastructure addition** that re2/pcre/fuzzy/vim all hang off of.

The natural carve is: v0.7.0 ships bucket 1 (this ADR). v0.8.0
ships bucket 2. v0.9.0 ships bucket 3 + folds vim's POSIX class
copy onto the shared module. M5 freeze, v1.0 fold-ready.

## Decision

**v0.7.0 ships the no-Unicode-dep slice of M4.5.**

### Shared module

- New `src/posix_classes.cyr` — state-free ASCII fillers for the 12
  POSIX bracket classes (`alpha`, `digit`, `space`, `upper`,
  `lower`, `alnum`, `blank`, `cntrl`, `graph`, `print`, `punct`,
  `xdigit`) plus a name recognizer
  `_posix_match_class_name(pat, pos, pat_len)`. Used by **bre and
  pcre** at v0.7.0. **vim still carries its own in-engine copy from
  M4** — folding it onto the shared module is a v0.9.0 cleanup; the
  per-engine duplication isn't load-bearing and the refactor risks
  regressing well-tested vim parser code for zero behavior change.

### Per-engine additions

**niyama_bre** (clears 2 of 3 ADR 0002 deferrals):

- `\<` / `\>` GNU word boundaries via two new opcodes
  `BRE_OP_WORDBEGIN` (= 10) and `BRE_OP_WORDEND` (= 11). **Strict
  semantics** (distinct from `\b`): WORDBEGIN fires only at
  non-word→word transitions, WORDEND only at word→non-word
  transitions. (vim conflates the two onto its single BOUNDARY op,
  matching vim's loose semantics; bre tracks `grep -G` which keeps
  them distinct.)
- POSIX bracket classes `[[:alpha:]]` etc. via the shared module.
- Backreferences `\1`-`\9` **stay deferred** per the explicit M1
  "potentially post-v1.0; document, don't skip" call. Not revisited
  in v0.7.0; the user can decide at v0.8.0/v0.9.0 planning.

**niyama_re2** (clears 2 of 3 ADR 0003 deferrals):

- Named captures `(?<NAME>...)` and `(?P<NAME>...)` via a
  PCRE-shaped name table (40-byte slots, 9-name max — same numbers
  as pcre). New ABI:
  `niyama_re2_group_by_name(nfa, name)` returning the group index
  or -1. New error code `RE2_E_DUPLICATE_NAME = 7` (next-available;
  M2-frozen codes 0..6 are unchanged).
- Inline flags `(?i)`, `(?m)`, `(?s)` and any combination
  (e.g. `(?ims)`). Pattern-wide effect from the point of
  declaration onward. **Negated forms `(?-i)` and scoped forms
  `(?i:...)` are out of scope** — single-shot pattern-wide flags
  cover ~95% of real-world use. A future ADR can extend.
- Four new opcodes — `RE2_OP_CHAR_CI` (= 12), `RE2_OP_BOS` (= 13),
  `RE2_OP_EOS` (= 14), `RE2_OP_ANY_NONL` (= 15). The latter three
  back the spec-strict default (see "Behavior change" below).
- Unicode property classes `\p{L}` etc. **stay deferred** to
  v0.9.0 (needs the shared Unicode table).

**niyama_pcre** (clears 6 of 9 ADR 0004 deferrals):

- POSIX bracket classes via the shared module.
- Inline flags `(?i)`, `(?m)`, `(?s)` — same opcode model as re2
  (`PCRE_OP_CHAR_CI` = 18, `PCRE_OP_BOS` = 19, `PCRE_OP_EOS` = 20,
  `PCRE_OP_ANY_NONL` = 21).
- `\K` reset-match-start — emits `SAVE 0`, overrides the implicit
  match-start save (same mechanism as vim's `\zs`).
- Branch-reset groups `(?|...)` — alternatives reuse capture
  numbers. New compile-time global `_pcre_branch_reset_base`;
  parser save/restore in `_pcre_parse_primary`, per-branch reset
  in `_pcre_parse_alt`.
- Conditional patterns `(?(N)yes|no)` and `(?(<NAME>)yes|no)`. New
  opcode `PCRE_OP_COND` (= 22) with `arg1` = group index, `arg2` =
  no-pc. Matcher consults the saves array at runtime to decide
  branches. New error code `PCRE_E_BAD_CONDITION = 8` for
  unrecognized condition forms or unknown named references; the
  M3-defined `PCRE_E_CONDITIONAL_UNSUPPORTED = 5` slot is **frozen
  but no longer emitted** (kept reserved per ABI freeze rules).
- Callouts `(?C)` and `(?C<num>)` — observability only. New opcode
  `PCRE_OP_CALLOUT` (= 23) is zero-width and records the callout
  number; consumers query via `niyama_pcre_last_callout()`. **No
  callback API** — that would need function pointers and a defined
  ABI for callout handlers, neither of which fits the v0.7.0
  carve.
- Lookbehind, recursion, and Unicode property classes **stay
  deferred** (lookbehind + recursion → v0.8.0; Unicode → v0.9.0).

### Behavior change: niyama_re2 and niyama_pcre default anchors / dot

`^`, `$`, and `.` are now **strict-by-default** in re2 and pcre,
matching the RE2 / PCRE specs:

- `^` matches **only at pos 0** (string start). `(?m)` opts into
  loose multiline (also after `\n`).
- `$` matches **only at pos len** (string end). `(?m)` opts into
  loose multiline (also before `\n`).
- `.` **excludes `\n`**. `(?s)` opts into dot-matches-newline.

Pre-v0.7.0 implementations were silently loose on all three.
Existing tests didn't exercise the distinguishing cases
(no `\n` in inputs), so no test broke; a downstream consumer that
relied on loose semantics WILL see a behavior change. Documented
in CHANGELOG. Pre-surface-freeze (M5) is the right time to make
this change — once frozen at v1.0, semantics can't move.

niyama_bre and niyama_vim **keep their pre-v0.7.0 loose defaults**
— vim is loose by spec (line anchors), and POSIX BRE multi-line
`^`/`$` matches `grep`/`sed` traditional behavior even though it's
not strictly POSIX-conformant. Changing those would be more risk
than benefit for v0.7.0.

### Deferrals confirmed for v0.8.0 / v0.9.0

- v0.8.0 candidates: lookbehind `(?<=)` `(?<!)` (fixed-width
  compile-time analysis), pcre recursion `(?R)` `(?P>name)`, fuzzy
  exact-start recovery in `_search`, and the bre/vim backref
  re-decision.
- v0.9.0 candidates: `src/unicode.cyr` shared module, Unicode
  property classes `\p{L}` (re2 + pcre + vim), Unicode NFD
  (`FUZZY_FLAG_UNICODE_NFD`), and the vim → posix_classes refactor.

## Consequences

- **Positive** — Eight features land cleanly under one shared
  module. The cross-engine surface looks more like itself
  (`group_by_name`, inline-flag spelling) which makes downstream
  consumer code uniform. The strict-default fix lands while we can
  still move semantics; M5 freeze is the last chance.
- **Positive** — Each feature has working tests and fuzz coverage
  added in the same release. Aggregate test count: 362 → 483
  (+121 assertions across bre/re2/pcre); fuzz count: 1636 → 1658
  (+22 assertions including new feature smokes and rejection
  invariants).
- **Negative** — pcre opcode count grew from 18 → 24, expanding
  the matcher dispatch and the branch-target shifter. We pay
  another ~80 lines of matcher code; not enough to consider
  refactoring opcodes into a table.
- **Negative** — vim's POSIX class implementation is now
  duplicated with the shared module. The duplication is harmless
  at runtime (different prefixes, no symbol collision) but it's
  technical debt that v0.9.0 needs to clean up.
- **Neutral** — `(?m)` and `(?s)` have a real semantic effect on
  re2/pcre now (where pre-v0.7.0 they would have been no-ops since
  the matcher was already loose). Documented in the ADR + CHANGELOG.
- **Neutral** — `PCRE_E_CONDITIONAL_UNSUPPORTED = 5` slot is
  reserved-but-unused. Future engines adding new error codes start
  at 9; the slot stays for ABI stability.

## Alternatives considered

- **Wider v0.7.0 — drag all of M4.5 in.** Rejected: 11 features +
  Unicode infrastructure in one release is too much for one
  reviewable ADR + CHANGELOG entry; the user explicitly asked to
  carve.
- **Narrower v0.7.0 — bre-only catch-up.** Rejected: the user
  picked the wide carve. The pcre/re2 features are small and
  share parser/matcher infrastructure with each other (CHAR_CI,
  BOS/EOS/ANY_NONL appear in both engines), so splitting them
  would have meant designing the same opcodes twice.
- **Fold vim onto posix_classes now.** Rejected: vim's POSIX
  class parsing is intermingled with mode-aware bracket dispatch.
  Refactoring it carries regression risk for zero behavior change.
  v0.9.0 is the right time, alongside the Unicode work that vim
  also picks up.
- **Keep loose default for `^`/`$`/`.` in re2/pcre, make `(?m)`
  and `(?s)` no-ops.** Rejected: this ships the names of the
  inline flags but none of their semantics. PCRE-spec-strict is
  what consumers writing portable patterns expect; pre-v1.0 is the
  last window to make this change.
- **Branch-reset via duplicating compile passes per branch.**
  Rejected: the `_pcre_branch_reset_base` global + parse_alt-level
  reset is simpler than re-running the compiler with a fresh
  capture counter per branch. The global save/restore in
  `_pcre_parse_primary` keeps nested non-branch-reset groups
  uncontaminated.
- **Real callout callbacks via function pointers.** Rejected:
  needs a defined ABI for callout handlers (signature, calling
  convention, error returns) that would itself want an ADR. The
  observability-only stub (last-callout-number) covers the
  "did the matcher reach this point?" use case without that
  surface commitment.
