# niyama — Benchmark history

Per-engine benchmark numbers captured at each major release as the
regression-detection floor for the next release. P(-1) step 2 (CLAUDE.md
§ Process) sets this baseline; step 7 (post-review) re-runs against it.

Re-run any single engine with `cyrius bench tests/<engine>.bcyr`.

## Conventions

- All measurements via `cyrius bench` — wall-clock per-iteration averages
  with min/max captured. iter-count is per-bench (10k–100k typical).
- Numbers are wall-clock on a single x86_64 development machine; not
  comparison-grade across hosts. The use is *regression detection
  within niyama*, not "niyama vs PCRE2 the C library."
- A regression = a row's `avg` increasing by more than ~20% (single-host
  noise margin) without a known cause. Smaller drift is logged but not
  acted on.

## v0.8.0 baseline (P(-1) step 2 — pre-hardening)

**Date**: 2026-05-05
**Cyrius**: 5.8.65
**Host**: Linux 7.0.3-arch1-1 x86_64

### bre

| Bench | avg | min | max | iters |
|---|---|---|---|---|
| `bre_compile_literal` | 2µs | 1µs | 205µs | 10 000 |
| `bre_compile_dot_star` | 2µs | 1µs | 96µs | 10 000 |
| `bre_compile_quantifier` | 2µs | 1µs | 96µs | 10 000 |
| `bre_compile_group` | 2µs | 1µs | 56µs | 10 000 |
| `bre_search_literal_hit` | 40µs | 35µs | 131µs | 10 000 |
| `bre_search_literal_miss` | 7µs | 5µs | 87µs | 10 000 |
| `bre_search_dot_star` | 102µs | 99µs | 155µs | 1 000 |
| `bre_search_class` | 1µs | 1µs | 57µs | 1 000 |
| `bre_search_quantifier` | 2µs | 2µs | 59µs | 10 000 |
| `bre_search_anchored` | 1µs | 1µs | 72µs | 100 000 |
| `bre_search_group` | 15µs | 13µs | 72µs | 1 000 |

### re2

| Bench | avg | min | max | iters |
|---|---|---|---|---|
| `re2_compile_literal` | 3µs | 2µs | 26µs | 10 000 |
| `re2_compile_alt` | 3µs | 3µs | 54µs | 10 000 |
| `re2_compile_email` | 4µs | 3µs | 16µs | 10 000 |
| `re2_search_literal` | 41µs | 35µs | 415µs | 10 000 |
| `re2_search_alt` | 71µs | 64µs | 274µs | 10 000 |
| `re2_search_class` | 148µs | 141µs | 216µs | 1 000 |
| `re2_search_email` | 14µs | 13µs | 58µs | 10 000 |

### pcre

| Bench | avg | min | max | iters |
|---|---|---|---|---|
| `pcre_compile_literal` | 3µs | 2µs | 114µs | 10 000 |
| `pcre_compile_email` | 4µs | 3µs | 588µs | 10 000 |
| `pcre_compile_backref` | 4µs | 2µs | 160µs | 10 000 |
| `pcre_compile_lookahead` | 3µs | 2µs | 93µs | 10 000 |
| `pcre_search_literal` | 17µs | 14µs | 104µs | 10 000 |
| `pcre_search_alt` | 61µs | 56µs | 131µs | 10 000 |
| `pcre_search_email` | 8µs | 6µs | 85µs | 10 000 |

### fuzzy

| Bench | avg | min | max | iters |
|---|---|---|---|---|
| `fuzzy_compile_default` | 971ns | 811ns | 202µs | 100 000 |
| `fuzzy_compile_opts` | 959ns | 811ns | 122µs | 100 000 |
| `fuzzy_distance_short` | 806ns | 761ns | 24µs | 100 000 |
| `fuzzy_distance_long` | 2µs | 2µs | 21µs | 100 000 |
| `fuzzy_match` | 794ns | 762ns | 6µs | 100 000 |
| `fuzzy_search_short` | 2µs | 2µs | 23µs | 100 000 |
| `fuzzy_search_long_256B` | 26µs | 25µs | 58µs | 10 000 |
| `fuzzy_search_prefix` | 1µs | 1µs | 35µs | 100 000 |
| `fuzzy_medium_pattern_distance` | 14µs | 14µs | 38µs | 10 000 |

### vim

| Bench | avg | min | max | iters |
|---|---|---|---|---|
| `vim_compile_magic` | 2µs | 1µs | 31µs | 10 000 |
| `vim_compile_very_magic` | 2µs | 1µs | 82µs | 10 000 |
| `vim_compile_nomagic` | 2µs | 1µs | 100µs | 10 000 |
| `vim_compile_very_nomagic` | 2µs | 841ns | 202µs | 10 000 |
| `vim_compile_zs_ze` | 2µs | 1µs | 75µs | 10 000 |
| `vim_compile_posix` | 3µs | 2µs | 105µs | 10 000 |
| `vim_search_magic` | 6µs | 5µs | 90µs | 10 000 |
| `vim_search_very_magic` | 6µs | 5µs | 88µs | 10 000 |
| `vim_search_nomagic` | 6µs | 5µs | 64µs | 10 000 |
| `vim_search_very_nomagic` | 6µs | 5µs | 71µs | 10 000 |
| `vim_search_zs_ze` | 2µs | 1µs | 26µs | 10 000 |
| `vim_search_posix` | 2µs | 2µs | 19µs | 10 000 |
| `vim_search_word_bound` | 2µs | 1µs | 56µs | 10 000 |

## Cross-engine observations (from this baseline)

A few rows worth flagging for the step 3 internal review:

- **`re2_search_class` at 148µs** is the highest per-iteration cost in
  the entire matrix — ~37× `re2_compile_email`. The benchmark's input
  is intentionally character-class-heavy; check whether the Pike NFA's
  per-byte class-lookup loop has a hot inner that could be tightened.
- **`bre_search_dot_star` at 102µs** is the second-highest, which is
  consistent with `.*` being the worst-case Pike NFA pattern (every
  position must be tried). Less actionable but worth a glance for any
  byte-stepped redundancy.
- **`bre_search_literal_hit` at 40µs vs `bre_search_literal_miss` at
  7µs** — 5.7× gap. Hits do extra work to record match position; the
  miss path bails fast. Expected. No action.
- **`re2_search_literal` at 41µs vs `pcre_search_literal` at 17µs** —
  pcre is 2.4× faster on literal search than re2. pcre uses
  byte-stepped backtracking with a fast-bail path; re2 uses
  codepoint-stepped Pike NFA threads with full thread queueing.
  Architectural difference, expected. Documenting so it doesn't
  read as a re2 regression in step 7.
- **fuzzy compile is sub-microsecond** — fastest engine to compile, by
  10–20× over the regex engines. Expected (no opcode emission, just
  a memcpy into the handle).

These don't move v0.8.0 → v0.9.0. They're seeds for the deep review +
optimization opportunities in step 3.

## v0.9.0 post-hardening (P(-1) step 7 — diff vs. v0.8.0)

**Date**: 2026-05-05 (same-day rerun)
**Cyrius**: 5.8.65 (unchanged)
**Host**: Linux 7.0.3-arch1-1 x86_64 (unchanged)

Step 6 work landed: G2 (re2 POSIX bracket classes), C1 (boundary
tests across 4 engines), C3 (invalid-UTF-8 fuzz seeds across 3
fuzz harnesses), F1 (4 architecture notes). No matcher-loop or
opcode changes. Expected: zero regression.

### Diff summary (all rows; > 5% change is the action threshold)

All 47 measurements within **±2.5%** of the v0.8.0 baseline.
Treating as **noise** (single-host wall-clock, run-to-run
variance is in this range).

Notable rows:

- `re2_search_class`: 148µs → 147µs (−0.7%). The bench harness's
  re2 class search hits the same byte-bitmap path post-G2; POSIX
  class additions are inside the parser, not the matcher.
- `bre_search_literal_hit`: 40µs → 41µs (+2.5%). Within run noise.
- `fuzzy_compile_default`: 971ns → 984ns (+1.3%). Within run noise.

No bench moved more than 2.5% in either direction. Conclusion:
**no regressions from v0.9.0 step-6 work.**

### B4 finding (pcre in-loop saves alloc) — gated decision

The v0.9.0 review flagged `pcre.cyr` allocating
`PCRE_MAX_SAVES * 8` bytes (160 bytes) inside `_pcre_match_run` at
6 sites (SPLIT, LOOKAHEAD, NLOOKAHEAD, LOOKBEHIND, RECURSE, plus
top-level entry) as a possible perf concern for long-running
consumers. Step-7 measurement gates the refactor.

**Result**: no measurable impact at the v0.8.0 bench corpus level
(pcre rows all within noise). The benchmarks don't deeply nest
LOOKAHEAD / RECURSE patterns; allocation cost is invisible at
this workload.

**Decision**: **defer to post-fold.** Per ADR 0009 + 004
(architecture/004-no-shared-matcher-kernel.md), per-engine
optimizations are isolated and don't affect the v1.0 ABI freeze.
A future cyim-style consumer hitting deeply-nested patterns can
motivate the refactor with a real bench. Adding it speculatively
near freeze trades risk for unmeasured benefit.

The audit note (`docs/audit/2026-05-05-audit.md` § 2 Buffer safety)
documents B4 with this status.

### Going forward

v1.0 closeout will re-run benches once more as the final regression
gate. If any row moves > 5% between v0.9.0 step-7 and v1.0 closeout
without a known cause, that's a step-9 redo.
