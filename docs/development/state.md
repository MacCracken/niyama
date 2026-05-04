# niyama — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.4.0** — M3 shipped 2026-05-03. niyama_pcre engine live: backtracking
matcher with PCRE-light features (backref, lookahead, atomic, named
captures, possessive quantifiers), per ADR 0004. Catastrophic-
backtracking risk mitigated by configurable step-limit (default 1M
steps) + hard recursion-depth bound (256). niyama now ships three
engines covering the regex flavor space: `bre` (POSIX strict, no
backref), `re2` (linear-time-safe, no backref/lookaround/atomic),
`pcre` (full backtracking with bounded mitigation).

## Toolchain

- **Cyrius pin**: `5.8.42` (in `cyrius.cyml [package].cyrius`)

## Source

- `src/main.cyr` — smoke entry (prints identity banner).
- `src/test.cyr` — top-level test entry per `[build].test`.
- **`src/bre.cyr`** — POSIX BRE engine (M1). ~600 lines, Pike NFA.
- **`src/re2.cyr`** — RE2-flavor linear-time engine (M2). ~700 lines, Pike NFA.
- **`src/pcre.cyr`** — PCRE-light backtracking engine (M3). ~1100 lines.
- Future: `src/fuzzy.cyr` (M3.5), `src/vim.cyr` (M4).

## Engines shipped

| Engine | Status | ABI prefix | Matcher | Notes |
|---|---|---|---|---|
| **bre** | ✅ v0.2.0 (M1) | `niyama_bre_*` | Pike NFA | POSIX BRE minus backrefs (per ADR 0002). 68 unit tests. |
| **re2** | ✅ v0.3.0 (M2) | `niyama_re2_*` | Pike NFA | ERE + linear-time guarantee at API (per ADR 0003). 76 unit tests. Backref/lookaround/atomic/recursion rejected at compile. |
| **pcre** | ✅ v0.4.0 (M3) | `niyama_pcre_*` | Backtracking | Perl-compat (per ADR 0004). 83 unit tests. Backref + lookahead + atomic + named captures + possessive. Step-limit + depth bound. |
| fuzzy | planned (M3.5) | `niyama_fuzzy_*` | TBD | Levenshtein / typo-tolerant. |
| vim | planned (M4) | `niyama_vim_*` | TBD | magic / nomagic / very-magic / very-nomagic. |

## Engine-selection guidance for consumers

| Want | Use |
|---|---|
| `grep -G` / `sed` POSIX BRE compatibility | `bre` |
| ERE + provable linear-time DoS-safety on untrusted input | `re2` |
| Backref, lookahead, named captures, atomic groups | `pcre` |
| Both ERE features AND DoS-safety | `re2` (no backref/lookaround) |
| Both PCRE features AND bounded DoS-safety | `pcre` with `niyama_pcre_set_step_limit()` |

## Fold-ready artifact

- `dist/niyama.cyr` — single-include bundle. v0.4.0 bundles
  `src/bre.cyr` + `src/re2.cyr` + `src/pcre.cyr`. M5 surface freeze
  ADR will lock its public symbol set.

## Tests

- `tests/niyama.tcyr` — scaffold smoke (2 assertions).
- **`tests/bre.tcyr`** — 68 BRE assertions across 13 groups.
- **`tests/re2.tcyr`** — 76 RE2 assertions, 4 adversarial linear-time tests.
- **`tests/pcre.tcyr`** — 83 PCRE assertions, all 7 deferred-feature
  rejection codes, step-limit-guard verification.
- **`tests/bre.bcyr`** — BRE bench (4 compile + 7 search benches).
- **`tests/re2.bcyr`** — RE2 bench (3 compile + 4 search + 3 DoS-resistance benches).
- **`tests/pcre.bcyr`** — PCRE bench (4 compile + 3 search + 5 PCRE-feature + 1 bounded-DoS).
- **`fuzz/bre.fcyr`** — BRE fuzz (210 assertions).
- **`fuzz/re2.fcyr`** — RE2 fuzz (221 assertions, 4 rejection invariants).
- **`fuzz/pcre.fcyr`** — PCRE fuzz (229 assertions, 5 rejection invariants
  + catastrophic-backtracking guard exercise + adversarial sweep).

Aggregate: `cyrius test` reports 4 files, 229 assertions all passing.
`cyrius fuzz` reports 3 files, 660 assertions all passing.

## Dependencies

Direct (declared in `cyrius.cyml`):

- stdlib — string, fmt, alloc, io, vec, str, syscalls, assert
  (auto-included for `[build].test`; per-test/bench files include
  the explicit deps they need).

## Consumers

| Consumer | Status | Notes |
|----------|--------|-------|
| [cyim](https://github.com/MacCracken/cyim) | Parser-side ready (1.2.0) | All three flavors (`bre`, `re2`, `pcre`) ready to wire. cyim ADR 0002 keeps consumer code change at zero — flavor parser already threaded. |
| owl | Planned | Pager / cat-class utility. |
| agnoshi | Planned | AI shell — `fuzzy` flavor (M3.5) is the immediate win. |
| daimon | Planned | Agent orchestration — **`re2` ready** for DoS-safe pattern gates. **`pcre` with step-limit** for richer patterns when consumers can tolerate bounded cost. |

## Next

M3.5 — `niyama_fuzzy` engine (Levenshtein / typo-tolerant matching).
agnoshi shell completion + daimon agent fuzzy-name-match are the
immediate consumers. See [`roadmap.md`](roadmap.md).
