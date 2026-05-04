# 0004 — niyama_pcre engine ABI shape, matcher architecture, and scope

**Status**: Accepted
**Date**: 2026-05-03

## Context

M3 ships niyama's third engine: pcre — Perl-compatible regex. This
is the engine that brings the features niyama_re2 deliberately
rejects (backreferences, lookaround, atomic groups). The roadmap
calls it "the largest fuzz target" and that's accurate: PCRE has
the richest CVE history of any regex engine in production. niyama's
take has to be conservative about scope and aggressive about
hardening.

Three load-bearing decisions had to land before code:

### 1. Matcher architecture — backtracking, not Pike NFA

The engine kernel that bre and re2 share — Pike NFA / Thompson
construction with per-PC-per-step dedup — **cannot** support PCRE
features. Concretely:

- **Backreferences** break the PC-dedup invariant. Two threads at
  the same `pc` with different captured groups are NOT equivalent
  if a future `\1` will compare them. Pike NFA's whole performance
  story (linear time) depends on collapsing equivalent threads;
  backref kills that.
- **Lookaround** requires running a sub-match from arbitrary
  positions. Pike NFA's single forward sweep can't accommodate it.
- **Atomic groups** need commit semantics — once the group succeeds,
  no backtracking past it. Pike NFA doesn't backtrack at all, so
  there's nothing to commit against.

Real PCRE2, Onigmo, Boost.Regex, java.util.regex — all backtracking.
niyama_pcre joins them. This means catastrophic-backtracking
becomes an in-engine concern (unlike bre/re2 where it's structurally
impossible), and the **step-limit guard is load-bearing**, not a
defense in depth.

### 2. Step-limit guard against catastrophic backtracking

Every backtracking matcher needs one. PCRE2 calls it `match_limit`;
RE2's docs explain why pure Pike NFA doesn't need one. niyama_pcre
ships:

- A per-`niyama_pcre_match` / `_search` step counter. Each
  instruction execution increments it. When the counter exceeds
  `_pcre_step_limit`, the match aborts and returns no-match (NOT
  an error — same shape as a normal mismatch from the consumer's
  point of view; the failure mode is "input is adversarial / pattern
  is pathological").
- `niyama_pcre_set_step_limit(n)` API to configure it. Default
  `1_000_000` steps, picked to be high enough that legitimate matches
  on 10KB-class inputs complete and low enough that 200-byte
  catastrophic patterns hit the limit in milliseconds rather than
  hours.
- `niyama_pcre_last_step_count()` — observability hook. Returns the
  step count of the most recent match call. Lets consumers tune
  their limit and detect close-to-limit matches.

This is the niyama_pcre answer to "but isn't backtracking
DoS-vulnerable?" — yes, structurally. The mitigation is the step
limit. Consumers who absolutely cannot tolerate backtracking
should use niyama_re2.

### 3. Initial feature scope

PCRE2's full surface is ~50 features. M3 picks a subset based on
real consumer demand vs. implementation complexity vs. CVE history.

**In scope for M3 (v0.4.0):**

| Feature | Notes |
|---|---|
| All ERE features (literals, `.`, anchors, char classes, `\d`/`\w`/`\s`/`\b`, `*` `+` `?` `{n,m}`, lazy, alternation, capturing + non-capturing groups) | Inherited posture from re2; rewrites as backtracking ops. |
| Backreferences `\1`-`\9` | The headline PCRE feature. |
| Named captures `(?<name>...)` and `(?P<name>...)` | Both syntaxes accepted. Lookup via `niyama_pcre_group_by_name(nfa, name)`. |
| Lookahead `(?=...)` and negative `(?!...)` | Variable-width OK (forward-only sub-match). |
| Atomic groups `(?>...)` | Once-matched, no backtracking inside. |
| Possessive quantifiers `*+`, `++`, `?+`, `{n,m}+` | Equivalent to atomic-wrapping. |
| Step-limit guard (configurable, default 1M) | Catastrophic-backtracking mitigation. |

**Deferred from M3 — post-v1.0 unless a consumer asks:**

| Feature | Why deferred |
|---|---|
| Lookbehind `(?<=...)` `(?<!...)` | Needs compile-time width analysis (PCRE2 requires fixed-width; only relaxed in 10.43); adds parser complexity disproportionate to demand. M3.5 candidate if a consumer asks. |
| Unicode property classes `\p{L}` etc. | Requires a Unicode database (~50KB+ generated table). Big complexity for a feature with narrow demand in AGNOS-lineage consumers (we're ASCII-heavy). |
| POSIX bracket classes `[:alpha:]` etc. | Deferred to M4 (vim flavor inherits the same semantics — implement once). |
| Conditional patterns `(?(cond)yes|no)` | Uncommon; adds matcher complexity. |
| Recursion / subroutine calls `(?R)`, `(?P>name)`, `(?N)` | Uncommon; non-regular by design. |
| Inline flags `(?i)` `(?m)` `(?s)` | Post-v1.0; needs case-folding tables for `(?i)` to do meaningful work. |
| Branch-reset groups `(?\|...)` | Uncommon. |
| Callouts `(?C)` | Uncommon; runtime-callback feature. |
| `\K` reset-match-start | Uncommon. |

The deferral list is long. That's the point — M3 is "PCRE-light"
covering the consumer-asked features, not "PCRE-complete". Future
ADRs (0008+) extend the surface as demand surfaces.

## Decision

**niyama_pcre ships a backtracking matcher with the PCRE-light
feature set above, mirroring the niyama_bre / niyama_re2 ABI shape
plus PCRE-specific extensions for named captures and the step-limit
guard.**

### ABI surface

Mirrors niyama_bre / niyama_re2:

- `niyama_pcre_compile(pat)` → opaque NFA pointer or `0` on error.
- `niyama_pcre_match(nfa, s)` → `1` if anchored match, else `0`.
- `niyama_pcre_search(nfa, s)` → first match offset, or `-1`.
- `niyama_pcre_search_at(nfa, s, len, from)` → as above, from offset.
- `niyama_pcre_group_start(nfa, n)` / `niyama_pcre_group_end(nfa, n)`
  → group bounds (group 0 = whole match; user 1..9).
- `niyama_pcre_last_error()` → distinct error code from most recent
  failed compile.

PCRE-specific additions:

- `niyama_pcre_group_by_name(nfa, name)` → group index for a named
  capture, or `-1` if no group has that name. Once you have the
  index, `_group_start` / `_group_end` work as for positional
  groups. Up to 9 named captures total (matches the positional
  limit; named captures share the same `1..9` slot).
- `niyama_pcre_set_step_limit(n)` → configure the per-call step
  budget. Process-global, not per-NFA. Default `1_000_000`.
- `niyama_pcre_last_step_count()` → step count from most recent
  match call. Observability only — `0` is a fresh-init value.

### Error codes (frozen ABI from M3 onward)

| Code | Constant | Meaning |
|---|---|---|
| 0 | `PCRE_E_OK` | last compile succeeded |
| 1 | `PCRE_E_SYNTAX` | generic syntax error |
| 2 | `PCRE_E_LOOKBEHIND_UNSUPPORTED` | `(?<=` `(?<!` — deferred |
| 3 | `PCRE_E_UNICODE_PROP_UNSUPPORTED` | `\p{L}` — deferred |
| 4 | `PCRE_E_RECURSION_UNSUPPORTED` | `(?R)` `(?P>name)` — deferred |
| 5 | `PCRE_E_CONDITIONAL_UNSUPPORTED` | `(?(cond)...)` — deferred |
| 6 | `PCRE_E_TOO_LARGE` | pattern compile exceeds limits |
| 7 | `PCRE_E_DUPLICATE_NAME` | two named captures with the same name |

Note: niyama_pcre **accepts** lookahead, atomic, backref, named
captures — those are not rejected. The deferral set above gets
explicit error codes so consumers know exactly what's missing.

### New opcodes (in addition to the bre/re2 set)

| Opcode | Operands | Semantics |
|---|---|---|
| `PCRE_OP_BACKREF` | group index `n` | Match against bytes captured in group `n`. Fail if group didn't participate or if input doesn't match. |
| `PCRE_OP_LOOKAHEAD_START` | end PC | Sub-match from current pos to end-PC; on success, set pc = end-PC + 1, restore captures from before. |
| `PCRE_OP_NLOOKAHEAD_START` | end PC | As above but inverted: success if sub-match fails. |
| `PCRE_OP_LOOKAHEAD_END` | — | Implicit success marker for sub-match; the matcher recognizes it. |
| `PCRE_OP_ATOMIC_START` | end PC | Run sub-pattern; on success, jump past end-PC (no backtracking through the atomic block). |
| `PCRE_OP_ATOMIC_END` | — | Marker; matcher uses it for control flow. |

### Catastrophic-backtracking guard semantics

- Every step in the backtracking matcher (each instruction
  execution) increments `_pcre_step_count`.
- If `_pcre_step_count > _pcre_step_limit`, the matcher returns
  no-match. This is **NOT an error** — `niyama_pcre_match` /
  `_search` return `0` / `-1` as for a normal miss.
- Rationale: from the consumer's point of view, "pattern X did not
  match input Y" is the right answer when Y is adversarial — same
  as if the pattern simply didn't fit. Differentiating "honestly
  didn't match" from "exceeded budget" can be done via
  `niyama_pcre_last_step_count()` checking against the limit.

## Consequences

### Positive

- **Real PCRE-feature consumers can land on niyama** without
  reaching for an external dep. Backref, named captures, lookahead,
  atomic — the load-bearing PCRE-only features — are all here.
- **Step-limit makes the DoS posture explicit.** Unlike bre/re2
  where DoS-resistance is structural, niyama_pcre's posture is
  bounded: "you get backtracking, you get the step limit, full
  stop." Consumers can pick their tradeoff — re2 if they need
  hard DoS-safety, pcre if they need the features.
- **Diff-testing across the three engines is now meaningful.**
  bre/re2/pcre overlap on most of ERE; cross-engine diff tests
  catch divergence. M5 hardening will lean on this.

### Negative

- **niyama_pcre is the largest engine in the repo.** ~1500 lines
  estimate. Largest fuzz target, largest CVE surface to audit at
  M5. Acceptable: this is what the engine is for, and the deferral
  list above keeps it from being PCRE2-sized.
- **Backtracking matcher means real recursion in cyrius.** Cyrius
  doesn't have native stack-depth limits; the matcher needs an
  explicit depth counter alongside the step counter, and OOM
  scenarios on deeply-nested patterns become a concern. M5 will
  audit.
- **Step-limit is process-global, not per-NFA.** A multi-tenant
  consumer that wants different limits for different patterns has
  to call `_set_step_limit` between calls. M5 may revisit if a
  consumer asks for per-NFA limits.

### Neutral

- **Named captures landed in pcre, not re2.** ADR 0003 deferred
  named captures from re2 to consolidate at M3. M3 is here; pcre
  has them. re2 still rejects `(?<name>...)` with `RE2_E_SYNTAX` —
  consumers who want named captures use pcre. A future ADR may
  backfill named captures into re2 if a DoS-safe-named-capture
  consumer materializes.
- **Lookbehind deferral is the most-likely-revisited deferral.**
  cyim's `:s/old/new/` could use `(?<=foo)bar` patterns. If cyim
  asks, M3.5 ADR + lookbehind ship. Until then, deferred.

## Alternatives considered

- **Try to extend Pike NFA to support PCRE features.** Rejected.
  Backref structurally breaks PC-dedup; lookaround needs sub-matching
  the Pike model can't do. The fork-and-add path doesn't end
  somewhere nice.
- **Implement full PCRE2 surface in M3.** Rejected — too large for
  one milestone. Deferral list keeps M3 to the consumer-asked
  subset; future ADRs extend.
- **Skip niyama_pcre entirely; recommend external PCRE2 binding.**
  Rejected. niyama is sovereign Cyrius (per ADR 0001's "no
  backwards-compat shims for non-AGNOS regex APIs"). An external
  PCRE2 binding undermines the whole positioning.
- **Make step-limit return a distinct error rather than no-match.**
  Considered. PCRE2 returns `PCRE2_ERROR_MATCHLIMIT` for this. The
  rejection rationale is consumer ergonomics — most consumers who
  hit the limit just want "no, this pattern doesn't apply"; the
  observability hook (`_last_step_count`) covers the rare case
  where it matters.
- **Ship lookbehind in M3.** Considered (roadmap mentions it as
  initial scope). Rejected per the deferral rationale above —
  variable-width lookbehind needs significant parser work
  (forced-fixed-width analysis), and the consumer demand isn't
  load-bearing for M3. M3.5 candidate.

## References

- [niyama ADR 0001](0001-additional-engines-repo-sandhi-pattern.md) — positioning, fold lifecycle.
- [niyama ADR 0002](0002-bre-engine-abi-and-scope.md) — niyama_bre ABI.
- [niyama ADR 0003](0003-re2-engine-abi-and-scope.md) — niyama_re2 ABI; named captures deferred for cross-engine consolidation here.
- [PCRE2 documentation](https://www.pcre.org/current/doc/html/pcre2pattern.html) — flavor reference.
- [PCRE2 `match_limit` semantics](https://www.pcre.org/current/doc/html/pcre2api.html) — the prior art for niyama_pcre's step-limit guard.
- [Russ Cox — Regular Expression Matching: the Virtual Machine Approach](https://swtch.com/~rsc/regexp/regexp2.html) — backtracking matcher design.
- [`docs/development/roadmap.md`](../development/roadmap.md) — M3 acceptance criteria.
