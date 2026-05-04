# 0002 — niyama_bre engine ABI shape and scope

**Status**: Accepted
**Date**: 2026-05-03

## Context

M1 ships niyama's first engine: POSIX BRE (Basic Regular Expressions —
the flavor `grep -G` / `sed` ship). Two open questions had to land
before code:

1. **ABI shape.** Cyrius stdlib `lib/regex.cyr` exposes
   `regex_compile`, `regex_match`, `regex_search`, `regex_search_at`,
   `regex_group_start`, `regex_group_end` (one foundational ERE-ish
   engine, Pike NFA). niyama could mirror that surface per engine
   (`niyama_bre_compile`, `niyama_bre_match`, …) or deviate. cyim
   ADR 0002 already commits to a uniform per-flavor dispatch arm
   that picks an engine by name and calls compile/match/search —
   the more niyama mirrors stdlib's shape, the smaller cyim's
   integration cost.

2. **Backref scope.** POSIX BRE's grammar includes `\1`-`\9`
   backreferences. Backrefs are non-regular — Pike NFA / Thompson
   construction can't handle them in linear time, only a
   backtracking matcher can. A backtracking matcher carries a
   real CVE surface (catastrophic backtracking) and adds non-trivial
   compiler/runtime complexity. Three options were on the table:

   - Skip silently (treat `\1` as a literal `1` after `\` escape).
   - Reject explicitly at compile time, document the omission.
   - Implement via a separate backtracking codepath now.

   Cyrius stdlib's Pike NFA already drops backrefs entirely (no
   error, no support). Repeating that posture in niyama would
   produce silent semantic divergence from real `grep -G`: a user's
   pattern would compile and "match" or "no-match" with no signal
   that the engine ignored a backref.

## Decision

**niyama_bre ships with the stdlib-mirroring ABI and rejects
backrefs explicitly at compile time.** Concretely:

### ABI surface (mirrors stdlib `regex_*`)

- `niyama_bre_compile(pat)` → opaque NFA pointer (heap-allocated),
  or `0` on syntax / unsupported-feature error. Last error code
  retrievable via `niyama_bre_last_error()`.
- `niyama_bre_match(nfa, s)` → `1` if `s` matches the pattern at
  position 0 (anchored to start), `0` otherwise.
- `niyama_bre_search(nfa, s)` → start offset of first match anywhere
  in `s`, or `-1`.
- `niyama_bre_search_at(nfa, s, len, from)` → start offset of first
  match in `s[from..len)`, or `-1`.
- `niyama_bre_group_start(nfa, n)` / `niyama_bre_group_end(nfa, n)`
  → capture-group `n` bounds (group 0 = whole match; user groups
  `1..9`). Returns `-1` if group didn't participate.
- `niyama_bre_last_error()` → distinct error code from the most
  recent failed compile (see error table below). `0` after a
  successful compile.

### POSIX BRE feature set shipped in M1

| Feature | Shipped | Notes |
|---|---|---|
| Literal-by-default `*` `(` `)` `{` `}` `+` `?` | yes | `*` is special only after a literal/atom, literal at start of pattern or group |
| `\(...\)` capturing groups (1..9) | yes | Saves indices `1*2` / `1*2+1` etc. |
| `\{n,m\}` / `\{n,\}` / `\{n\}` quantifiers | yes | `n_max <= 1000` per stdlib precedent |
| `^` anchor at start, `$` anchor at end | yes | `^` literal except at pattern/group start; `$` literal except at pattern/group end (per POSIX) |
| `.` any byte | yes | |
| `[...]` / `[^...]` char classes, `[a-z]` ranges | yes | POSIX bracket-expression body; bracket classes `[:alpha:]` etc. deferred to M4 (vim) reuse |
| Escapes `\.` `\*` `\\` `\n` `\t` etc. | yes | |
| `\1`-`\9` backreferences | **rejected at compile** | Returns `0` from compile; `niyama_bre_last_error()` → `BRE_E_BACKREF_UNSUPPORTED` (= 2). See § Backref rejection. |
| GNU `\<` / `\>` word boundaries | deferred to M4 (vim) | Not strict POSIX; vim flavor inherits the same semantics |
| `\d` / `\w` / `\s` / `\b` (ERE-style) | not shipped | Not POSIX BRE; stdlib `regex_*` covers ERE-ish callers |
| Bracket POSIX classes `[:alpha:]` etc. | deferred to M4 | Vim flavor needs them too — implement once, reuse |

### Backref rejection

`niyama_bre_compile` walks the pattern and returns `0` if any
`\<digit>` (where digit is `1`-`9`) is encountered outside a
character class. `niyama_bre_last_error()` returns
`BRE_E_BACKREF_UNSUPPORTED`. Backref support is **not promised for
v1.0** — see § Future scope below.

### Error codes

| Code | Constant | Meaning |
|---|---|---|
| 0 | `BRE_E_OK` | last compile succeeded (or never called) |
| 1 | `BRE_E_SYNTAX` | generic syntax error (unbalanced `\(`, malformed `\{`, trailing `\`, etc.) |
| 2 | `BRE_E_BACKREF_UNSUPPORTED` | pattern contains `\1`-`\9`; use niyama_pcre at M3 |
| 3 | `BRE_E_TOO_LARGE` | pattern compiled to > REGEX_MAX_INSTRS instructions, or > REGEX_MAX_CLASSES classes |
| 4 | `BRE_E_BAD_ANCHOR` | reserved — currently unused; `^`/`$` mid-pattern are treated as literals (POSIX) |

The numeric values are part of the ABI from M1 onward — additions
get new codes; existing codes do not change meaning.

## Consequences

### Positive

- **cyim integration cost stays at one elif arm + one dispatch
  arm.** Per cyim ADR 0002, the parser-side already exists; mirroring
  stdlib's `*_compile` / `*_match` / `*_search` / `*_search_at`
  shape means cyim's `_matcher_regex` dispatch is a 1:1 swap of the
  function names.
- **Backref omissions become discoverable.** A consumer who writes
  `\(foo\)\1` in BRE gets a compile error pointing at PCRE (M3),
  not a silent miscompile that disagrees with `grep -G`.
- **Same ABI shape repeats for re2 / pcre / fuzzy / vim.** M2+
  ADRs only have to cover flavor-specific deviations
  (`niyama_pcre_compile_with_flags` etc.), not redesign the
  surface.

### Negative

- **niyama_bre is not a complete POSIX BRE matcher.** Users who
  need backrefs in BRE patterns (a real if narrow case — `grep -G`
  scripts in the wild) have no in-niyama answer until either
  niyama_pcre lands at M3 or backrefs ship in BRE post-v1.0. This
  is the cost we accept to keep M1 in linear time.
- **Error-code surface is a permanent ABI commitment.** Adding new
  failure modes is fine; renumbering existing ones is not. Recorded
  here so the M5 freeze ADR has a concrete frozen list to enumerate.

### Neutral

- **`niyama_bre_last_error()` is a thread-unsafe global** (mirrors
  stdlib's `_re_err` model). Acceptable for M1 — niyama is
  single-threaded use today; threadsafe error reporting becomes a
  P(-1) concern at M5 if a consumer needs it.
- **GNU `\<` / `\>` deferred to M4 (vim).** vim's word-boundary
  semantics are the same as GNU's; implementing once in vim and
  having BRE call into it (or copy the implementation) is cleaner
  than two parallel codepaths. M5 cross-engine refactor will
  consolidate.

## Future scope (post-v1.0)

Backref support in niyama_bre is **possible** but not committed. If
a consumer materializes who needs POSIX-strict BRE *with* backrefs
(distinct from "wants a Perl-compat regex" — that's PCRE's
territory), the path is:

1. New ADR (`0007-bre-backref-support.md` or similar) recording
   the consumer + the design (separate backtracking matcher
   triggered by patterns that contain backrefs, with bounded-budget
   catastrophic-backtracking guard).
2. Bumped major version (or post-fold patch via cyrius release
   cycle).
3. New error code semantics: `BRE_E_BACKREF_UNSUPPORTED` would
   become legacy / never returned, but the numeric code stays
   reserved.

Not v1.0 work. The default posture is: **PCRE engine (M3) is the
home for backref-using patterns.**

## Alternatives considered

- **Silent skip / treat-as-literal.** Rejected — produces
  silent semantic divergence from `grep -G` and from any other
  POSIX BRE reference. Per niyama's "explicit reject + document"
  rule, omissions must be discoverable at compile time.
- **Implement backtracking backref matcher in M1.** Rejected —
  doubles the engine surface, doubles the fuzz target, and adds
  catastrophic-backtracking CVE risk to the smallest engine
  precisely when its purpose is to shake out the niyama dispatch
  surface. Too much for M1.
- **Translate BRE → ERE pattern string and dispatch to stdlib
  `regex_compile`.** Rejected — couples niyama to stdlib
  internals, can't add features stdlib doesn't have (e.g. future
  backref support, GNU word-boundary), and doesn't shake out the
  per-engine module organization that M2+ engines need.
- **Different ABI shape (e.g. options-bag struct from day one).**
  Rejected for M1 — premature. cyim's existing `_matcher_regex`
  expects the stdlib-style positional API; an options-bag fits
  better at M4 (vim) when magic / nomagic mode flags arrive.
  M4's ADR will revisit.

## References

- [niyama ADR 0001](0001-additional-engines-repo-sandhi-pattern.md) — positioning, fold lifecycle.
- [cyrius stdlib `lib/regex.cyr`](https://github.com/MacCracken/cyrius/blob/main/lib/regex.cyr) — Pike NFA template; niyama_bre instruction model + matcher are forked from this engine.
- [cyim ADR 0002 — `--regex=<flavor>` extensibility shape](https://github.com/MacCracken/cyim/blob/main/docs/adr/0002-regex-extensibility-shape.md) — first consumer's surface.
- [POSIX.1-2017 § 9.3 Basic Regular Expressions](https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/V1_chap09.html) — the spec niyama_bre conforms to (minus backrefs).
- [`docs/development/roadmap.md`](../development/roadmap.md) — M1 acceptance criteria.
