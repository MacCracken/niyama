# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

## [1.0.9] — 2026-09-08

P(-1) hardening pass: audit, refactor, optimization and security sweep.
**One CRITICAL and eight HIGH findings, all fixed.** Full report in
[`docs/audit/2026-09-08-audit.md`](docs/audit/2026-09-08-audit.md).

Method note: the review ran as a 6-dimension parallel pass followed by an
adversarial refutation pass (32 raw findings, 3 refuted, 29 confirmed).
Every finding below was independently reproduced with a compiled probe
against the real source before a repair was written, and each repair was
re-measured after. Two findings the reviewers confirmed were rejected on
measurement — see **Measured and rejected**.

### Security

- **CRITICAL — heap out-of-bounds write via a crafted pattern.**
  `_<engine>_emit_raw` is the only place enforcing `MAX_INSTRS`, but the
  quantifier and alternation paths grow the program with
  `_<engine>_shift_right` and then increment `_instr_n` directly. The `?`
  branch performs no emit afterwards, so nothing checked the ceiling.
  Pattern `"(?:" × 400 + "a" × 4095 + ")?" × 400` drove `_pcre_instr_n` to
  **4496** against `MAX_INSTRS = 4096` — 6 400 bytes past the instruction
  region, through the class bitmaps and name table and out of the 68 200-byte
  NFA allocation entirely. Leaked instruction words were then observed
  inside a subsequent `alloc()` result. `niyama_pcre_compile` still returned
  0 with `TOO_LARGE`, so the API looked correct while the heap was already
  corrupt — which is why 661 tests missed it. Fixed with a ceiling check at
  all **16** shift-based growth sites (bre 2, re2 4, pcre 5, vim 5).
  `instr_n` now stops at exactly 4096. Present in re2, the engine daimon
  uses as a DoS-safe gate on untrusted input.
- **HIGH — three unbounded, unreclaimable heap-growth sites.** `alloc()` is
  a bump allocator whose only reclaim (`alloc_reset()`) invalidates every
  outstanding pointer, so a library holding caller-visible handles can never
  call it; every per-call `alloc()` is a permanent leak.
  - `_<engine>_pike_run` allocated two 160-byte scratch arrays **per start
    position** of a search. Measured on re2, pattern `Z`, 20 000-byte
    subject: **6 400 320 B → 0 B** (exactly 320 B per input byte).
  - pcre's backtracker allocated a 160-byte snapshot at every executed
    SPLIT / LOOKAHEAD / NLOOKAHEAD / LOOKBEHIND / RECURSE, bounded only by
    the 1M step limit. Measured on `(a+)+b` over 24 `a`s:
    **45 715 840 B → 0 B**.
  - `_pcre_run_at` allocated the top-level saves array per start offset.
  All three hoisted to process-lifetime buffers in `_<engine>_lazy_init`;
  pcre's snapshots became a depth-indexed pool sized
  `(PCRE_MAX_DEPTH + 2) × PCRE_MAX_SAVES × 8`.
- **HIGH — the catastrophic-backtracking guard was (len+1)× weaker than
  documented.** `_pcre_run_at` reset `_pcre_step_count = 0` on entry and
  `niyama_pcre_search_at` calls it once per start offset, so a search's real
  budget was `step_limit × (len + 1)` — 4 billion steps on a 4 KB subject,
  not the documented 1 000 000. The reset moved to a new
  `_pcre_begin_match()` called once per public entry point.
  `niyama_pcre_set_step_limit()` / `_last_step_count()` unchanged.
- **LOW — `from` was never validated** on any of the four
  `niyama_<engine>_search_at` entry points; a negative offset ran the
  matcher backwards off the front of the subject buffer. (The v0.9.0 audit's
  input-validation table asserted these checks were present. They were not.)
  `len < 0` is rejected too.
- **LOW — unchecked `alloc()` results.** All 33 lazy-init allocations across
  the five engines are now checked; on failure the engine reports
  `TOO_LARGE` and returns without setting its init flag, so a later call
  retries rather than running on half-initialised state. Previously heap
  exhaustion became a write to the NULL page.

### Fixed

- **`{n,m}` was compiled by re-parsing the atom, corrupting captures.**
  `_<engine>_apply_brace_q` rewound the source cursor and re-ran
  `_parse_primary` per repetition, re-running its non-idempotent side
  effects — one capture index, one name-table slot and one class bitmap
  burned per copy — and re-entering itself, which clobbered the shared
  `_q_splits` scratch. Affected all four regex engines:

  | Pattern | Before | Now |
  |---|---|---|
  | `(?:a{1,2}){1,2}` vs `"aa"` | no match | match |
  | `(a){2}(b)` vs `"aab"` | group 2 = 2nd copy of `a` | group 2 = `b` |
  | `(a){10}` | rejected `SYNTAX` | compiles |
  | `(?<x>a){2}` | rejected `DUPLICATE_NAME` | compiles |

  The re2 case fails **open**: a nested-quantifier deny-rule in a pattern
  gate silently stops matching with `last_error() == 0`. Repetitions are now
  relocated copies of the compiled atom (`_tmpl_save` / `_tmpl_append`); no
  parser state is touched per repetition, which removes the re-entrancy and
  so fixes the `_q_splits` clobber by construction.
- **Split-list capacity contradicted the advertised limit.** All four
  engines reject `n_max > 1000`, but the list held 64 — so `{0,65}`
  through `{0,1000}` were falsely `TOO_LARGE`. Capacity is now
  `<ENGINE>_MAX_QSPLITS = 1000`; `MAX_INSTRS` remains the real backstop.
- **`niyama_fuzzy_search` returned an out-of-range offset.** Under
  `FUZZY_FLAG_UNICODE_NFD` the offset indexed the normalized scratch buffer,
  not the caller's string: a subject of five U+00E9 plus `"dog"` (13 bytes)
  returned **15**. New `_fuzzy_norm_to_orig` translates back by walking the
  original a codepoint at a time; now returns 10, and the result is clamped
  to the subject length regardless.
- **fuzzy silently truncated over-long subjects.** Every `_fuzzy_dp_*` entry
  clamped to `FUZZY_MAX_TEXT_LEN`, so a match past byte 4096 reported "no
  match" with `FUZZY_E_OK` — indistinguishable from a real miss, and
  reachable *inside* the documented limit under NFD because decomposition
  expands the subject. Now reported (see **Added**). `niyama_fuzzy_match`
  additionally rejects the negative error distance, which would otherwise
  have satisfied `d <= max_edits` and returned a **false positive match**.
- **vim `\>` fired at word starts.** `\<` and `\>` both compiled to
  `VIM_OP_BOUNDARY`, the symmetric `\b` test, so each fired at both ends of
  a word; `\>x` matched `"x"` at position 0. `\<foo\>` only looked correct
  because each end is independently a boundary. Now `VIM_OP_WORDBEGIN` (15)
  and `VIM_OP_WORDEND` (16), matching bre's strict semantics since v0.7.0.
- **`\K` inside a lookaround moved the outer match start.** Slot 0 is the
  match start, not a user capture, and positive lookarounds keep their
  sub-captures — so a `\K` inside one rewrote the outer start to a
  zero-width position inside the lookaround, producing an inverted group-0
  span. Slot 0 is now restored on the positive-lookaround success path,
  scoping `\K` as PCRE2 does. Top-level `\K` unchanged.
- Stale in-source documentation corrected: `PCRE_OP_LOOKBEHIND`'s comment
  claimed width and end_pc were bit-packed into arg1 (the emitter and
  matcher put the width in arg2); `src/posix_classes.cyr`'s header still
  described vim as carrying its own copy and called the fold a "v0.9.0
  cleanup" two releases after it happened in v0.8.0;
  `niyama_fuzzy_search`'s comment still described the pre-v0.8.0
  `end - plen` heuristic and cited ADR 0005 as outstanding work.
- `BRE_E_BAD_ANCHOR` is documented as reserved. ADR 0010's freeze table
  marked slot 4 "live" while ADR 0002 called it "reserved — currently
  unused"; the code confirms ADR 0002 (it is never emitted), and the
  declaration now carries the reserved-slot comment pcre's reserved codes
  have.

### Added

- `FUZZY_E_TEXT_TOO_LONG = 4` — subject longer than `FUZZY_MAX_TEXT_LEN`,
  checked after normalization on all four fuzzy entry points. Additive: a new
  value on the existing `niyama_fuzzy_last_error()` accessor, no signature or
  return-value change.
- `PCRE_E_DEPTH_EXCEEDED = 12` — match-time signal that the backtracker hit
  its recursion ceiling, readable via `niyama_pcre_last_error()` after a
  match or search. See **Known limitation**.

### Changed

- **Refactor — duplication consolidated** (per CLAUDE.md § Refactoring
  Policy: three or more instances, measured, same test gates as new code).
  - `_<engine>_at_word_char` was a byte-identical 10-line copy in all four
    engines — the `\w` / `\b` / `\<` / `\>` definition, so a change to the
    character set had to land in four places. Now one
    `_posix_is_word_char` in `src/posix_classes.cyr` with four one-line
    delegates, preserving every call site.
  - The per-thread save-copy loop was hand-inlined **22 times** across the
    three Pike matchers (bre 5, re2 10, vim 7), each repeating the 168-byte
    thread stride and 8-byte header offset. Now `_<engine>_thread_saves_to`.
  - Not done: consolidating the NFA-blob edit primitives
    (`_shift_right` / `_patch_arg1` / `_patch_arg2` / `_shift_targets_one`),
    the class-bitmap helpers, and `_lit_byte`. All are genuine duplication,
    but each is parameterised by engine-specific globals (`_instr_base`,
    `_err`, `_pos`) and sits on the pattern-parsing core. Deferred rather
    than bundled into a release already carrying this many security fixes.
    Note the reviewer's claim that `_lit_byte` is "four byte-identical
    copies" is **not accurate** — the four differ in which engine's error
    constant they set, though all four constants are `= 1`.

### Performance

- **Every one of the 53 benchmark rows is faster or neutral. Zero
  regressions.** Sequential baseline-vs-post-review: mean **−14.29%**,
  median −15.48%, best `fuzzy_medium_pattern_distance` **−33.3%**, worst
  +0.5% (inside its own spread). Confirmed by an interleaved A/B of v1.0.8
  vs v1.0.9 source on the same host (mean −12.81%, median −15.74%).
- These are a **side effect of the leak repairs**, not separate optimization
  work: `_pike_run` was making two *locked* `alloc()` calls per start
  position, so an unanchored N-byte search did 2N locked allocations.
  Removing them is the 5–25% on every `*_search_*` row across the four
  engines. fuzzy's larger 15–27% adds the `_fuzzy_prefold` hoist
  (`_fuzzy_fold` had been recomputed `plen × slen` times for a value
  depending only on the column) and the removal of a per-call 8-byte
  allocation.
- An ASCII fast path was added to `_fuzzy_maybe_normalize`: NFD is provably
  the identity below U+0080, and `str_normalize` allocates ~60× the input
  per call, so an NFD-flagged handle scanning ASCII lines leaked steadily
  for no benefit.
- Full numbers in [`docs/benchmarks.md`](docs/benchmarks.md).

### Measured and rejected

- **Dedup-guarding the per-thread save copy** (16 call sites across
  bre/re2/vim): stage the copy only when `_m_lastgen` shows the target pc is
  not already claimed this generation. Implemented, all 661 assertions
  passed, then measured over 31 rows interleaved — mean −0.29%, median
  −0.12%, every row inside its noise band. The dedup rarely fires at those
  sites, so the guard only adds a branch. **Reverted.**
- The reviewers also flagged "all four engines re-run the matcher from every
  start offset (O(N²))" and "pcre rescans the instruction stream on every
  RECURSE". Both were **refuted** in the verification pass and not acted on.

### Known limitation — pcre recursion depth

`_pcre_match_run` recurses natively for SPLIT and SAVE, so depth scales with
**input position**, not pattern nesting. With `PCRE_MAX_DEPTH = 256`, a
quantifier that must consume more than ~250 positions exhausts it:
`a*$` over 300 `a`s and `(a){200}` over 200 `a`s both report no match
(`(a){100}` over 100 works). Confirmed pre-existing — identical on v1.0.8.

The limit is **not a tunable**: measured against an 8 MB stack,
`_pcre_match_run` frames are ~20 KB and the process SIGSEGVs past ~400
frames, so 256 is already near the ceiling. Raising it trades a false
negative for a crash. The real fix is an explicit heap-allocated backtrack
stack replacing native recursion — a matcher-core rewrite, deferred rather
than carried in this release. Mitigated here by making the condition
observable via `PCRE_E_DEPTH_EXCEEDED`, so a caller can distinguish "no
match" from "gave up".

### Tests / fuzz

- `cyrius test` 6 files / **747 assertions** (was 661; **+86**), 0 failures.
  Every finding above has a regression assertion: brace nesting and capture
  numbering in all four engines, the falsely-rejected `(a){10}` /
  `(?<x>a){2}` / `[a]{0,70}`, negative `from`/`len` on all four
  `search_at`, vim `\<`/`\>` asymmetry, fuzzy over-long rejection and
  NFD offset range, pcre `\K`-in-lookaround and depth observability.
- `cyrius fuzz` 5 harnesses / **1689 assertions**, 0 failures (unchanged).

### Bench

- 53 rows, **0 regressions**, mean −14.29%. See **Performance**.

### ABI summary

- Error codes: `FUZZY_E_TEXT_TOO_LONG = 4` and `PCRE_E_DEPTH_EXCEEDED = 12`
  added (additive; existing accessors). Reserved-but-unused, unchanged:
  `BRE_E_BAD_ANCHOR = 4`, `PCRE_E_LOOKBEHIND_UNSUPPORTED = 2`,
  `PCRE_E_RECURSION_UNSUPPORTED = 4`, `PCRE_E_CONDITIONAL_UNSUPPORTED = 5`.
- Opcodes: `VIM_OP_WORDBEGIN = 15`, `VIM_OP_WORDEND = 16` added. Internal
  encoding — never crosses the public surface.
- The 42 public `niyama_<engine>_*` functions are unchanged in name,
  arity and return semantics. ADR 0010's freeze holds.
- `dist/niyama.cyr` regenerated: 7258 lines (was 6664).


## [1.0.8] — 2026-09-07

Toolchain + vendored-stdlib refresh. No engine source changes; the
`niyama_<engine>_*` surface is untouched (frozen per ADR 0010).

### Changed

- `cyrius` pin bumped 6.5.29 → 6.6.0 — matches the installed toolchain
  wrapper. The 6.5.29 pin had drifted: the wrapper reported
  `manifest-pin: 6.5.29` while `cycc` was already 6.6.0, so every build
  printed `warning: cyrius.cyml pins 6.5.29 but cycc is 6.6.0 —
  toolchain drift`. Note the pin is **advisory for compiler selection** —
  it did not hold builds at 6.5.29; niyama has been compiling under
  6.6.0's `cycc` since the wrapper was installed. The bump is therefore a
  metadata sync that silences the warning, not a compiler change.
- **Vendored `lib/` re-synced from the 6.6.0 snapshot** via
  `cyrius lib sync` (24 files, all byte-identical to
  `~/.cyrius/versions/6.6.0/lib`). 9 files changed content:
  `fmt.cyr`, `io.cyr`, `unicode/casefold.cyr`, `unicode/normalize.cyr`,
  and the five `syscalls_*.cyr` platform leaves. The two `unicode`
  changes are comment/whitespace only — the tables and decode paths
  niyama actually calls are unchanged. Upstream's substantive fixes in
  this range (`fmt_float_buf` carry, `getenv` 8 KB truncation) are in
  modules niyama never calls; `grep` confirms zero `fmt_float` / `getenv`
  references across `src/`, `tests/`, `fuzz/`.
- **Three undeclared files dropped from `lib/`**: `atomic.cyr`,
  `fnptr.cyr`, `result.cyr`. None appear in `[deps].stdlib`, so
  `cyrius lib sync` does not refresh them — they were stale carry-over
  from an older vendor pass, and `result.cyr` in particular was a live
  hazard (see below). They now resolve from the pinned snapshot the same
  way `lib/bench.cyr` always has, so the include graph is unchanged and
  nothing can go stale behind the sync.
- `dist/niyama.cyr` regenerated via `cyrius distlib` at v1.0.8 — 6664
  lines, **byte-identical to v1.0.7 except the version header line**.
- `dist/niyama.deps` — **new file**, auto-generated by `cyrius distlib`
  on 6.6.0 alongside the bundle. A sidecar listing the 9 stdlib leaf
  requirements (`string fmt alloc io vec str syscalls assert unicode`)
  for `cyrius deps` to consume downstream. It belongs with the
  fold-ready artifact, so it is checked in rather than ignored.

### Fixed

- **`tests/niyama.bcyr` did not compile** — the scaffold smoke benchmark
  called `bench("noop", &bench_noop, 1000000)`, but stdlib `lib/bench.cyr`
  exposes no `bench()`; the real API is
  `bench_new()` / `bench_run()` / `bench_report_all()`. It also included
  no stdlib at all, relying on a "Stdlib auto-included via cyrius.cyml"
  comment — but `bench` is not in the declared `[deps].stdlib` set, so it
  could never have resolved. `cycc` refused to emit
  (`1 reachable undefined function(s)`). Rewritten against the real API
  with the same explicit include block the five per-engine harnesses
  carry; it now reports `niyama_noop: ~3ns avg [1000000 iters]`.
  **Pre-existing, not a 6.6.0 regression** — verified to fail identically
  under the 6.5.29 toolchain. It stayed invisible because `cyrius audit`
  was broken from 5.8.65 onward and the documented bench procedure names
  the five per-engine harnesses individually. No engine code involved;
  assertion counts unaffected.

### Security

- **Stale `lib/result.cyr` removed before it could miscompile** (severity
  **LOW**, defence in depth — caught pre-build, never shipped). 6.6.0
  changes `Result` to a `: stack` value form: `Ok(v)` / `Err(e)` return a
  register pair instead of a 16-byte bump allocation, which changes the
  arity of every Result-valued call. `cyrius lib sync` refreshes only the
  declared `[deps].stdlib` subset, and `result.cyr` is not in it — so a
  plain sync would have left 6.6.0's `io.cyr` including a v5.8.28-era
  `result.cyr`, mixing the two ABIs. Avoided by wiping `lib/` before the
  sync rather than syncing over it. **niyama itself has zero exposure** —
  `grep` finds no `Result` / `Ok(` / `Err(` / `is_ok` / `result_unwrap`
  use anywhere in `src/`, `tests/`, `fuzz/`; the risk was confined to the
  vendored tree. Worth recording as a procedure note: **re-vendoring
  `lib/` means wiping it, not syncing over it**, because the sync's scope
  is narrower than the include graph's.

### Tests / fuzz

- No assertion-count change: `cyrius test` 6 files / **661 assertions**,
  `cyrius fuzz` 5 harnesses / **1689 assertions**, both 0 failures —
  identical to the v1.0.7 baseline re-measured immediately before the
  bump. `tests/niyama.bcyr` joins the runnable bench set (6 harnesses,
  was 5).

### Bench

- **No regression.** Naive before/after over the 5 per-engine harnesses
  (53 measurements, 3 runs each, per-row medians) showed 2 rows over the
  ±5% action threshold — `fuzzy_distance_long` (+5.5%) and `fuzzy_match`
  (+6.0%) — against a suspicious **+1.76% mean drift across every row**,
  including `bre`, which touches none of the changed stdlib modules. That
  uniform positive bias is the signature of host drift over the session,
  not of a code change, so the comparison was redone as a **controlled
  interleaved A/B**: same `cycc` (6.6.0), same sources, only `lib/`
  differing, alternating old/new runs 5× each to cancel time-ordered
  drift. Result over the 3 affected engines (`fuzzy`, `pcre`, `re2`, 29
  rows): **0 rows exceed ±5%**, drift mean **−0.12%**, median **−0.23%**,
  largest single drift **+2.5%** (`fuzzy_case_insensitive`, inside its own
  7.2% run-to-run spread). The two flagged fuzzy rows land at −0.8% and
  +0.1% under the control. Confirmed directly: the old-lib tree
  re-measured after the fact gives `fuzzy_match: 462ns` against the
  new-lib 461ns — the "before" 435ns was simply an earlier, faster host
  state.

### Toolchain notes

- **DCE binary: 323,416 B → 323,432 B (+16 B, +0.005%).**
- **Correction to the v1.0.7 entry**, which recorded "the DCE'd smoke
  binary grew 4,544 B → 401,240 B". 401,240 B is the **non-DCE** size on
  6.5.29; that toolchain's DCE build is 323,416 B. Both were re-measured
  here against the 6.5.29 toolchain to attribute the bump honestly. The
  substance of the v1.0.7 note is unaffected — the ~88× growth was still
  `[deps] stdlib` auto-include starting to link at 6.5.16, per ADR 0008's
  `lib/unicode/*_data.cyr` tables.
- **`cyrius audit` works again on 6.6.0.** It was known-broken from
  5.8.65 (missing `~/.cyrius/bin/check.sh`); that file is still absent,
  but 6.6.0's audit no longer depends on it. Two findings, both handled
  above or accepted: the `tests/niyama.bcyr` compile failure (fixed), and
  a `fmt` disagreement on `tests/{bre,re2,pcre,vim}.tcyr`. The `fmt`
  finding is **cosmetic and deliberately not actioned** — `cyrius fmt`
  wants continuation lines flattened to a 6-space indent, replacing the
  align-under-open-paren style used consistently across these files. Same
  accepted-cosmetic-noise category as the 11 long-line `cyrius lint`
  warnings, which are unchanged at 11.
- **`cyrius build` / `cyrius deps` rewrite `cyrius.cyml`'s `description`
  field**, truncating it at the first `;` — the trailing
  "; foldable into stdlib per sandhi pattern" was silently dropped and had
  to be restored. Watch for this on the next bump; it is a toolchain
  manifest-normalisation bug, not an intentional edit.


## [1.0.7] — 2026-08-19

### Changed

- `cyrius` pin bumped 6.4.64 → 6.5.29 — matches installed toolchain wrapper
  (6.4.64 pin had drifted; wrapper printed `manifest-pin: 6.4.64 (drift — wrapper
  is 6.5.29)` on every invocation). No engine source changes — niyama's `[deps]`
  carries no carved-out modules. Verified green on 6.5.29: `cyrius deps --no-lock`
  resolves cleanly (21 stdlib files + 7 under `lib/unicode/`), clean DCE build OK,
  `.tcyr` suite 6 files / 661 assertions, `cyrius fuzz` 5 harnesses / 1689
  assertions, all 5 per-engine bench harnesses (`bre`, `re2`, `pcre`, `fuzzy`,
  `vim`) run clean — 0 failures throughout.
- `dist/niyama.cyr` regenerated via `cyrius distlib` at v1.0.7 (6664 lines,
  byte-identical to v1.0.6 except the version header — pin-only release).

### Fixed

- **`src/main.cyr` smoke banner wrote 2 bytes past the end of its string
  literal.** The `syscall(1, 1, "...", 87)` write length was 87 while the literal
  is 85 bytes, so every run emitted two bytes of adjacent `.rodata` to stdout
  (visible as a trailing `00 00` in `./build/niyama | hexdump -C`). Corrected to
  85. Severity **LOW** (defense in depth): the over-read is a fixed 2-byte
  constant into the program's own read-only segment, is not attacker-influenced
  (niyama's smoke entry takes no input), and cannot reach past the mapped page.
  Pre-existing since v1.0.0; found by the v1.0.7 buffer-safety re-scan per
  CLAUDE.md § Security Hardening step 2. Engine code is unaffected — the defect
  was confined to the smoke entry, which is not part of the frozen
  `niyama_<engine>_*` surface and not included in `dist/niyama.cyr`.
- Smoke banner version string was pinned at `niyama 1.0.0` since the v1.0.0
  freeze; it now reports the real `VERSION` (`niyama 1.0.7`). The replacement is
  length-neutral, so the corrected 85-byte write stays exact.

### Tests / fuzz

- No assertion-count change: `cyrius test` 6 files / **661 assertions**,
  `cyrius fuzz` 5 harnesses / **1689 assertions**, both 0 failures — identical
  to the v1.0.6 baseline re-measured on 6.4.64 immediately before the bump.

### Bench

- **No regression.** All **57** measurements across the 5 harnesses compared
  old-pin (6.4.64) vs new-pin (6.5.29) on the same host, same tree, same
  vendored `lib/`. Single-run sampling showed 14 rows apparently over the ±5%
  action threshold, so each pin was re-run **3×** and compared on per-row
  medians: **0 rows** exceed ±5% once each row's own run-to-run spread is
  accounted for. Aggregate drift mean **+0.04%**, median **+0.20%**. The one
  row still nominally over threshold, `bre_search_class` (+7.0%), has an
  old-side spread of 8.3% — noise, not signal. Both pins report the same
  measured timer floor (~1.34µs/clock-read, subtracted per sample), so the
  comparison is methodologically like-for-like.

### Toolchain notes — binary size grew because `[deps]` finally links

- **The DCE'd smoke binary grew 4,544 B → 401,240 B (~88×) across the bump.**
  The cause is *not* lost dead-code elimination — it is that cyrius **≤ 6.5.15
  silently ignored `[deps] stdlib` auto-include**, and **6.5.16 fixed it**. The
  old binary was small because the declared stdlib was never linked at all.

  Measured with a probe calling `unicode_category(48)` / `unicode_category(65)`
  with **no explicit `include`**, relying solely on `cyrius.cyml [deps]`:

  | toolchain | result |
  |---|---|
  | 6.4.64, 6.5.10, 6.5.14, 6.5.15 | `warning: undefined function 'unicode_category'` ×3, no working binary |
  | 6.5.16, 6.5.17, 6.5.20, 6.5.29 | builds, returns the correct categories (exit 80), 0 warnings |

  Note the old failure mode was a **warning, not an error** — a manifest-only
  build could emit a binary containing an undefined call.

  So the new size is the manifest's 9 declared stdlib modules being linked as
  declared, ~307 KB of which is the `lib/unicode/*_data.cyr` tables. With an
  explicit `include "lib/unicode/categories.cyr"` — the spelling niyama actually
  uses — 6.4.64 links fine and produces 66,920 B, confirming the module itself
  was always reachable when named directly.

- **niyama was never functionally affected**, which is why this went unnoticed
  through v1.0.6. Every `tests/*.tcyr` and every engine consumer uses explicit
  `include` lines (`lib/unicode/categories.cyr`, `casefold.cyr`, `normalize.cyr`,
  …), and `src/main.cyr` references no stdlib at all — it only uses the `syscall`
  builtin. Nothing ever depended on the broken auto-include path.

- **Residual observation** (not acted on): 6.5.16+ links every module the
  manifest declares whether or not the entry point references it, so the smoke
  binary carries the full unicode tables it never calls. That is defensible
  link-as-declared behaviour rather than a defect, and `[deps]` is shared with
  the library surface, so it stays as-is. Recorded because binary size is a
  tracked release metric per CLAUDE.md § CI/Release.

  Scope note: this affects the **smoke binary only**. niyama ships as a library;
  its consumer-facing artifact is the `dist/niyama.cyr` source bundle, whose size
  is unchanged.

## [1.0.6] — 2026-07-16

### Changed

- `cyrius` pin bumped 6.2.1 → 6.4.64 — matches installed toolchain wrapper
  (6.2.1 pin had drifted; wrapper printed `manifest-pin: 6.2.1 (drift — wrapper
  is 6.4.64)` on every invocation). Zero source changes — niyama's `[deps]`
  carries no carved-out modules. Verified green on 6.4.64: `cyrius deps` resolves
  cleanly, DCE build OK, `.tcyr` suite 6 files / 661 assertions, `cyrius fuzz`
  5 files / 1689 assertions, all 5 per-engine bench harnesses (`bre`, `re2`,
  `pcre`, `fuzzy`, `vim`) run clean, all 0 failures.
- `dist/niyama.cyr` regenerated via `cyrius distlib` at v1.0.6 (6664 lines,
  byte-identical to v1.0.5 except the version header — pin-only release).

## [1.0.5] — 2026-06-12

### Changed

- `cyrius` pin bumped 6.1.27 → 6.2.1 (ecosystem-wide stdlib pin sweep onto the
  current toolchain). No source changes — niyama's `[deps]` carries no carved-out
  modules. Verified green on 6.2.1: `cyrius deps` resolves cleanly, `.tcyr` suite
  109/109, bench 6/6, `dist/niyama.cyr` regenerated via `cyrius distlib`.

## [1.0.4] — 2026-06-10

### Changed

- `cyrius` pin bumped 6.0.1 → 6.1.27 — matches installed
  toolchain wrapper (6.0.1 pin had drifted; wrapper printed
  `manifest-pin: 6.0.1 (drift — wrapper is 6.1.27)` on every
  invocation). Zero source changes; build/test/fuzz green
  byte-for-byte against 6.0.1 baseline (109 tests + 6 build
  tests, 5 fuzz suites, 0 failures).
- `dist/niyama.cyr` regenerated via `cyrius distlib` at v1.0.4
  (6664 lines, unchanged from v1.0.3 — pin-only release).

## [1.0.3] — 2026-05-21

### Changed

- `cyrius` pin bumped 5.11.4 → 6.0.1 — matches installed
  toolchain wrapper (5.11.4 pin had drifted; wrapper printed
  `manifest-pin: 5.11.4 (drift — wrapper is 6.0.1)` on every
  invocation). Zero source changes; build/test/fuzz green
  byte-for-byte against 5.11.4 baseline (109 tests, 5 fuzz
  suites, 0 failures).
- `dist/niyama.cyr` regenerated via `cyrius distlib` at v1.0.3
  (6664 lines, unchanged from v1.0.2 — pin-only release).

### Fixed

- **Release upload of `cyrius.lock` no longer fails.** Cyrius
  6.0.1's `cyrius deps` truncates `cyrius.lock` to 0 bytes when
  there are no external deps to lock (niyama is stdlib-only per
  CLAUDE.md "Rules"), and GitHub's release-asset API rejects
  0-byte assets with `size must be greater than or equal to 1`.
  Workaround mirrors the yukti / patra pattern: commit a stub
  `cyrius.lock`, run `cyrius deps --no-lock` in CI / release so
  the toolchain doesn't overwrite it. Drop `--no-lock` once the
  upstream lockfile writer is fixed.

### Added

- `cyrius.lock` (comment-only stub) is now shipped as a release
  asset alongside the source tarball, dist bundle, and smoke
  binaries; checksum included in `SHA256SUMS`.

## [1.0.2] — 2026-05-11

### Changed

- **Stdlib annotation pass**: every public fn in `src/*.cyr`
  carries a `: i64` return-type annotation. Mechanical pass
  matching cyrius's v5.11.x annotation arc; parse-only, zero
  runtime / codegen change.
- `cyrius` pin bumped 5.8.65 → 5.11.4 — required for `: i64`
  return-type syntax (v5.10.x REAL TYPE SYSTEM).
- `dist/niyama.cyr` regenerated via `cyrius distlib` at v1.0.2
  (6664 lines). Ready for next cyrius-side fold-in slot.

## [1.0.1] — 2026-05-06

**Fold-trigger release + bundled-dist correction.**

The v1.0.0 ship had a defect: `dist/niyama.cyr` was the
108-line `include`-manifest scaffold (header said "Do not edit
-- rebuild with: scripts/build-dist.sh (post-M1)" but the
build-dist.sh never landed). Vendoring that file as
`lib/niyama.cyr` would dangle on `include "src/posix_classes.cyr"`
from the consumer's lib/ path. v1.0.1 fixes the dist via
`cyrius distlib` — the canonical Cyrius bundling tool — yielding
a proper inlined 6,664-line single-file artifact.

### Fixed

- **Bundled dist** (`dist/niyama.cyr`): regenerated via
  `cyrius distlib` against new `[lib] modules = [...]` block
  in `cyrius.cyml`. Inlines all 7 modules (posix_classes,
  unicode_props, bre, re2, pcre, fuzzy, vim) into a single
  artifact ready for byte-identical fold-in. v1.0.0's manifest
  variant retired; sandhi/sakshi/patra/sigil precedent now
  honored.

### Planning

- **ADR 0011 — Triggered.** Fold trigger fires at cyrius v5.9.0
  ship. AGNOS-lineage consumer gate met by:
  1. cyim (consumer #1, active).
  2. AGNOS bare-metal kernel (consumer #2, queued for
     cyrius v5.10.0; long-horizon-confirmed pin).
  Cyrius v5.9.0 vendors `lib/niyama.cyr` byte-identical from
  this release. niyama-the-repo enters fold-maintenance mode:
  v1.x patches still land here, propagate via cyrius update;
  post-fold extensions land in cyrius stdlib's vendored copy.

## [1.0.0] — 2026-05-05

**Fold-ready release.** Public surface locked per ADR 0010.
v1.0 ships fold-ready but does not fold at this release per
ADR 0011 — cyrius stdlib vendors `dist/niyama.cyr` as
`lib/niyama.cyr` byte-identical when a second long-horizon
consumer materializes (cyim is #1). niyama-the-repo enters
maintenance mode; v1.x patches are bug-fix only.

This release sweeps the v0.9.0 review remainders (LOW-severity
items) and rounds out documentation completeness per the
agnosticos `example_claude.md` template.

### Planning

- **ADR 0011 — Accepted.** Fold readiness + post-v1.0 fold
  trigger. v1.0 is fold-ready (frozen surface ✓, audit clean ✓,
  comprehensive tests + fuzz + bench history ✓), but actual fold
  awaits the second long-horizon AGNOS-lineage consumer per ADR
  0001's gate. Trigger checklist documented for the future
  consumer-#2 moment.
- **CLAUDE.md completeness.** Added the 5 sections deferred from
  v0.9.0 P(-1) step 8: Cyrius Conventions, CI / Release,
  Documentation Structure, .gitignore (Required), CHANGELOG
  Format. niyama CLAUDE.md now matches the agnosticos
  `example_claude.md` template.
- **`docs/api/README.md`** — first consolidated public API
  reference. Mirrors ADR 0010's freeze contract in
  human-readable form per engine. Useful for consumers vendoring
  `dist/niyama.cyr`.

### Added

- **`tests/re2.tcyr` + `tests/pcre.tcyr`** — `(?i)ß`
  semantics-lock tests (C2). Locks the v0.8.0 1:1-fold-only
  behavior so a future implementer cannot silently swap to
  `unicode_fold` (full-fold) without an ADR.
- **`tests/pcre.tcyr`** — recursion + lookaround nesting tests
  (C4). Exercises `_pcre_recurse_stop_pc` save/restore at
  LOOKAHEAD / LOOKBEHIND / ATOMIC entry. 4 nested-pattern
  cases.
- **`tests/{bre,re2,pcre,fuzzy,vim}.tcyr`** — empty-pattern
  semantics-lock tests (C5). All 5 engines: empty pattern
  compiles, matches empty input, matches at start of any input
  (fuzzy: distance grows linearly with input length).

### Changed

- `.gitignore` — added release-artifact ignores (`cyrius-*.tar.gz`,
  `*.tar.gz`, `SHA256SUMS`) per CLAUDE.md § .gitignore section.
  niyama-specific note: `dist/` and `lib/` remain checked in
  (vendored stdlib + fold-ready artifact).
- `state.md` — engine-selection rubric extended with v1.0 known
  asymmetries section: fuzzy `_search_at` gap (A1), long
  property name limitations (G3), absolute anchor `\A`/`\z`/`\Z`
  porting note (G4), `(?i)` 1:1-fold-only semantics, and
  `compile_opts` asymmetry across engines.

### Removed

- `docs/development/v0.9.0-review-findings.md` — working doc
  for the v0.9.0 P(-1) review pass; lifecycle-bound, deleted
  at v1.0 ship per its own header note. Findings consolidated
  into the v0.9.0 + v1.0 CHANGELOG sections, the security
  audit, and the architecture notes.

### Tests / fuzz

- `tests/bre.tcyr` — **112** (was 109; +3 empty-pattern).
- `tests/re2.tcyr` — **175** (was 169; +3 ß semantics-lock,
  +3 empty-pattern).
- `tests/pcre.tcyr` — **206** (was 188; +3 ß, +12 recursion
  nesting, +3 empty-pattern).
- `tests/fuzzy.tcyr` — **57** (was 53; +4 empty-pattern).
- `tests/vim.tcyr` — **109** (was 106; +3 empty-pattern).
- `tests/niyama.tcyr` — 2 (unchanged).
- Aggregate: **6 files, 661 assertions, all passing** (was 627;
  +34).
- Fuzz: 1689 assertions, unchanged from v0.9.0 (no engine code
  changes in v1.0).

### Bench

No engine code changes in v1.0; bench numbers carry forward from
v0.9.0 unchanged. See `docs/benchmarks.md`.

### v1.0 surface (frozen — see ADR 0010)

- **5 public engines** (bre, re2, pcre, fuzzy, vim) with their
  per-engine ABIs locked.
- **39 public function symbols** across the engines (per
  `docs/api/README.md`).
- **34 distinct error code values** across the 5 engines, plus
  4 reserved-but-unused slots (3 in pcre, 0 in others) for ABI
  stability.
- **All `MAX_*` capacity limits frozen** (`MAX_INSTRS = 4096`,
  `MAX_CLASSES = 64`, `MAX_SAVES = 20`, `MAX_NAMES = 9`, fuzzy
  `MAX_PAT_LEN = 256`, `MAX_TEXT_LEN = 4096`).
- **Semantic invariants frozen**: re2/bre/vim no-backref through
  v1.0 (vim post-fold revisit pinned per ADR 0009); pcre as the
  only backtracking engine with step + depth bounds; Pike NFA
  linear-time guarantee for accepted patterns.

### Post-v1.0

niyama-the-repo enters maintenance mode. Bug-fix patch releases
(v1.0.x) only — no surface changes. Post-fold extensions per ADR
0010's evolution model land in cyrius stdlib's vendored
`lib/niyama.cyr` once the fold trigger fires (ADR 0011).

Pinned post-fold extension candidates:
- vim backref `\1`-`\9` (ADR 0009, decision-gated on cyim ask).
- fuzzy `niyama_fuzzy_search_at` (A1 review finding).
- Long Unicode property names + Unicode scripts (G3).
- `(?i)` full case folding via `unicode_fold` (C2 lock allows
  the upgrade post-fold with a new ADR).

## [0.9.0] — 2026-05-05

M5 P(-1) hardening + surface freeze release. Last release before
v1.0 fold-ready. Per ADR 0010, the public `niyama_<engine>_*` API
surface is locked through v1.0; post-fold extensions land via
cyrius stdlib once the fold gate is met.

The 9-step P(-1) pass per
[CLAUDE.md § Process](CLAUDE.md#p1-scaffold--project-hardening-before-any-new-features)
ran clean: cleanliness baseline → bench baseline → deep review →
PCRE2 CVE cross-check → security audit → fixes → post-review
benchmarks → freeze ADR → closeout. **No CRITICAL/HIGH findings.**

### Planning

- **ADR 0009 — Accepted.** bre/vim backref review concluded as
  asymmetric split: bre permanently out (v1.0 + post-fold), vim
  out for v1.0 with explicit post-fold revisit via cyrius stdlib
  `lib/niyama.cyr` if the fold gate is met. re2 structurally
  backref-free, permanent. Containment design captured for the
  post-fold vim extension. ADR 0002 + ADR 0006 footnotes updated.
- **ADR 0010 — Accepted.** Surface freeze. Locks public API,
  error-code numbering, opcode IDs, capacity limits, and semantic
  invariants for v1.0. Post-freeze evolution model documented:
  v1.x patch-only, post-fold via cyrius stdlib for additive
  extensions, v2.0 hypothetical for non-additive changes.
- **`docs/audit/2026-05-05-audit.md`** — first formal niyama
  security audit. 8-item checklist walked per first-party-standards;
  no CRITICAL/HIGH findings. Two MEDIUM items (boundary test
  coverage C1, invalid-UTF-8 input fuzz coverage C3) addressed
  in this release.
- **`docs/development/v0.9.0-review-findings.md`** — internal
  deep-review working doc. Findings split into v0.9.0 primaries
  vs v1.0 remainders per user direction. Deleted at v1.0.
- **`docs/benchmarks.md`** — first comprehensive bench history.
  v0.8.0 baseline + v0.9.0 step-7 diff (no regressions, all
  rows within ±2.5% noise).
- **`docs/architecture/`** — first 4 architecture notes:
  position-stepping asymmetry, save-table layout, class-bitmap
  scope, no-shared-matcher-kernel.

### Added

- **`niyama_re2`** — POSIX bracket classes `[[:alpha:]]`,
  `[[:digit:]]`, `[[:alnum:]]`, `[[:space:]]`, `[[:upper:]]`,
  `[[:lower:]]`, `[[:xdigit:]]`, `[[:punct:]]`, `[[:print:]]`,
  `[[:graph:]]`, `[[:cntrl:]]`, `[[:blank:]]`. **Fixes ADR 0007
  oversight** — bre/pcre/vim got POSIX classes in v0.7.0 / v0.8.0;
  re2 was missed. v0.9.0 closes the gap by hooking
  `_re2_parse_class` into the shared `src/posix_classes.cyr`
  module. ~10 lines of new parser code; no new opcode
  (re-uses existing `RE2_OP_CLASS`).
- **Boundary tests per regex engine** — bre / re2 / pcre / vim
  test groups verifying `MAX_*` limits error cleanly. Locks the
  freeze contract.
- **Invalid-UTF-8 input fuzz seeds** — re2 / pcre / vim fuzz
  harnesses gain explicit malformed-UTF-8 input cases (lone
  continuation byte, truncated 2/3/4-byte sequences, invalid
  leading byte, lone surrogate, mixed valid+invalid). Validates
  the codepoint-stepping outer loop's `_uc_decode_utf8` fallback
  handles invalid bytes without crashing or looping.

### Tests / fuzz

- `tests/bre.tcyr` — **109** (was 107; +2 boundary).
- `tests/re2.tcyr` — **169** (was 147; +18 POSIX classes,
  +4 boundary).
- `tests/pcre.tcyr` — **188** (was 185; +3 boundary).
- `tests/fuzzy.tcyr` — **53** (unchanged).
- `tests/vim.tcyr` — **106** (was 104; +2 boundary).
- `tests/niyama.tcyr` — **2** (unchanged).
- Aggregate: **6 files, 627 assertions, all passing** (was 598,
  +29).
- Fuzz: `bre 215` / `fuzzy 757` / `re2 241` (+11) / `pcre 250`
  (+11) / `vim 226` (+7). Aggregate: **5 files, 1689 assertions,
  all passing** (was 1660, +29).

### Bench (regression check)

All 47 measurements within **±2.5%** of v0.8.0 baseline. No
regressions. Detail in `docs/benchmarks.md`.

The B4 review finding (pcre in-loop saves alloc) showed no
measurable impact at the v0.8.0 bench corpus; deferred to
post-fold per the gating decision.

### Frozen at v1.0 surface (per ADR 0010)

- All `niyama_<engine>_*` public APIs.
- All `<ENGINE>_E_*` error code numeric values (incl.
  reserved-but-unused slots: `PCRE_E_LOOKBEHIND_UNSUPPORTED`,
  `PCRE_E_RECURSION_UNSUPPORTED`, `PCRE_E_CONDITIONAL_UNSUPPORTED`).
- All capacity limits (`MAX_INSTRS = 4096`, `MAX_CLASSES = 64`,
  `MAX_SAVES = 20`, `MAX_NAMES = 9`, `FUZZY_MAX_PAT_LEN = 256`,
  etc.).
- Semantic invariants: re2/bre/vim no-backref, pcre as the only
  backtracking engine, Pike NFA linear-time guarantees, fuzzy
  byte-Levenshtein.
- Opcode IDs per engine (internal but encoded in `dist/niyama.cyr`
  artifact).

### Deferred to v1.0

Per the v0.9.0 review's primaries-vs-remainders split:

- **A1** — fuzzy `_search_at` ABI gap (post-fold candidate per
  ADR 0010).
- **C2** — `(?i)ß` ASCII-only-fold semantics-lock test.
- **C4** — recursion + lookaround nesting tests (correctness
  verified by inspection in audit; tests come at v1.0).
- **C5** — empty-pattern tests across engines.
- **F2** — public API listing → `docs/api/`.
- **CLAUDE.md** completeness — Cyrius Conventions, CI/Release,
  Documentation Structure, `.gitignore`, CHANGELOG Format
  sections from agnosticos `example_claude.md` template.
- **Fold ADR** itself (template: sandhi ADR 0002).

## [0.8.0] — 2026-05-05

M4.5 completion release. Per ADR 0008's Unicode-stdlib pivot, every
remaining M4.5 deferral ships in one bundle (collapsed from a
four-sub-release plan at user direction — don't fragment a coherent
milestone). Cyrius 5.8.65's `lib/unicode/` carries the UCD tables
niyama would have built; `src/unicode_props.cyr` is the small new
parser module that wires `\p{NAME}` syntax onto the stdlib lookups.

### Changed

- **Toolchain pin** bumped from `5.8.42` → `5.8.65` in `cyrius.cyml`.
  Required floor: stdlib `lib/unicode/` arrived at .49/.50/.51, codec
  lift at .55, NFKC/NFKD at .60. ADR 0008 records the dependency.
- **`[deps].stdlib`** adds `unicode` — vendors
  `lib/unicode/{categories,casefold,normalize,_decode,_*data}.cyr`
  into `lib/` via `cyrius update`.
- **niyama_re2 / niyama_vim**: matcher loop is now codepoint-stepped
  (was byte-stepped). For ASCII inputs this is a no-op; for multi-byte
  UTF-8, `.`/`\d`/`\w`/`\s`/`[abc]` advance by full codepoint length
  rather than splitting a multi-byte sequence. niyama_pcre stays
  byte-stepped; UPROP and UCHAR_CI handle their own multi-byte
  decode internally.
- **niyama_re2 / niyama_vim**: literal multi-byte chars in patterns
  now compile to a single UCHAR opcode (one cp per opcode) instead
  of byte-by-byte CHAR sequences. Fixes a regression that
  codepoint-stepping would have introduced — pattern `α` no longer
  splits across two CHAR(0xCE)/CHAR(0xB1) opcodes.

### Added

- **`src/unicode_props.cyr`** — shared `\p{NAME}` / `\P{NAME}` parser
  helper. Recognizes 7 single-letter aggregate categories
  (`L M N P S Z C`) and 30 two-letter leaf categories
  (`Lu Ll Lt Lm Lo Mn Mc Me Nd Nl No Pc Pd Ps Pe Pi Pf Po Sm Sc Sk
  So Zs Zl Zp Cc Cf Cs Co Cn`). Returns a 30-bit GeneralCategory
  bitmask; matchers test via stdlib `unicode_category(cp)`. Long
  property names (`\p{Letter}`, `\p{Greek}`, etc.) and Script /
  Block properties are out of scope; pinned v0.8.x if a consumer
  asks.
- **niyama_re2** (clears 1 of 1 ADR 0003 deferral):
  - Unicode property classes `\p{L}` / `\P{L}` etc. via two new
    opcodes `RE2_OP_UPROP` (= 16), `RE2_OP_NUPROP` (= 17). Mask
    packed into upper 32 bits of word0.
  - Multi-byte literal pattern char support — `RE2_OP_UCHAR` (= 18)
    stores a codepoint; emitted whenever the parser sees a non-ASCII
    literal. Codepoint-stepped matcher advances by UTF-8 length.
  - `(?i)` Unicode case-fold upgrade — under `(?i)`, multi-byte
    literals emit `RE2_OP_UCHAR_CI` (= 19) which stores
    `unicode_to_lower(cp)`; matcher folds input cp the same way and
    compares. ASCII path unchanged (still uses `RE2_OP_CHAR_CI`).
    1:1 case mappings only; full case folding (`ß`↔`SS` etc.) is
    not in scope for v0.8.0.
  - New error code `RE2_E_BAD_PROPERTY = 8` for unrecognized
    `\p{NAME}`. Frozen-at-M2 codes 0..7 unchanged.
- **niyama_pcre** (clears 2 of 3 ADR 0004 deferrals):
  - Unicode property classes — `PCRE_OP_UPROP` (= 24),
    `PCRE_OP_NUPROP` (= 25). Top-level `\p{L}` works; inside a
    char class (`[\p{L}]`) still rejected — char classes are byte
    bitmaps, folding cp-properties in is out of scope.
  - `(?i)` Unicode case-fold upgrade — `PCRE_OP_UCHAR_CI` (= 26)
    for multi-byte literals under `(?i)`, mirroring re2.
  - Lookbehind `(?<=...)` `(?<!...)` — `PCRE_OP_LOOKBEHIND` (= 28),
    `PCRE_OP_NLOOKBEHIND` (= 29). Fixed-width compile-time analysis
    via new `_pcre_compute_width` helper. Variable-width bodies
    (alternation with mismatched arm widths, quantifiers, `\p{L}`,
    backref) reject with new error `PCRE_E_LOOKBEHIND_VARWIDTH`
    (= 10). PCRE2 10.43 model.
  - Recursion `(?R)` (whole pattern) and `(?P>NAME)` (named group)
    — `PCRE_OP_RECURSE` (= 30) plus a `_pcre_recurse_stop_pc`
    matcher global for stop-at-close-save semantics. Numeric
    `(?N)` syntax not in v0.8.0; pin v0.8.x if a consumer asks.
    Saves snapshot/restored across the recursive call so inner
    captures don't leak (PCRE2-compatible).
  - New error codes:
    - `PCRE_E_BAD_PROPERTY = 9` (unrecognized `\p{NAME}`)
    - `PCRE_E_LOOKBEHIND_VARWIDTH = 10` (variable-width body in
      `(?<=...)` / `(?<!...)`)
    - `PCRE_E_BAD_RECURSION_REF = 11` (`(?P>NAME)` referencing
      undefined group)
  - Frozen-but-narrowed slots: `PCRE_E_LOOKBEHIND_UNSUPPORTED` (= 2)
    no longer emitted at top level; `PCRE_E_UNICODE_PROP_UNSUPPORTED`
    (= 3) now narrows to "inside `[...]` only";
    `PCRE_E_RECURSION_UNSUPPORTED` (= 4) reserved-but-unused. ABI
    stability per ADR 0007's reserved-slot precedent.
- **niyama_fuzzy**:
  - `FUZZY_FLAG_UNICODE_NFD` (= 2) — pattern + input both pass
    through stdlib `str_normalize(s, NFD)` before the Levenshtein
    DP. `café` (NFC) and `café` (NFD) become equivalence-classed
    at distance 0.
  - Exact start-position recovery in `niyama_fuzzy_search` — new
    `_fuzzy_recover_start` reverse-DP pass replaces the
    `end_pos - plen` heuristic. Same O(plen × end_pos) complexity
    as the forward pass.
  - New error code `FUZZY_E_NFD_OVERFLOW = 3` for patterns whose
    NFD form exceeds `FUZZY_MAX_PAT_LEN`.
- **niyama_vim**:
  - Unicode property classes `\p{L}` etc. — `VIM_OP_UPROP` (= 12),
    `VIM_OP_NUPROP` (= 13). Codepoint-stepped matcher matches re2.
  - Multi-byte literal pattern char support — `VIM_OP_UCHAR` (= 14).
  - New error code `VIM_E_BAD_PROPERTY = 5`.
- **vim → src/posix_classes.cyr refactor** — `_vim_match_posix_class`
  and `_vim_class_apply_posix` now delegate to the shared module,
  deleting ~85 lines of duplicate POSIX-class fillers + name
  recognizer. Pre-v0.8.0 vim carried its own copy as deliberate
  duplication (per ADR 0007's risk-bounded v0.7.0 carve); the fold
  was the v0.9.0-was cleanup, pulled forward into v0.8.0 since
  ADR 0008 collapsed the M4.5 sub-releases.

### Deferred

- **bre / vim backref `\1`-`\9`** — surfaced at v0.8.0 ship per the
  M1 / M4 "potentially post-v1.0; document, don't skip" call. User
  direction: **v0.8.1 slot collapses (no release); review pin moves
  to v0.9.0 with broader scope** — not just "implement yes/no" but
  the exposure surface (ABI shape, kernel vs. per-engine policy,
  error-code reuse, re2 guarantee preservation, consumer impact).
  v0.9.0 deliverable is the review itself; whether code lands
  depends on what it concludes.

### Tests / fuzz

- `tests/bre.tcyr` — 107 (unchanged).
- `tests/re2.tcyr` — **147** (was 101): +37 unicode-prop, +5 (?i)
  Unicode, +4 multi-byte literals.
- `tests/pcre.tcyr` — **185** (was 140): +21 unicode-prop, +5 (?i)
  Unicode, +12 lookbehind, +12 recursion (with the 4 deferred-rejection
  tests removed since features now land).
- `tests/fuzzy.tcyr` — **53** (was 45): +4 NFD, +4 exact-start.
- `tests/vim.tcyr` — **104** (was 88): +13 unicode-prop, +3 multi-byte.
- Aggregate: **6 files, 598 assertions, all passing** (was 483, +115).
- Fuzz: **1660 assertions** (was 1658, +2 from refined pcre rejection
  invariants).

### ABI summary (frozen-at-M3 numbers preserved per ADR rules)

- New opcodes by engine: re2 +4 (UPROP, NUPROP, UCHAR, UCHAR_CI),
  pcre +6 (UPROP, NUPROP, UCHAR_CI, LOOKBEHIND, NLOOKBEHIND, RECURSE),
  vim +3 (UPROP, NUPROP, UCHAR).
- New error codes: re2 +1 (BAD_PROPERTY = 8), pcre +3 (BAD_PROPERTY
  = 9, LOOKBEHIND_VARWIDTH = 10, BAD_RECURSION_REF = 11), fuzzy +1
  (NFD_OVERFLOW = 3), vim +1 (BAD_PROPERTY = 5).
- Reserved (not emitted but slot kept): pcre slot 2
  (LOOKBEHIND_UNSUPPORTED), pcre slot 4 (RECURSION_UNSUPPORTED).

## [0.7.0] — 2026-05-03

M4.5 first-of-three catch-up release: the no-Unicode-dep slice. Per
ADR 0007, eight features land across bre/re2/pcre, all sharing one
small new module `src/posix_classes.cyr`. Lookbehind + pcre
recursion remain deferred to v0.8.0; Unicode work to v0.9.0.

### Added

- **`src/posix_classes.cyr`** — shared ASCII fillers + name
  recognizer for the 12 POSIX bracket classes (`alpha`, `digit`,
  `space`, `upper`, `lower`, `alnum`, `blank`, `cntrl`, `graph`,
  `print`, `punct`, `xdigit`). Used by bre + pcre. (vim still
  carries an in-engine copy from M4 — folding is a v0.9.0
  cleanup.)
- **niyama_bre**:
  - GNU `\<` / `\>` word boundaries with **strict** semantics —
    distinct from `\b`. Two new opcodes `BRE_OP_WORDBEGIN` (= 10)
    and `BRE_OP_WORDEND` (= 11). Matches `grep -G` traditional
    behavior.
  - POSIX bracket classes `[[:alpha:]]` etc. via the shared
    module. Unknown class names → `BRE_E_SYNTAX`.
- **niyama_re2**:
  - Named captures `(?<NAME>...)` and `(?P<NAME>...)` with
    `niyama_re2_group_by_name(nfa, name)` lookup. Mirrors the
    pcre name-table mechanism (40-byte slots, 9-name max).
  - Inline flags `(?i)`, `(?m)`, `(?s)` and combinations
    (e.g. `(?ims)`). Pattern-wide effect from declaration onward.
  - Four new opcodes — `RE2_OP_CHAR_CI` (= 12), `RE2_OP_BOS`
    (= 13), `RE2_OP_EOS` (= 14), `RE2_OP_ANY_NONL` (= 15).
  - New error code `RE2_E_DUPLICATE_NAME = 7` (next-available;
    M2-frozen codes 0..6 unchanged).
- **niyama_pcre**:
  - POSIX bracket classes via the shared module.
  - Inline flags `(?i)`, `(?m)`, `(?s)` + combinations.
  - `\K` reset-match-start — emits `SAVE 0`, overrides implicit
    match-start save (same mechanism as vim's `\zs`).
  - Branch-reset groups `(?|...)` — alternatives reuse capture
    numbers.
  - Conditional patterns `(?(N)yes|no)` and `(?(<NAME>)yes|no)`
    via new opcode `PCRE_OP_COND` (= 22). Matcher consults saves
    array at runtime to dispatch.
  - Callouts `(?C)` and `(?C<num>)` — observability only via new
    opcode `PCRE_OP_CALLOUT` (= 23) and
    `niyama_pcre_last_callout()` accessor. No callback API.
  - Six new opcodes total (`PCRE_OP_CHAR_CI` = 18,
    `PCRE_OP_BOS` = 19, `PCRE_OP_EOS` = 20,
    `PCRE_OP_ANY_NONL` = 21, `PCRE_OP_COND` = 22,
    `PCRE_OP_CALLOUT` = 23).
  - New error code `PCRE_E_BAD_CONDITION = 8` for unrecognized
    condition forms or unknown named references in
    `(?(...)...)`. The `PCRE_E_CONDITIONAL_UNSUPPORTED = 5`
    slot is **frozen but no longer emitted** — kept reserved per
    ABI freeze rules.
- ADR 0007 — records the v0.7.0 carve, what's in vs. deferred,
  the strict-default behavior change, and the cross-engine
  uniformity gains.
- Tests — 121 new assertions across `tests/{bre,re2,pcre}.tcyr`:
  bre word boundaries (9), bre POSIX classes (28), re2 named
  captures (12), re2 inline flags (12), re2 strict defaults (3),
  pcre POSIX classes (8), pcre inline flags (12), pcre strict
  defaults (3), pcre `\K` (4), pcre branch-reset (8), pcre
  conditional (10), pcre callout (8), plus a handful of supporting
  assertions. Aggregate test count: 362 → 483.
- Fuzz — 22 new assertions across
  `fuzz/{bre,re2,pcre}.fcyr`: extended pattern alphabets to
  exercise the new parser branches (POSIX bracket bodies, inline
  flags, named captures, branch-reset, conditional, callout) and
  invariant checks on duplicate-name rejection (re2 + pcre) and
  bad-condition rejection (pcre). Aggregate fuzz count: 1636 →
  1658.

### Changed

- **niyama_re2 BEHAVIOR CHANGE** — `^`, `$`, and `.` are now
  **strict-by-default** (RE2-spec compliant). `^` matches only at
  pos 0; `$` only at pos len; `.` excludes `\n`. Multi-line
  semantics require `(?m)`; dot-matches-newline requires `(?s)`.
  Pre-v0.7.0 implementations were silently loose. Existing tests
  didn't exercise distinguishing inputs (no `\n`), so no test
  regressed; downstream consumers relying on loose semantics will
  see the change. Pre-surface-freeze (M5) is the right time to
  fix this; M5 freezes semantics until v1.0.
- **niyama_pcre BEHAVIOR CHANGE** — same strict-default fix as re2,
  for the same reason. PCRE-spec compliant.
- niyama_bre's bracket parser routed through the shared
  `_posix_match_class_name` recognizer instead of falling through
  to syntax error on `[[:` prefixes.
- niyama_pcre's bracket parser routed through the shared
  `_posix_match_class_name` recognizer; previously rejected
  POSIX-class brackets with `PCRE_E_SYNTAX`.
- `RE2_NFA_HEADER_SIZE` extended 216 → 256 to make room for the
  name-table offset/count words at offsets 216/224.
- `dist/niyama.cyr` bundle now includes
  `src/posix_classes.cyr` ahead of the engine modules.

### Notes

- niyama_bre and niyama_vim **keep their pre-v0.7.0 loose
  defaults** for `^`/`$`/`.` (vim is loose by spec; POSIX BRE
  matches `grep` traditional behavior). The strict-default fix is
  re2/pcre-only.
- Backreferences `\1`-`\9` in bre and vim **remain rejected**
  per the explicit M1/M4 "potentially post-v1.0; document, don't
  skip" calls. Not revisited in v0.7.0.
- `(?-i)` negated inline flags and `(?i:...)` scoped forms are
  out of scope; the `(?ims)` pattern-wide spelling covers ~95%
  of real-world use.

## [0.6.0] — 2026-05-03

M4 — fifth and final pre-catch-up engine: vim (`niyama_vim_*`).
vim/cyim flavor with all four magicness modes, `\<`/`\>` word
boundaries, `\zs`/`\ze` match-position markers, and POSIX bracket
classes `[[:alpha:]]` etc. Pike NFA matcher (fork of re2) — `\1`-`\9`
backref rejected by default per ADR 0006, flagged for v0.9.0 review
alongside the bre backref question.

### Added

- `src/vim.cyr` — Pike NFA matcher with vim-flavor parser. ~1100
  lines. Mode-dependent character dispatch covers all four magicness
  variants without duplicating engine logic.
- Public ABI mirroring the established per-engine shape:
  - `niyama_vim_compile(pat)` — defaults to `VIM_MODE_MAGIC`.
  - `niyama_vim_compile_opts(pat, mode)` — explicit mode flag.
  - `niyama_vim_match` / `_search` / `_search_at` /
    `_group_start` / `_group_end` / `_last_error`.
- **All four magicness modes** as opts-flag-controlled values:
  - `VIM_MODE_VERY_MAGIC` (= 0): all metachars special bare.
  - `VIM_MODE_MAGIC` (= 1, default): `*` `.` `[` special bare;
    `\(` `\|` `\+` `\?` `\=` `\{` `\}` need backslash.
  - `VIM_MODE_NOMAGIC` (= 2): only `^` `$` special bare; `\.` `\*` `\[`
    for those.
  - `VIM_MODE_VERY_NOMAGIC` (= 3): nearly everything literal; `\^` `\$`
    needed even for anchors.
- vim feature set: `\<`/`\>` word boundaries, `\zs`/`\ze`
  match-position markers, `\d`/`\D`/`\w`/`\W`/`\s`/`\S` predefined
  classes, brace quantifiers `\{n,m\}` greedy AND `\{-n,m\}` lazy
  (vim's lazy syntax), `\+` `\?` `\=` quantifiers (and bare `+` `?`
  `=` in very-magic), `\(...\)` groups (and `(...)` in very-magic),
  `\|` alternation (and `|` in very-magic), bare/escape-flipped
  forms throughout.
- POSIX bracket classes inside `[...]`: `[[:alpha:]]`,
  `[[:digit:]]`, `[[:space:]]`, `[[:upper:]]`, `[[:lower:]]`,
  `[[:alnum:]]`, `[[:blank:]]`, `[[:cntrl:]]`, `[[:graph:]]`,
  `[[:print:]]`, `[[:punct:]]`, `[[:xdigit:]]`. Implementation
  shared with v0.9.0 catch-up for bre/pcre/re2.
- `\zs` resets match-start (overrides implicit start-save). `\ze`
  freezes match-end (parser tracks `_vim_saw_ze` to suppress
  implicit final SAVE 1 so `\ze` wins).
- `\1`-`\9` backreferences **rejected at compile** with
  `VIM_E_BACKREF_UNSUPPORTED`. Decision flagged for v0.9.0 review
  alongside the bre backref question.
- ADR 0006 — niyama_vim engine ABI, matcher architecture, and
  scope. Records the Pike-NFA-not-backtracking decision, the
  four-mode opts surface, the `\zs`/`\ze` SAVE-emission strategy,
  and the deferral list (mid-pattern mode switching, vim's vast
  `\X` escape menagerie, replacement-language helpers).
- `tests/vim.tcyr` — 88 unit tests across 13 groups: per-mode
  parser semantics (×4 modes), `\<`/`\>` boundaries, `\zs`/`\ze`
  match-position semantics, all 12 POSIX bracket classes,
  predefined classes, backref rejection, anchors, invalid mode
  rejection, lazy brace, DoS-resistance.
- `fuzz/vim.fcyr` — 219-assertion harness with mode-coverage sweep
  (every random pattern exercised across all 4 modes), rejection
  invariants (backref, bad mode), and a linear-time adversarial
  pattern.
- `tests/vim.bcyr` — bench harness across all four modes plus
  vim-feature benches.

### Performance floor (M4, x86_64, cyrius 5.8.42)

- `vim_compile_*` (per mode): 3-5 μs.
- `vim_search_magic` / `_very_magic` / `_nomagic` / `_very_nomagic`
  (3-way alt over 75-byte text): 7-8 μs across all modes.
- `vim_search_zs_ze` (`foo\zsbar\zebaz`): ~3 μs.
- `vim_search_posix` (`[[:alpha:]]\+`): ~3 μs.
- `vim_search_word_bound` (`\<word\>`): ~3 μs.

### Changed

- `dist/niyama.cyr` — bundle now includes `src/vim.cyr` alongside
  the four prior engines. `NIYAMA_VERSION` → `"0.6.0"`.
- `src/main.cyr` smoke banner reflects M4 status.

### Deferred (not in M4 — see ADR 0006)

- Mid-pattern mode switching (`\v` / `\m` / `\M` / `\V` mid-pattern).
  Opts-flag-only entry in M4; mid-pattern switching is a v0.9.0
  candidate if asked.
- vim's vast `\X` escape menagerie (`\a`, `\A`, `\l`, `\L`, `\u`,
  `\U`, `\x`, `\X`, `\o`, `\O`, `\h`, `\H`, etc.) — niyama_vim
  ships standard `\d`/`\w`/`\s`; vim-specific extras are post-v1.0
  unless asked.
- Replacement-language helpers (`~`, `&`, `\u`/`\l`/`\U`/`\L`).
  Replacement is a consumer concern; cyim handles its own.

### Roadmap reorg (2026-05-03)

Consolidated 11 features that prior ADRs (0003 / 0004 / 0005) had
deferred unilaterally as "post-v1.0" or "M3.5 candidate" into a new
v0.9.0 (M4.5) catch-up milestone. The rationale: deferrals that
silently shrink what ships in v1.0 are scope decisions belonging to
the user, not to the engine implementer. The v0.9.0 catch-up release
clears those deferrals before the M5 freeze, so the surface that
gets frozen is the surface the roadmap originally promised.

Items consolidated:

- **niyama_re2**: named captures, Unicode property classes `\p{L}`,
  inline flags `(?i)/(?m)/(?s)`.
- **niyama_pcre**: lookbehind, `\p{L}`, POSIX bracket classes,
  recursion, conditional patterns, inline flags, branch-reset
  groups, callouts, `\K`.
- **niyama_bre**: GNU `\<`/`\>` word boundaries, POSIX bracket
  classes (shared implementations with pcre/vim).
- **niyama_fuzzy**: Unicode NFD normalization, exact start-position
  recovery in `_search`.

`niyama_bre` `\1`-`\9` backreferences remain per the user's M1
explicit "potentially post-v1.0; document, don't skip" call —
v0.9.0 is a natural revisit point but not a commitment.

ADRs 0002 / 0003 / 0004 / 0005 updated to point their deferral
sections at v0.9.0; `docs/development/roadmap.md` adds the M4.5
milestone; `docs/development/state.md` updates the "Next" sequencing.
No code changes; no shipped engines affected.

## [0.5.0] — 2026-05-03

M3.5 — fourth engine: fuzzy (`niyama_fuzzy_*`). The one engine in
niyama that isn't regex — Levenshtein edit-distance matching for
shell completion, fuzzy-name lookup, and typo-tolerant command
matching. Per ADR 0005.

### Added

- `src/fuzzy.cyr` — Wagner–Fischer Levenshtein DP with two-row
  optimization. ~300 lines (smallest engine in niyama). Three
  match modes via three named functions.
- Public ABI mirroring niyama_bre / niyama_re2 / niyama_pcre per
  ADR 0002, plus fuzzy-specific options:
  - `niyama_fuzzy_compile(pat)` — default options (max_edits=2).
  - `niyama_fuzzy_compile_opts(pat, max_edits, flags)` — full opts.
  - `niyama_fuzzy_match(h, s)` — anchored full-string fuzzy match.
  - `niyama_fuzzy_search(h, s)` — substring-fuzzy: best contiguous
    slice of `s` within edit distance.
  - `niyama_fuzzy_search_prefix(h, s)` — prefix-fuzzy: pattern is
    a typo-tolerant prefix of `s`. The shell-completion shape.
  - `niyama_fuzzy_distance(h, s)` — full-string distance.
  - `niyama_fuzzy_last_distance()` — distance from the last match
    call. Useful for "match found AND it cost N typos".
  - `niyama_fuzzy_last_error()` — error code from last compile.
- `FUZZY_FLAG_CASE_INSENSITIVE` (= 1) — ASCII case-fold flag.
- ADR 0005 — niyama_fuzzy ABI shape and scope. Records the
  algorithm choice (DP over bitap), the three-mode API, and the
  Unicode-NFD deferral.
- `tests/fuzzy.tcyr` — 45 unit tests across 7 groups: distance
  correctness against known Levenshtein values (kitten/sitting,
  Saturday/Sunday), all three match modes, threshold edges,
  case-insensitive flag, last_distance / last_error
  observability, real-world command-completion sketches.
- `fuzz/fuzzy.fcyr` — 757-assertion harness verifying all five
  Levenshtein **mathematical invariants** on randomized inputs:
  identity (`d(s,s)=0`), symmetry (`d(a,b)=d(b,a)`),
  non-negativity, length bound (`d(a,b) ≤ max(|a|,|b|)`), triangle
  inequality (`d(a,c) ≤ d(a,b)+d(b,c)`). Plus the no-crash sweep
  and compile-error invariants.
- `tests/fuzzy.bcyr` — bench harness covering compile, distance,
  match, search (substring + prefix), case-insensitive, and a
  larger-pattern stress.

### Performance floor (M3.5, x86_64, cyrius 5.8.42)

- `fuzzy_compile_*` — ~1 μs.
- `fuzzy_distance` (6-byte pattern, 5-byte text): ~1 μs.
- `fuzzy_distance` (6-byte pattern, 30-byte text): ~3 μs.
- `fuzzy_match`: ~1 μs.
- `fuzzy_search_short` (6-byte pattern, 26-byte text): ~3 μs.
- `fuzzy_search_long` (6-byte pattern, 256-byte text): ~17 μs.
- `fuzzy_search_prefix`: ~2 μs.
- `fuzzy_case_insensitive`: ~1 μs.
- `fuzzy_medium_pattern_distance` (25-byte pattern, 46-byte text):
  ~15 μs.

### Changed

- `dist/niyama.cyr` — bundle now includes `src/fuzzy.cyr` alongside
  `src/bre.cyr`, `src/re2.cyr`, and `src/pcre.cyr`.
  `NIYAMA_VERSION` → `"0.5.0"`.
- `src/main.cyr` smoke banner reflects M3.5 status.

### Deferred (not in M3.5 — see ADR 0005)

- Unicode NFD normalization (`FUZZY_FLAG_UNICODE_NFD`) — needs
  ~25KB Unicode decomposition table; ASCII-heavy AGNOS consumers
  don't benefit yet. Post-v1.0.
- Exact start-position recovery in `_search` — currently returns
  `end - len(pat)` clamped. Heuristic covers ≥90% of consumer
  cases; M5 may revisit if asked.

## [0.4.0] — 2026-05-03

M3 — third engine: pcre (`niyama_pcre_*`). Backtracking matcher (the
first non-Pike-NFA engine in niyama) bringing the features re2
deliberately rejects: backreferences, lookahead, atomic groups, named
captures, possessive quantifiers. Catastrophic-backtracking risk is
mitigated by an explicit step-limit guard with a configurable budget
(default 1M steps) and a hard recursion-depth bound (256). Per ADR
0004.

### Added

- `src/pcre.cyr` — backtracking matcher with PCRE-flavor parser,
  ~1100 lines. New opcodes: `PCRE_OP_BACKREF`, `PCRE_OP_LOOKAHEAD`,
  `PCRE_OP_NLOOKAHEAD`, `PCRE_OP_LOOKAHEAD_END`, `PCRE_OP_ATOMIC`,
  `PCRE_OP_ATOMIC_END`. Same instruction layout as bre/re2 for
  shared opcodes.
- Public ABI mirroring niyama_bre / niyama_re2 plus pcre-specific
  extensions:
  - `niyama_pcre_compile` / `_match` / `_search` / `_search_at`
  - `niyama_pcre_group_start` / `_group_end` (groups 0..9)
  - `niyama_pcre_group_by_name(nfa, name)` — named-capture lookup
  - `niyama_pcre_last_error()` — frozen error code set
  - `niyama_pcre_set_step_limit(n)` — configurable step budget
  - `niyama_pcre_last_step_count()` — observability hook
- **PCRE feature set in M3**: ERE base (literals, `.`, anchors,
  `\d`/`\w`/`\s`/`\b`, classes, `*` `+` `?` `{n,m}`, lazy,
  alternation, capturing + non-capturing), plus:
  - **Backreferences `\1`-`\9`** — the headline PCRE feature.
  - **Lookahead `(?=...)` and `(?!...)`** — variable-width.
  - **Atomic groups `(?>...)`** — no internal backtracking.
  - **Possessive quantifiers `*+`, `++`, `?+`, `{n,m}+`** —
    desugared to atomic-wrapping at compile.
  - **Named captures `(?<name>...)` and `(?P<name>...)`** — both
    syntaxes; lookup by name. Up to 9 named (shared with positional).
  - Step-limit + depth-limit catastrophic-backtracking guard.
- ADR 0004 — niyama_pcre engine ABI shape, matcher architecture,
  and scope. Records the backtracking-vs-Pike-NFA decision, the
  step-limit semantics, and the deferral list.
- `tests/pcre.tcyr` — 83 unit tests across 13 groups, including
  backref correctness, lookahead semantics, atomic-blocks-backtracking
  demo, named-capture lookup, all 7 deferred-feature rejection codes,
  and the step-limit guard kicking in on `(a+)+b` against 25 'a's.
- `fuzz/pcre.fcyr` — 229-assertion harness with adversarial pattern
  generator, every rejection invariant, and 4 catastrophic-backtracking
  patterns under tight step limit.
- `tests/pcre.bcyr` — bench harness covering compile + search +
  PCRE-specific features + the bounded-DoS bench (`(a+)+b` on 30
  'a's with `step_limit=50k` terminates in ~2.4ms).

### Performance floor (M3, x86_64, cyrius 5.8.42)

- `pcre_compile_*` — 4-5 μs avg (literal, email-class, backref,
  lookahead).
- `pcre_search_literal` (256-byte input): ~17 μs.
- `pcre_search_email`: ~9 μs.
- `pcre_search_alt` (3-way alt): ~61 μs.
- `pcre_backref` (`(\w+) \1` on `hello hello world`): ~2 μs.
- `pcre_lookahead` (`\w+(?=@)` on 256-byte input): ~180 μs (largest
  search bench — lookahead validates against multiple match start
  positions).
- `pcre_atomic` (`(?>a*)b` on `aaaaaaaaaaaaab`): ~3 μs.
- `pcre_named_captures`: ~3 μs.
- **Catastrophic-backtracking guard**: `(a+)+b` against 30 'a's with
  no terminator, `step_limit=50000` — ~2.4 ms. Without the limit
  this pattern would explore ~2^30 paths (would never terminate in
  practice).

### Changed

- `dist/niyama.cyr` — bundle now includes `src/pcre.cyr` alongside
  `src/bre.cyr` and `src/re2.cyr`. `NIYAMA_VERSION` → `"0.4.0"`.
- `src/main.cyr` smoke banner reflects M3 status.

### Deferred (not in M3 — see ADR 0004 for rationale)

- Lookbehind `(?<=...)` `(?<!...)` — needs fixed-width analysis;
  M3.5 candidate if a consumer asks. Compile rejects with
  `PCRE_E_LOOKBEHIND_UNSUPPORTED`.
- Unicode property classes `\p{L}` — needs Unicode database. Rejects
  with `PCRE_E_UNICODE_PROP_UNSUPPORTED`.
- POSIX bracket classes `[:alpha:]` — deferred to M4 (vim flavor
  inherits the same semantics).
- Recursion `(?R)` `(?P>name)` — `PCRE_E_RECURSION_UNSUPPORTED`.
- Conditional patterns `(?(...)...)` — `PCRE_E_CONDITIONAL_UNSUPPORTED`.
- Inline flags `(?i)` `(?m)` `(?s)` — generic `PCRE_E_SYNTAX`.

## [0.3.0] — 2026-05-03

M2 — second engine: re2 (`niyama_re2_*`). Linear-time Pike NFA matcher
with ERE-flavor parser and **explicit compile-time rejection of every
feature that would break the linear-time guarantee**. Each
non-regular construct gets its own error code so consumers know
exactly which engine to fall back to (niyama_pcre at M3). Per ADR
0003.

### Added

- `src/re2.cyr` — Pike NFA matcher with ERE syntax: `\d` `\D` `\w`
  `\W` `\s` `\S` `\b` `\B`, alternation `|`, `(...)` capturing
  groups, `(?:...)` non-capturing, greedy AND lazy quantifiers
  (`*` `+` `?` `{n,m}` and lazy variants). Same Pike NFA matcher
  kernel as niyama_bre — the linear-time guarantee is structural.
- Public ABI: `niyama_re2_compile`, `niyama_re2_match`,
  `niyama_re2_search`, `niyama_re2_search_at`,
  `niyama_re2_group_start`, `niyama_re2_group_end`,
  `niyama_re2_last_error`. Mirrors `niyama_bre_*` shape per ADR
  0002.
- **Compile-time rejection** of features that would break linear
  time, each with its own error code:
  - `RE2_E_BACKREF_UNSUPPORTED` (= 2) — `\1`-`\9` backreferences.
  - `RE2_E_LOOKAROUND_UNSUPPORTED` (= 3) — `(?=...)` `(?!...)`
    `(?<=...)` `(?<!...)` lookaround.
  - `RE2_E_ATOMIC_UNSUPPORTED` (= 4) — `(?>...)` atomic groups.
  - `RE2_E_RECURSION_UNSUPPORTED` (= 5) — `(?R)` `(?P>name)`
    recursion / subroutine calls.
  - `RE2_E_TOO_LARGE` (= 6) — pattern compile exceeds instruction
    or class limits.
- ADR 0003 — niyama_re2 engine ABI shape and scope. Records the
  ERE feature set, the linear-time guarantee, the rejection
  contract, and the deferred-to-post-M3 named-capture decision.
- `tests/re2.tcyr` — 76 unit tests across 11 feature groups,
  including each rejection error code AND adversarial linear-time
  patterns (`(a|a)*b`, `(a*)*b`, Cox's `a?{30}a{30}` adversary,
  deep alternation × repetition).
- `fuzz/re2.fcyr` — 221-assertion harness, including the four
  rejection-invariant checks (every pattern matching `\1`/`(?=`/
  `(?>`/`(?R` MUST yield the corresponding error code).
- `tests/re2.bcyr` — bench harness with **dedicated DoS-resistance
  benches**: `(a|a)*b` and `(a*)*b` against 200-`a` inputs that
  would DoS a backtracking engine.

### Performance floor (M2, x86_64, cyrius 5.8.42)

- `re2_compile_*` — 3-5 μs avg (literal, alt, email-class).
- `re2_search_literal` (256-byte input): ~44 μs.
- `re2_search_alt` (3-way alt over 256-byte input): ~73 μs.
- `re2_search_email`: ~15 μs.
- **DoS-resistance** (the headline numbers — these would never
  terminate on a backtracking engine):
  - `(a|a)*b` against 200 `a`s, no match: ~84 μs.
  - `(a*)*b` against 200 `a`s, no match: ~60 μs.
  - Cox's `a?{30}a{30}` adversary, match: ~201 μs.

### Changed

- `dist/niyama.cyr` — bundle now includes `src/re2.cyr` alongside
  `src/bre.cyr`. `NIYAMA_VERSION` → `"0.3.0"`.
- `src/main.cyr` smoke banner reflects M2 status.

### Deferred (not in M2)

- Named captures `(?P<name>...)` / `(?<name>...)` — deferred to
  post-M3. Both re2 and pcre will share the named-capture API
  surface; designing it once with both engines in hand avoids
  shipping it twice.
- Unicode property classes `\p{L}` — M2 is ASCII-only.
- Inline flags `(?i)` `(?m)` `(?s)` — post-v1.0 unless asked.

## [0.2.0] — 2026-05-03

M1 — first engine shipped: POSIX BRE (`niyama_bre_*`). Per ADR 0002,
ABI mirrors stdlib `regex_*` shape; `\1`-`\9` backreferences are
rejected at compile time with `BRE_E_BACKREF_UNSUPPORTED` (deferred
to potentially post-v1.0 — explicit reject + document, never silent
skip).

### Added

- `src/bre.cyr` — POSIX BRE engine. Forks the cyrius stdlib
  `lib/regex.cyr` Pike NFA / Thompson construction (instruction
  model, class bitmap, matcher) with a BRE-flavor parser. All
  globals `_bre_*`-prefixed for collision-free coexistence with
  `lib/regex.cyr` in the same program.
- Public ABI: `niyama_bre_compile`, `niyama_bre_match`,
  `niyama_bre_search`, `niyama_bre_search_at`,
  `niyama_bre_group_start`, `niyama_bre_group_end`,
  `niyama_bre_last_error`. Error codes frozen at M1: `BRE_E_OK` (0),
  `BRE_E_SYNTAX` (1), `BRE_E_BACKREF_UNSUPPORTED` (2),
  `BRE_E_TOO_LARGE` (3), `BRE_E_BAD_ANCHOR` (4 — reserved).
- POSIX BRE features: literals, `.`, `*`, `\(...\)` capturing groups
  (1..9), `\{n,m\}` / `\{n,\}` / `\{n\}` quantifiers, `^` start
  anchor (only at byte 0), `$` end anchor (only at end), `[...]` /
  `[^...]` bracket expressions with ranges, common backslash
  escapes (`\.` `\*` `\\` `\n` `\t` etc.).
- POSIX-faithful literal-by-default for `+`, `?`, `(`, `)`, `{`,
  `}`, `|`. `^`/`$` are literal in mid-pattern positions.
- ADR 0002 — niyama_bre engine ABI shape and scope. Records ABI
  surface, error-code numbering (frozen ABI at M1), backref
  rejection policy + post-v1.0 reconsideration path.
- `tests/bre.tcyr` — 68 unit tests covering literals, dot, star,
  POSIX literal-by-default for `+`/`?`, anchor placement, bracket
  expressions, groups + brace quantifiers, escapes, backref
  rejection, syntax error paths, search semantics, and a
  catastrophic-backtracking-class adversarial pattern (asserts
  linear-time DoS-resistance).
- `fuzz/bre.fcyr` — randomized fuzz harness, 200-iter sweep over
  metachar-heavy random patterns; smoke corpus drawn from past
  stdlib regex bug-fix history (3-way alt regression at v5.7.18,
  body-pc re-entry regression).
- `tests/bre.bcyr` — bench harness across compile + search paths.

### Performance floor (M1, x86_64, cyrius 5.8.42)

- `bre_compile_*` — ~3 μs avg (literal, dot-star, quantifier, group).
- `bre_search_literal_hit` (256-byte input, 6-byte needle): ~44 μs.
- `bre_search_literal_miss` (22-byte input): ~9 μs.
- `bre_search_dot_star` (256-byte input, `.*needle.*`): ~112 μs.
- `bre_search_anchored` (`^needle`): ~2 μs (anchor short-circuits).
- `bre_search_group` (256-byte input, two captures): ~16 μs.

Floor recorded for M5 hardening regression detection.

### Changed

- Cyrius pin bumped: `5.7.24` → `5.8.42` (`cyrius.cyml [package].cyrius`).
- `dist/niyama.cyr` — placeholder replaced with the M1 bundle
  (single-include over `src/bre.cyr`). `NIYAMA_VERSION` now `"0.2.0"`.

### Deferred (not in M1)

- `\1`-`\9` backreferences — rejected at compile per ADR 0002.
  Patterns needing backrefs should use `niyama_pcre` (M3); BRE
  backref support is potentially post-v1.0 work, gated on a real
  consumer ask.
- GNU `\<` / `\>` word boundaries — deferred to M4 (vim flavor
  inherits the same semantics).
- `[:alpha:]` POSIX bracket character classes — deferred to M4.

## [0.1.0] — 2026-04-28

Initial scaffold. Repo positioning, doc-tree, ADR 0001 (sandhi-pattern
positioning + fold lifecycle), roadmap to v1.0. No engines shipped
yet — engine work begins at M1 (POSIX BRE).

### Added

- Project scaffold via `cyrius init niyama` (first-party-standards
  conformance: VERSION, cyrius.cyml, CLAUDE.md, CHANGELOG.md,
  README.md, LICENSE, CI workflows, doc-tree).
- README.md positioning: additional-regex-engines repo for the
  AGNOS-lineage Cyrius ecosystem, sandhi-pattern lifecycle, M0–v1.0
  engine roadmap, planned consumers (cyim, owl, agnoshi, daimon).
- ADR 0001 — niyama as the additional-engines repo following the
  sandhi-pattern fold lifecycle. Records:
  - Why one foundational engine in cyrius stdlib + additional engines
    in niyama (rather than per-engine repos or stdlib expansion).
  - The fold-back gate (≥2 long-horizon consumers + 1.0.0 + frozen
    surface + explicit fold ADR per sandhi ADR 0002 template).
  - Speculative cyrius 5.8.0 fold target (CONDITIONAL on consumer
    count — not a deadline).
- `docs/development/roadmap.md` — M0 (scaffold, done), M1 (bre — first
  engine, picked to shake out dispatch surface), M2 (re2, DoS-safe),
  M3 (pcre, largest fuzz target), M3.5 (fuzzy, Levenshtein), M4 (vim
  flavor), M5 (hardening + freeze), v1.0 (fold-ready).
- `docs/development/state.md` — initial scaffold snapshot.
- `dist/niyama.cyr` — placeholder for the fold-ready single-file
  artifact (sandhi precedent: `dist/sandhi.cyr` is what stdlib
  vendored byte-identical at the fold).
