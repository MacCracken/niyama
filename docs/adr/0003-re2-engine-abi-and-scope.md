# 0003 — niyama_re2 engine ABI shape and scope

**Status**: Accepted
**Date**: 2026-05-03

## Context

M2 ships niyama's second engine: re2 — a Thompson-NFA / Pike-matcher
flavor whose distinguishing promise is **linear-time matching, no
exceptions**. Concrete consumer cases:

- daimon-orchestrated agents pattern-gating untrusted input. A
  catastrophic-backtracking pattern submitted to a backtracking
  engine is a denial-of-service vector. A Pike NFA isn't.
- agnoshi log analysis on attacker-controlled lines.
- Any consumer who wants the existing niyama_bre matcher's
  DoS-resistance with a richer flavor (alternation, `\d`/`\w`/`\s`,
  `+`, `?`, lazy quantifiers, `{n,m}`).

The engine kernel itself — Pike NFA, instruction model, class
bitmaps, capture-saves — is the **same** machinery niyama_bre and
cyrius stdlib `lib/regex.cyr` already use. The Pike NFA is
linear-time **by construction** (per-PC-per-step dedup means the
thread-set size is bounded by the instruction count, not pattern
complexity). What re2 adds over bre / over the implicit posture in
stdlib is **explicit, advertised, error-coded rejection at compile
time** of features that, if implemented, would break that
linear-time guarantee.

Three open questions had to land before code:

1. **What features get explicit rejection?** Real Google RE2 rejects
   `\1`-style backreferences, lookaround (`(?=...)` `(?!...)`
   `(?<=...)` `(?<!...)`), atomic groups (`(?>...)`), and
   recursion / conditional patterns. niyama_re2 mirrors that — each
   gets a distinct error code so consumers know exactly *why* a
   pattern was rejected.

2. **What's in scope for the ERE feature set?** Stdlib's `lib/regex.cyr`
   is already a Pike NFA Thompson-NFA matcher with ERE-ish syntax
   (`\d` `\w` `\s` `\b`, `[abc]`, quantifiers including lazy, `|`,
   `(...)` capturing, `(?:...)` non-capturing). niyama_re2 ships
   that same feature set — what it *adds* over stdlib is the safety
   guarantee at the API layer + the cross-engine-uniform
   `niyama_<flavor>_*` ABI shape from ADR 0002.

3. **Named captures `(?P<name>...)`?** Real RE2 supports them.
   niyama_bre at M1 only has positional captures (1..9) per ADR
   0002's stdlib-mirroring shape. Adding named-capture support
   would be a cross-engine surface change — the
   `niyama_<flavor>_group_*` ABI would need a name-resolution arm.
   Initial decision: defer named captures to consolidate the
   named-capture API surface across re2 + pcre. The pcre
   implementation landed at M3 (per ADR 0004); the re2 catch-up
   reusing that mechanism is part of v0.9.0 (M4.5 catch-up).

## Decision

**niyama_re2 ships an ERE-flavor Pike NFA engine with explicit
compile-time rejection of non-regular features, mirroring the
niyama_bre ABI shape from ADR 0002.** Concretely:

### ABI surface (mirrors niyama_bre_* per ADR 0002)

- `niyama_re2_compile(pat)` → opaque NFA pointer (heap-allocated),
  or `0` on syntax / unsupported-feature error. Last error code
  retrievable via `niyama_re2_last_error()`.
- `niyama_re2_match(nfa, s)` → `1` if `s` matches at position 0
  (anchored), `0` otherwise.
- `niyama_re2_search(nfa, s)` → start offset of first match, or `-1`.
- `niyama_re2_search_at(nfa, s, len, from)` → first match in
  `s[from..len)`, or `-1`.
- `niyama_re2_group_start(nfa, n)` / `niyama_re2_group_end(nfa, n)`
  → capture-group `n` bounds (group 0 = whole match; user groups
  1..9). `-1` if not participating.
- `niyama_re2_last_error()` → distinct error code from the most
  recent failed compile.

### ERE feature set shipped in M2

| Feature | Shipped | Notes |
|---|---|---|
| Literals + escapes (`\.` `\*` `\\` `\n` `\t` `\r` etc.) | yes | |
| `.` any byte | yes | |
| `^` start anchor, `$` end anchor | yes | At pattern boundaries; in middle they're treated per stdlib (`^` and `$` anywhere with multiline-off semantics — anchored to start/end of input). |
| `[...]` / `[^...]` bracket expressions, ranges | yes | |
| `\d` `\D` `\w` `\W` `\s` `\S` | yes | ASCII semantics (digit = `0`-`9`, word = `[A-Za-z0-9_]`, space = `[ \t\n\v\f\r]`). |
| `\b` `\B` word boundaries | yes | ASCII word definition. |
| `*` `+` `?` greedy quantifiers | yes | |
| `*?` `+?` `??` lazy quantifiers | yes | |
| `{n}` `{n,}` `{n,m}` brace quantifiers (greedy + `{n,m}?` lazy) | yes | `n_max ≤ 1000` per stdlib precedent. |
| Alternation `\|` | yes | |
| `(...)` capturing groups (1..9) | yes | |
| `(?:...)` non-capturing groups | yes | |
| **`\1`-`\9` backreferences** | **rejected at compile** | `RE2_E_BACKREF_UNSUPPORTED` (= 2). Use niyama_pcre at M3. |
| **`(?=...)` `(?!...)` lookahead** | **rejected at compile** | `RE2_E_LOOKAROUND_UNSUPPORTED` (= 3). Use niyama_pcre at M3. |
| **`(?<=...)` `(?<!...)` lookbehind** | **rejected at compile** | `RE2_E_LOOKAROUND_UNSUPPORTED` (= 3). |
| **`(?>...)` atomic groups** | **rejected at compile** | `RE2_E_ATOMIC_UNSUPPORTED` (= 4). |
| **`(?R)` `(?P>name)` recursion / subroutine calls** | **rejected at compile** | `RE2_E_RECURSION_UNSUPPORTED` (= 5). |
| `(?P<name>...)` named captures | deferred to v0.9.0 (M4.5 catch-up) | Consolidates with niyama_pcre's named-capture mechanism (ADR 0004). |
| Unicode property classes `\p{L}` | deferred to v0.9.0 (M4.5 catch-up) | Shared Unicode decomposition + property table across re2/pcre/fuzzy/vim. |
| Inline flags `(?i)` `(?m)` `(?s)` | deferred to v0.9.0 (M4.5 catch-up) | Case-fold + multiline + single-line matcher flags. |

### Linear-time guarantee — what it actually means

The Pike NFA matcher (forked from cyrius stdlib `lib/regex.cyr`) runs
a thread-set forward through the input. Per-step dedup via
`_re2_m_lastgen[pc] == gen` ensures each `(pc, position)` pair is
visited at most once. Match time is therefore O(N × M) where N is
input length and M is compiled-instruction count — **regardless of
pattern shape**. Catastrophic-backtracking inputs (`(a|a|a)*`-class
adversaries) that DoS PCRE-style backtracking matchers terminate in
linear time on Pike NFA.

The compile-time rejection of backref / lookaround / atomic /
recursion is what *protects* this guarantee. Real RE2 makes the same
choice for the same reason. niyama_re2 inherits both halves.

### Error codes (frozen ABI from M2 onward)

| Code | Constant | Meaning |
|---|---|---|
| 0 | `RE2_E_OK` | last compile succeeded (or never called) |
| 1 | `RE2_E_SYNTAX` | generic syntax error |
| 2 | `RE2_E_BACKREF_UNSUPPORTED` | pattern contains `\1`-`\9`; use niyama_pcre at M3 |
| 3 | `RE2_E_LOOKAROUND_UNSUPPORTED` | `(?=` `(?!` `(?<=` `(?<!`; use niyama_pcre at M3 |
| 4 | `RE2_E_ATOMIC_UNSUPPORTED` | `(?>...)` atomic groups; use niyama_pcre at M3 |
| 5 | `RE2_E_RECURSION_UNSUPPORTED` | `(?R)` etc.; use niyama_pcre at M3 |
| 6 | `RE2_E_TOO_LARGE` | pattern compiled too large (instr or class limit) |

Numeric values are part of the M2-frozen ABI. `RE2_E_BACKREF_UNSUPPORTED` is `2`
because that's the same numeric meaning as `BRE_E_BACKREF_UNSUPPORTED` —
deliberate cross-engine consistency for the one error code that
crosses engine boundaries.

## Consequences

### Positive

- **Pike NFA's DoS-resistance becomes a load-bearing API guarantee.**
  Consumers who specifically need DoS-safety (daimon agent gates,
  agnoshi log scanners) get a contract: any pattern that compiles
  is linear-time matchable.
- **Discoverability of the rejection.** A consumer who pastes a
  PCRE-style pattern with `(?<=foo)bar` lookbehind gets
  `RE2_E_LOOKAROUND_UNSUPPORTED`, not "syntax error" — they know
  exactly which engine to switch to (niyama_pcre at M3).
- **Cross-engine ABI consistency.** Same shape as niyama_bre
  (M1) — cyim's `_matcher_regex` dispatch arm is one extra
  function-name swap.

### Negative

- **More feature-set surface than niyama_bre.** ~700 lines of parser
  vs. ~600 for BRE. Larger fuzz target, larger CVE surface to
  audit at M5. Acceptable: ERE is what most niyama consumers
  default to in practice; spending the parsing complexity here
  pays back.
- **Named captures deferred.** Real RE2 ships them. niyama_re2 M2
  does not. Consumers who need named captures wait for the
  post-M3 cross-engine API consolidation. Mitigated by 1..9
  positional captures already covering the common case.

### Neutral

- **Per-engine fuzz harness is stricter than M1's.** The harness
  (fuzz/re2.fcyr) runs an adversarial pattern generator —
  catastrophic-backtracking-class inputs included. Each must
  compile + match in linear time. This is more stringent than
  fuzz/bre.fcyr (which mostly checked compile-doesn't-crash).
- **Unicode + inline flags deferred to v0.9.0 (M4.5 catch-up).**
  Documented above. Cross-engine Unicode table shared with
  pcre/fuzzy/vim; inline flags share case-fold mechanism with
  pcre.

## Alternatives considered

- **Skip the explicit rejection — let the parser fail generically.**
  Rejected. The point of niyama_re2 over plain stdlib `regex_*` is
  the *advertised* safety guarantee. A consumer reading
  `RE2_E_LOOKAROUND_UNSUPPORTED` knows the engine made a deliberate
  choice; a generic syntax error is indistinguishable from a typo.
- **Implement lookaround / atomic groups in re2.** Rejected —
  defeats the purpose. Lookaround forces re-matching from arbitrary
  positions, breaking single-pass linear-time. Atomic groups
  require backtracking semantics. Both belong in niyama_pcre (M3).
- **Add named captures `(?P<name>...)` in M2.** Rejected for
  scope-creep — named-capture API is a cross-engine concern (re2 +
  pcre at minimum want it). Better to design once at M3 with both
  engines in hand than ship one shape now and rev it later.
- **Skip M2 entirely; tell consumers to use niyama_bre with a
  Pike NFA disclaimer.** Rejected — consumer demand is for ERE
  flavor (alternation, `\d`, `+`, `?`, `{n,m}`), not BRE. niyama_bre
  is for `grep -G` / `sed` compat, not the general DoS-safe case.

## References

- [niyama ADR 0001](0001-additional-engines-repo-sandhi-pattern.md) — positioning, fold lifecycle.
- [niyama ADR 0002](0002-bre-engine-abi-and-scope.md) — niyama_bre ABI; niyama_re2 mirrors the shape.
- [cyrius stdlib `lib/regex.cyr`](https://github.com/MacCracken/cyrius/blob/main/lib/regex.cyr) — Pike NFA template; niyama_re2 forks the matcher kernel.
- [Google RE2 syntax](https://github.com/google/re2/wiki/Syntax) — flavor reference.
- [Russ Cox — Regular Expression Matching Can Be Simple And Fast](https://swtch.com/~rsc/regexp/regexp1.html) — the Pike NFA technique paper that informs both stdlib and niyama_re2.
- [`docs/development/roadmap.md`](../development/roadmap.md) — M2 acceptance criteria.
