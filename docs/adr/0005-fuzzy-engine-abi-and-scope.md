# 0005 — niyama_fuzzy engine ABI shape and scope

**Status**: Accepted
**Date**: 2026-05-03

## Context

M3.5 ships niyama's fourth engine: fuzzy — Levenshtein edit-distance
matching. It's the one engine in niyama that **isn't regex** in the
strict sense; the niyama positioning per ADR 0001 is "additional
rule-engines for the AGNOS-lineage Cyrius ecosystem", and fuzzy
matching sits in the same pattern-matching neighborhood that owns
shell completion, fuzzy file finding, typo-tolerant command lookup.

Three load-bearing decisions had to land before code:

### 1. Algorithm — DP, not bitap

Three families of approximate-string-matching algorithms were
candidates:

- **Standard DP (Levenshtein)** — O(n × m) time, O(min(n,m)) memory
  with two-row optimization. Simple, well-known, easy to verify.
- **Bitap (Wu–Manber)** — O(n × m / w) where w is word size. Faster
  for short patterns (≤64 chars on 64-bit), but adds bitset
  encoding complexity.
- **Ukkonen's algorithm** — O(n × k) where k is the threshold.
  Faster when k is small relative to n.

For fuzzy-match consumers — shell completion, command lookup, file
prefix matching — patterns are short (typically 1-32 chars) and
text is short (typically the same). The constant-factor win of
bitap doesn't pay back the implementation complexity here. Picking
**standard DP** keeps niyama_fuzzy small (~300 lines vs. ~600 for
bitap with full feature parity) and the M5 audit surface minimal.

### 2. Match modes — three of them

Real consumer needs split into three semantically-distinct match
shapes:

- **Anchored / full-string match** — `match("hello", "helo")` →
  yes, distance 1. Used for "is THIS string roughly equal to THAT
  one?" Comparison-style.
- **Substring-fuzzy search** — find a contiguous slice of text
  within edit distance of the pattern. Used for "does the text
  contain something LIKE this?" Search-style.
- **Prefix-fuzzy search** — pattern is a prefix of text with at
  most K typos. Used for shell completion: typing `cofig` should
  still find `config-set` because `cofig` is within 1 edit of the
  prefix `confi` of `config-set`.

niyama_fuzzy ships all three as distinct functions
(`niyama_fuzzy_match`, `_search`, `_search_prefix`). Trying to
overload one function with a mode flag adds API surface for no
ergonomic gain.

### 3. Unicode and case-folding

Real-world fuzzy matching often wants:

- Case-insensitive comparison: `Config` should match `config`.
- Unicode-aware normalization (NFD): `café` should match `cafe`
  with one substitution after NFD-decomposing the `é`.

For M3.5 scope:

- **Case-folding**: ASCII-only via `FUZZY_FLAG_CASE_INSENSITIVE`.
  Consumers get the case-fold without paying the full Unicode
  table cost. ≥90% of consumer use cases are ASCII (filenames,
  command names, shell history) per the AGNOS-lineage consumer
  set.
- **Unicode NFD normalization**: deferred to v0.9.0 (M4.5 catch-up).
  Adding NFD requires shipping a ~25KB Unicode decomposition
  table; v0.9.0 ships the shared table used by re2/pcre/fuzzy/vim,
  so fuzzy's NFD lands alongside.

`FUZZY_FLAG_UNICODE_NFD` ships at v0.9.0 with the shared Unicode
decomposition table. ABI stays compatible — additions only.

## Decision

**niyama_fuzzy ships a Levenshtein DP matcher with three explicit
match modes and an ASCII-only case-fold flag.** Concretely:

### ABI surface

Mirrors the niyama_bre / niyama_re2 / niyama_pcre shape per
ADR 0002, with additions for the fuzzy-specific options:

- `niyama_fuzzy_compile(pat)` — compile with default options
  (max_edits=2, no case-fold). Returns opaque handle or `0` on
  error. Errors retrievable via `niyama_fuzzy_last_error()`.
- `niyama_fuzzy_compile_opts(pat, max_edits, flags)` — full
  options form. `flags` is a bitmask of `FUZZY_FLAG_*` constants.
- `niyama_fuzzy_match(handle, s)` — anchored: `1` if Levenshtein
  distance between `pat` and `s` is ≤ max_edits, else `0`.
- `niyama_fuzzy_search(handle, s)` — substring-fuzzy: `1` if `s`
  contains a contiguous slice within edit distance of `pat`.
  Returns approximate start offset of best match; `-1` on no
  match. (For anchored start position recovery — exact start of
  best slice — see § Approximate start.)
- `niyama_fuzzy_search_prefix(handle, s)` — prefix-fuzzy: `1` if
  some prefix of `s` is within edit distance of `pat`.
- `niyama_fuzzy_distance(handle, s)` — actual Levenshtein
  distance between `pat` and `s` (full strings). `-1` if the
  distance exceeds an internal cap (currently
  `max(strlen(pat), strlen(s)) + 1`, which is unreachable; a real
  no-match signal would only apply if we add bounded-distance
  early-exit later).
- `niyama_fuzzy_last_distance()` — actual distance from the most
  recent `_match` / `_search` / `_search_prefix` / `_distance`
  call. Useful for "match found, AND it cost N typos."
- `niyama_fuzzy_last_error()` — error code from most recent
  `_compile` / `_compile_opts`.

### Flag constants

| Flag | Value | Meaning |
|---|---|---|
| `FUZZY_FLAG_CASE_INSENSITIVE` | 1 | ASCII A-Z ↔ a-z folded before comparison |

(More flags reserved for post-v1.0 — `FUZZY_FLAG_UNICODE_NFD` is the
next likely addition.)

### Error codes (frozen ABI from M3.5 onward)

| Code | Constant | Meaning |
|---|---|---|
| 0 | `FUZZY_E_OK` | last compile succeeded |
| 1 | `FUZZY_E_PATTERN_TOO_LONG` | pattern exceeds `FUZZY_MAX_PAT_LEN` (256 bytes) |
| 2 | `FUZZY_E_INVALID_THRESHOLD` | max_edits is negative |

Numeric values frozen. Future additions get new codes.

### Approximate start

The substring-fuzzy DP gives us the **end** position of the best
match for free. Recovering the exact **start** position requires
either backtracking through the DP (extra memory) or running a
second DP on reversed strings. M3.5 ships the heuristic
(`end - len(pat)` clamped); v0.9.0 (M4.5 catch-up) ships the
exact start via reverse-DP pass.

## Consequences

### Positive

- **Smallest engine in niyama** by line count (~300). Smallest
  fuzz target, smallest CVE surface. Easy to audit at M5.
- **Three immediate consumers covered**: agnoshi shell completion
  (`_search_prefix`), daimon agent fuzzy-name-match (`_match`),
  cyim's `:e <prefix>` completion (`_search_prefix`).
- **Clean handle layout** — fuzzy doesn't need the instruction /
  class-bitmap / saves machinery the regex engines need. Just
  pattern + length + options. ~300 bytes per handle.

### Negative

- **No Unicode** in M3.5. Real fuzzy matching often wants `é ≈ e`,
  Cyrillic-Latin lookalike detection, etc. Documented deferral.
  v0.9.0 (M4.5 catch-up) ships `FUZZY_FLAG_UNICODE_NFD` with the
  shared Unicode decomposition table.
- **Approximate start position** for `_search` — heuristic in
  M3.5; exact-start via reverse-DP at v0.9.0 (M4.5 catch-up).

### Neutral

- **No "compile" benefit**: unlike regex engines where compile
  pre-builds an instruction stream, fuzzy "compile" just stashes
  the pattern + options. The compile/match split is for ABI
  consistency with the other niyama engines, not for performance.
  Consumers can call match/search directly with no perceived cost.

## Alternatives considered

- **Bitap (Wu–Manber) algorithm.** Rejected — faster constant
  factor on short patterns but more code complexity. DP is
  simpler to verify; M5 audit cost is what matters here.
- **Ukkonen's algorithm.** Rejected — faster for small thresholds
  but the DP table is small enough that the algorithmic win
  doesn't pay back the implementation cost for niyama-class
  patterns (≤256 chars).
- **Single overloaded API with mode flag.** Rejected — the three
  match shapes (anchored / substring / prefix) are
  semantically-distinct enough that consumers reading
  `niyama_fuzzy_search(_, _, MODE_PREFIX)` would have to look up
  what mode does what every time. Three named functions cost two
  more entry points and zero ambiguity.
- **Ship Unicode NFD in M3.5.** Rejected for M3.5 — Unicode
  decomposition table is ~25KB, would have made M3.5 the milestone
  introducing the cross-engine Unicode dependency. Rolled into
  v0.9.0 catch-up where re2/pcre/fuzzy/vim share one table.
- **Ship exact start-position recovery in M3.5.** Rejected for M3.5
  scope; the heuristic covers ≥90% of consumer use cases. v0.9.0
  catch-up ships exact-start via reverse-DP pass.

## References

- [niyama ADR 0001](0001-additional-engines-repo-sandhi-pattern.md) — positioning, fold lifecycle.
- [niyama ADR 0002 / 0003 / 0004](0002-bre-engine-abi-and-scope.md) — prior engine ABIs that niyama_fuzzy mirrors.
- [Wagner-Fischer algorithm (Wikipedia)](https://en.wikipedia.org/wiki/Wagner%E2%80%93Fischer_algorithm) — the Levenshtein DP.
- [Ukkonen 1985 — "Algorithms for Approximate String Matching"](https://www.cs.helsinki.fi/u/ukkonen/InfCont85.PDF) — alternative algorithm considered.
- [`docs/development/roadmap.md`](../development/roadmap.md) — M3.5 acceptance criteria.
