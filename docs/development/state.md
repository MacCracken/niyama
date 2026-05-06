# niyama — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.9.0** — M5 P(-1) hardening + surface freeze release shipped
2026-05-05. **Last release before v1.0 fold-ready.** Per ADR 0010,
the public `niyama_<engine>_*` API surface is locked through v1.0;
post-fold extensions (per ADR 0009 + the v0.9.0 review remainders)
land via cyrius stdlib once the fold gate is met. The 9-step P(-1)
pass ran clean: no CRITICAL/HIGH security findings, no bench
regressions (all 47 measurements within ±2.5% of v0.8.0 baseline).

**v0.8.0** — M4.5 completion release shipped 2026-05-05. Per ADR 0008
(Unicode-stdlib pivot + sub-release collapse), every remaining M4.5
deferral landed: `\p{L}` for re2/pcre/vim, multi-byte literal
pattern chars, `(?i)` Unicode case-fold upgrade, fuzzy NFD via
stdlib `str_normalize`, fuzzy exact-start recovery, vim →
posix_classes refactor, pcre lookbehind (fixed-width), pcre
recursion `(?R)` / `(?P>NAME)`. v0.8.1 collapsed (no release);
backref question moved to v0.9.0 review (resolved as ADR 0009).

**v0.8.x ladder rule.** Any feature deferred *out of v0.8.0 during
implementation* gets a pinned v0.8.x slot in `roadmap.md`, not a
floating "post-v1.0 / vN.0 maybe" note. Pins don't force releases —
slots that don't warrant a tag collapse to a CHANGELOG note.
Established post-v0.7.0; prevented silent roadmap shrinkage and
release-tag dust through the v0.8.0 → v0.9.0 → v1.0 sequence.

## Toolchain

- **Cyrius pin**: `5.8.65` (in `cyrius.cyml [package].cyrius`).
  Bumped from `5.8.42` post-v0.7.0 to pull in the stdlib
  `lib/unicode/` tree (categories at .49, casefold at .50, normalize
  at .51, codec lift at .55, NFKC/NFKD at .60). Required floor for
  v0.8.0 onward per ADR 0008.

## Source

- `src/main.cyr` — smoke entry (prints identity banner).
- `src/test.cyr` — top-level test entry per `[build].test`.
- **`src/posix_classes.cyr`** — shared POSIX bracket-class fillers
  + name recognizer (v0.7.0). vim folded onto this module at v0.8.0.
- **`src/unicode_props.cyr`** — shared `\p{NAME}` / `\P{NAME}`
  parser + GeneralCategory bitmask lookup (v0.8.0). Used by
  re2/pcre/vim. Backed by stdlib `lib/unicode/categories.cyr`.
  ~140 lines.
- **`src/bre.cyr`** — POSIX BRE engine (M1, +`\<\>` and POSIX
  classes from v0.7.0). Pike NFA.
- **`src/re2.cyr`** — RE2-flavor linear-time engine (M2, +named
  captures + inline flags from v0.7.0; +`\p{L}` + multi-byte
  literals + `(?i)` Unicode upgrade from v0.8.0). Pike NFA,
  codepoint-stepped matcher loop.
- **`src/pcre.cyr`** — PCRE-light backtracking engine (M3,
  +POSIX classes / inline flags / `\K` / branch-reset / conditional
  / callouts from v0.7.0; +`\p{L}` / `(?i)` Unicode / fixed-width
  lookbehind / recursion from v0.8.0).
- **`src/fuzzy.cyr`** — Levenshtein edit-distance engine (M3.5,
  +Unicode NFD + exact-start recovery from v0.8.0).
- **`src/vim.cyr`** — vim/cyim flavor engine (M4, +`\p{L}` +
  multi-byte literals + posix_classes refactor from v0.8.0). Pike
  NFA, codepoint-stepped matcher loop.

## Engines shipped

| Engine | Status | ABI prefix | Algorithm | Notes |
|---|---|---|---|---|
| **bre** | ✅ v0.7.0 (M1 + v0.7.0 catch-up) | `niyama_bre_*` | Pike NFA | POSIX BRE minus backrefs (per ADR 0002). v0.7.0 adds GNU `\<\>` boundaries + POSIX bracket classes (ADR 0007). |
| **re2** | ✅ v0.8.0 (M2 + M4.5 complete) | `niyama_re2_*` | Pike NFA, codepoint-stepped | ERE + linear-time guarantee at API (per ADR 0003). v0.7.0: named captures, inline flags, strict `^/$/.` defaults. v0.8.0: `\p{L}` Unicode props, multi-byte literal patterns, `(?i)` Unicode case-fold (ADR 0008). |
| **pcre** | ✅ v0.8.0 (M3 + M4.5 complete) | `niyama_pcre_*` | Backtracking | Perl-compat (per ADR 0004). Step-limit + depth bound. v0.7.0: POSIX classes, inline flags, `\K`, branch-reset, conditional, callouts. v0.8.0: `\p{L}`, `(?i)` Unicode, fixed-width lookbehind, recursion `(?R)` / `(?P>NAME)` (ADR 0008). |
| **fuzzy** | ✅ v0.8.0 (M3.5 + M4.5 complete) | `niyama_fuzzy_*` | Levenshtein DP | Edit-distance, three match modes (per ADR 0005). v0.8.0: `FUZZY_FLAG_UNICODE_NFD`, exact start-position recovery via reverse-DP. |
| **vim** | ✅ v0.8.0 (M4 + M4.5 complete) | `niyama_vim_*` | Pike NFA, codepoint-stepped | vim/cyim flavor, 4 magicness modes (per ADR 0006). v0.8.0: `\p{L}` Unicode props, multi-byte literal patterns, POSIX-class code folded onto `src/posix_classes.cyr`. |

## Engine-selection guidance for consumers

| Want | Use |
|---|---|
| `grep -G` / `sed` POSIX BRE compatibility (no backref) | `bre` |
| ERE + provable linear-time DoS-safety on untrusted input | `re2` |
| **Backref `\1`-`\9`** in any flavor | `pcre` (per ADR 0009 — bre/vim do not implement backref through v1.0; vim *may* gain it post-fold via cyrius stdlib) |
| Lookahead, lookbehind, named captures, atomic groups, recursion | `pcre` |
| Both ERE features AND DoS-safety | `re2` (no backref/lookaround) |
| Both PCRE features AND bounded DoS-safety | `pcre` with `niyama_pcre_set_step_limit()` |
| Typo-tolerant matching, shell completion, fuzzy-name lookup | `fuzzy` |
| vim/cyim-flavor patterns (`\<`, `\>`, `\zs`/`\ze`, magicness) | `vim` |
| Unicode property classes `\p{L}` etc. | `re2`, `pcre`, or `vim` (v0.8.0+) |
| Case-insensitive / multi-line / dot-newline matching | `re2` or `pcre` with `(?i)`/`(?m)`/`(?s)` |
| Conditional or branch-reset patterns | `pcre` (`(?(...)...)`, `(?\|...)`) |
| Linear-time *family* (DoS-safe by construction) | `re2`, `bre`, `vim` — backref structurally rejected |
| Backtracking *family* (full PCRE feature set, step-limit guarded) | `pcre` |

## Fold-ready artifact

- `dist/niyama.cyr` — single-include bundle. v0.8.0 prepends both
  shared modules (`src/posix_classes.cyr` then `src/unicode_props.cyr`)
  ahead of the five engine modules. Consumers also need stdlib
  `lib/str.cyr` and `lib/unicode/{categories,casefold,normalize}.cyr`.
  M5 surface freeze ADR will lock its public symbol set.

## Tests

- `tests/niyama.tcyr` — scaffold smoke (2 assertions).
- **`tests/bre.tcyr`** — 109 BRE assertions (v0.9.0: +2 boundary).
- **`tests/re2.tcyr`** — 169 RE2 assertions (v0.9.0: +18 POSIX
  classes per G2 fix, +4 boundary).
- **`tests/pcre.tcyr`** — 188 PCRE assertions (v0.9.0: +3 boundary).
- **`tests/fuzzy.tcyr`** — 53 fuzzy assertions.
- **`tests/vim.tcyr`** — 106 vim assertions (v0.9.0: +2 boundary).
- **`tests/{bre,re2,pcre,fuzzy,vim}.bcyr`** — per-engine bench harnesses.
- **`fuzz/{bre,re2,pcre,fuzzy,vim}.fcyr`** — per-engine fuzz harnesses
  (v0.9.0: invalid-UTF-8 input seeds added to re2/pcre/vim).

Aggregate: `cyrius test` reports **6 files, 627 assertions** all passing
(was 598; +29 v0.9.0).
`cyrius fuzz` reports **5 files, 1689 assertions** all passing
(was 1660; +29 v0.9.0).

Bench history captured in [`../benchmarks.md`](../benchmarks.md).
Security audit history in [`../audit/`](../audit/).
Architecture invariants in [`../architecture/`](../architecture/).

## Dependencies

Direct (declared in `cyrius.cyml`):

- stdlib — string, fmt, alloc, io, vec, str, syscalls, assert, unicode.
  (`unicode` added post-v0.7.0 alongside the toolchain bump per ADR
  0008 — pulls in the `lib/unicode/{categories,casefold,normalize,
  _decode}.cyr` tree.)

## Consumers

| Consumer | Status | Notes |
|----------|--------|-------|
| [cyim](https://github.com/MacCracken/cyim) | Parser-side ready (1.2.0) | All five flavors (`bre`, `re2`, `pcre`, `fuzzy`, `vim`) ready to wire. cyim ADR 0002 keeps consumer code change at zero. |
| owl | Planned | Pager / cat-class utility. |
| agnoshi | Planned | AI shell — `fuzzy` ready for shell completion. |
| daimon | Planned | Agent orchestration — `re2` ready for DoS-safe pattern gates; `pcre` with step-limit for richer patterns; `fuzzy` for fuzzy-name match. |

## Next

1. **v1.0 — fold-ready release.** `dist/niyama.cyr` is the
   byte-identical fold candidate; the public surface is frozen
   per ADR 0010. Remaining v1.0 work is the **remainders sweep**:
   - LOW review findings (A1 fuzzy `_search_at` doc, C2 (?i)ß
     semantics-lock test, C4 recursion+lookaround nesting tests,
     C5 empty-pattern tests, F2 `docs/api/`).
   - CLAUDE.md completeness — Cyrius Conventions, CI/Release,
     Documentation Structure, `.gitignore`, CHANGELOG Format
     sections from agnosticos `example_claude.md`.
   - Fold ADR (template: sandhi ADR 0002).
   - Final closeout (test/fuzz re-run, bench diff vs v0.9.0,
     downstream check against cyim).
2. **Post-fold (post-v1.0)** — Cyrius stdlib vendors
   `lib/niyama.cyr` per the sandhi v5.7.0 lifecycle, conditional
   on fold gate met (≥2 long-horizon consumers; cyim is #1).
   Stdlib-side extensions per ADR 0010's post-freeze evolution
   model: vim backref (per ADR 0009), `_search_at` for fuzzy,
   long property names + Unicode scripts, etc. — additive only.
3. niyama-the-repo enters maintenance mode at v1.0; future
   v1.x patches are bug-fix only.
3. **M5 (post-v0.9.0)** — P(-1) hardening + closeout + surface freeze.
4. **v1.0** — fold-ready release.

See [`roadmap.md`](roadmap.md) for the full plan.
