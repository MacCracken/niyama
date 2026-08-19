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

## v1.0.7 pin bump (6.4.64 → 6.5.29 — same-day A/B)

**Date**: 2026-08-19
**Cyrius**: 6.4.64 (old pin) vs 6.5.29 (new pin)
**Host**: Linux 7.1.8-arch1-3 x86_64

### Method

Unlike the v0.8.0 / v0.9.0 rows above, this is a **same-day A/B on one
host**, not a comparison against a stored baseline. Both toolchains built
the identical working tree with the identical vendored `lib/`, so the only
variable is the compiler. This matters because the numbers here are *not*
comparable to the v0.8.0 / v0.9.0 tables: `cyrius bench` now measures and
subtracts a per-sample timer floor (~1.34µs per clock read), which the
earlier baselines did not do. Both pins report that floor, so old-vs-new
below is like-for-like even though old-vs-v0.9.0 is not.

Each pin was run **3×** over all 5 harnesses; the table compares per-row
**medians**, with each row's own run-to-run spread shown so noise is
visible rather than inferred. A single run of each had suggested 14 rows
over the ±5% action threshold — all 14 collapsed into the noise band once
repeated, which is the reason the 3× protocol is recorded here.

### Result

**No regression.** Aggregate drift across all 57 measurements:
mean **+0.04%**, median **+0.20%**.
**Zero rows** exceed ±5% by more than their own run-to-run spread. The only
row nominally over threshold is `bre_search_class` (+7.0%), whose old-side
spread is 8.3% — noise.

| Bench | 6.4.64 (median) | 6.5.29 (median) | Δ | spread old / new |
|---|---|---|---|---|
| `bre_compile_literal` | 4.803µs | 4.671µs | -2.7% | 4.0% / 2.2% |
| `bre_compile_dot_star` | 4.859µs | 4.776µs | -1.7% | 2.2% / 0.9% |
| `bre_compile_quantifier` | 4.912µs | 4.775µs | -2.8% | 3.7% / 1.2% |
| `bre_compile_group` | 4.759µs | 4.695µs | -1.3% | 1.7% / 0.6% |
| `bre_search_literal_hit` | 32.638µs | 32.729µs | +0.3% | 0.4% / 0.9% |
| `bre_search_literal_miss` | 5.802µs | 5.847µs | +0.8% | 1.5% / 0.2% |
| `bre_search_dot_star` | 109.610µs | 110.382µs | +0.7% | 7.0% / 3.6% |
| `bre_search_class` | 1.406µs | 1.505µs | +7.0% | 8.3% / 0.5% |
| `bre_search_quantifier` | 2.180µs | 2.195µs | +0.7% | 2.4% / 1.7% |
| `bre_search_anchored` | 1.255µs | 1.275µs | +1.6% | 1.5% / 1.9% |
| `bre_search_group` | 15.192µs | 15.199µs | +0.0% | 1.2% / 2.2% |
| `re2_compile_literal` | 5.367µs | 5.393µs | +0.5% | 9.5% / 3.4% |
| `re2_compile_alt` | 5.907µs | 5.902µs | -0.1% | 7.0% / 0.7% |
| `re2_compile_email` | 6.256µs | 6.319µs | +1.0% | 6.9% / 3.2% |
| `re2_search_literal` | 32.865µs | 32.577µs | -0.9% | 3.3% / 2.6% |
| `re2_search_alt` | 62.472µs | 63.345µs | +1.4% | 2.6% / 1.6% |
| `re2_search_class` | 141.947µs | 141.979µs | +0.0% | 1.8% / 1.1% |
| `re2_search_email` | 13.660µs | 13.593µs | -0.5% | 2.2% / 3.2% |
| `re2_dos_alt_explosion (200a)` | 83.986µs | 86.291µs | +2.7% | 2.6% / 2.5% |
| `re2_dos_nested_star (200a)` | 59.148µs | 60.632µs | +2.5% | 2.5% / 4.4% |
| `re2_dos_optional_chain (30a)` | 207.328µs | 205.191µs | -1.0% | 1.2% / 1.5% |
| `pcre_compile_literal` | 5.345µs | 5.360µs | +0.3% | 2.6% / 2.0% |
| `pcre_compile_email` | 6.260µs | 6.173µs | -1.4% | 3.4% / 3.4% |
| `pcre_compile_backref` | 5.702µs | 5.723µs | +0.4% | 3.3% / 4.0% |
| `pcre_compile_lookahead` | 5.397µs | 5.383µs | -0.3% | 3.3% / 0.4% |
| `pcre_search_literal` | 12.019µs | 12.022µs | +0.0% | 1.1% / 2.6% |
| `pcre_search_alt` | 44.806µs | 45.072µs | +0.6% | 2.7% / 1.4% |
| `pcre_search_email` | 5.902µs | 5.929µs | +0.5% | 0.6% / 0.5% |
| `pcre_backref` | 770ns | 776ns | +0.8% | 1.9% / 3.1% |
| `pcre_lookahead` | 133.646µs | 134.052µs | +0.3% | 1.7% / 1.5% |
| `pcre_neg_lookahead` | 849ns | 865ns | +1.9% | 1.6% / 1.2% |
| `pcre_atomic` | 1.539µs | 1.559µs | +1.3% | 3.3% / 1.7% |
| `pcre_named_captures` | 1.773µs | 1.728µs | -2.5% | 1.6% / 1.6% |
| `pcre_dos_bounded (step_limit=50k)` | 1.956ms | 1.973ms | +0.9% | 1.8% / 1.0% |
| `fuzzy_compile_default` | 534ns | 528ns | -1.1% | 1.3% / 0.6% |
| `fuzzy_compile_opts` | 512ns | 513ns | +0.2% | 2.5% / 2.1% |
| `fuzzy_distance_short` | 461ns | 455ns | -1.3% | 2.6% / 1.1% |
| `fuzzy_distance_long` | 2.420µs | 2.443µs | +1.0% | 3.1% / 0.1% |
| `fuzzy_match` | 462ns | 459ns | -0.6% | 2.8% / 0.4% |
| `fuzzy_search_short` | 2.683µs | 2.690µs | +0.3% | 4.6% / 2.2% |
| `fuzzy_search_long_256B` | 27.521µs | 27.651µs | +0.5% | 0.2% / 11.0% |
| `fuzzy_search_prefix` | 1.524µs | 1.557µs | +2.2% | 4.9% / 2.2% |
| `fuzzy_case_insensitive` | 583ns | 590ns | +1.2% | 0.5% / 1.7% |
| `fuzzy_medium_pattern_distance` | 14.999µs | 15.070µs | +0.5% | 1.1% / 1.7% |
| `vim_compile_magic` | 4.849µs | 4.883µs | +0.7% | 5.4% / 2.2% |
| `vim_compile_very_magic` | 4.827µs | 4.783µs | -0.9% | 0.0% / 5.5% |
| `vim_compile_nomagic` | 4.888µs | 4.774µs | -2.3% | 3.0% / 2.7% |
| `vim_compile_very_nomagic` | 4.918µs | 4.844µs | -1.5% | 3.4% / 1.6% |
| `vim_compile_zs_ze` | 5.042µs | 5.050µs | +0.2% | 0.4% / 1.4% |
| `vim_compile_posix` | 5.077µs | 5.074µs | -0.1% | 1.8% / 0.9% |
| `vim_search_magic` | 5.321µs | 5.352µs | +0.6% | 1.5% / 1.4% |
| `vim_search_very_magic` | 5.409µs | 5.334µs | -1.4% | 2.9% / 2.8% |
| `vim_search_nomagic` | 5.354µs | 5.270µs | -1.6% | 1.3% / 2.5% |
| `vim_search_very_nomagic` | 5.361µs | 5.320µs | -0.8% | 1.5% / 4.0% |
| `vim_search_zs_ze` | 1.731µs | 1.691µs | -2.3% | 2.7% / 1.5% |
| `vim_search_posix` | 1.981µs | 1.957µs | -1.2% | 3.6% / 2.7% |
| `vim_search_word_bound` | 1.963µs | 1.945µs | -0.9% | 2.5% / 10.1% |

### Note — binary size, not speed

The pin bump carries a large **binary-size** regression that these
timings do not capture: the DCE'd smoke binary grew 4,544 B → 401,240 B,
bisected to cyrius 6.5.15 → 6.5.16. See CHANGELOG § 1.0.7 and
`development/state.md` § Toolchain. Runtime performance is unaffected —
that is what the table above establishes.
