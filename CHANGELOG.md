# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

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
