# niyama — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.5.0** — M3.5 shipped 2026-05-03. niyama_fuzzy engine live:
Levenshtein edit-distance matching with anchored / substring / prefix
modes per ADR 0005. niyama now ships **four engines**: three regex
flavors (bre, re2, pcre) plus fuzzy. One milestone remains before
v1.0: M4 (vim flavor) → M5 (hardening + freeze) → fold-ready 1.0.0.

## Toolchain

- **Cyrius pin**: `5.8.42` (in `cyrius.cyml [package].cyrius`)

## Source

- `src/main.cyr` — smoke entry (prints identity banner).
- `src/test.cyr` — top-level test entry per `[build].test`.
- **`src/bre.cyr`** — POSIX BRE engine (M1). ~600 lines, Pike NFA.
- **`src/re2.cyr`** — RE2-flavor linear-time engine (M2). ~700 lines, Pike NFA.
- **`src/pcre.cyr`** — PCRE-light backtracking engine (M3). ~1100 lines.
- **`src/fuzzy.cyr`** — Levenshtein edit-distance engine (M3.5). ~300 lines.
- Future: `src/vim.cyr` (M4).

## Engines shipped

| Engine | Status | ABI prefix | Algorithm | Notes |
|---|---|---|---|---|
| **bre** | ✅ v0.2.0 (M1) | `niyama_bre_*` | Pike NFA | POSIX BRE minus backrefs (per ADR 0002). |
| **re2** | ✅ v0.3.0 (M2) | `niyama_re2_*` | Pike NFA | ERE + linear-time guarantee at API (per ADR 0003). |
| **pcre** | ✅ v0.4.0 (M3) | `niyama_pcre_*` | Backtracking | Perl-compat (per ADR 0004). Step-limit + depth bound. |
| **fuzzy** | ✅ v0.5.0 (M3.5) | `niyama_fuzzy_*` | Levenshtein DP | Edit-distance, three match modes (per ADR 0005). |
| vim | planned (M4) | `niyama_vim_*` | TBD | magic / nomagic / very-magic / very-nomagic. |

## Engine-selection guidance for consumers

| Want | Use |
|---|---|
| `grep -G` / `sed` POSIX BRE compatibility | `bre` |
| ERE + provable linear-time DoS-safety on untrusted input | `re2` |
| Backref, lookahead, named captures, atomic groups | `pcre` |
| Both ERE features AND DoS-safety | `re2` (no backref/lookaround) |
| Both PCRE features AND bounded DoS-safety | `pcre` with `niyama_pcre_set_step_limit()` |
| Typo-tolerant matching, shell completion, fuzzy-name lookup | `fuzzy` |

## Fold-ready artifact

- `dist/niyama.cyr` — single-include bundle. v0.5.0 bundles
  `src/bre.cyr` + `src/re2.cyr` + `src/pcre.cyr` + `src/fuzzy.cyr`.
  M5 surface freeze ADR will lock its public symbol set.

## Tests

- `tests/niyama.tcyr` — scaffold smoke (2 assertions).
- **`tests/bre.tcyr`** — 68 BRE assertions.
- **`tests/re2.tcyr`** — 76 RE2 assertions, 4 adversarial linear-time tests.
- **`tests/pcre.tcyr`** — 83 PCRE assertions, 7 deferred-feature codes,
  step-limit-guard verification.
- **`tests/fuzzy.tcyr`** — 45 fuzzy assertions (distance correctness,
  3 match modes, case-fold, observability).
- **`tests/bre.bcyr`** — BRE bench (4 compile + 7 search benches).
- **`tests/re2.bcyr`** — RE2 bench (3 compile + 4 search + 3 DoS-resistance benches).
- **`tests/pcre.bcyr`** — PCRE bench (4 compile + 3 search + 5 PCRE-feature + 1 bounded-DoS).
- **`tests/fuzzy.bcyr`** — fuzzy bench (10 cases incl. case-fold, prefix,
  medium pattern).
- **`fuzz/bre.fcyr`** — BRE fuzz (210 assertions).
- **`fuzz/re2.fcyr`** — RE2 fuzz (221 assertions).
- **`fuzz/pcre.fcyr`** — PCRE fuzz (229 assertions).
- **`fuzz/fuzzy.fcyr`** — fuzzy fuzz (757 assertions, 5 mathematical
  invariants verified on randomized inputs).

Aggregate: `cyrius test` reports 5 files, 274 assertions all passing.
`cyrius fuzz` reports 4 files, 1417 assertions all passing.

## Dependencies

Direct (declared in `cyrius.cyml`):

- stdlib — string, fmt, alloc, io, vec, str, syscalls, assert
  (auto-included for `[build].test`; per-test/bench files include
  the explicit deps they need).

## Consumers

| Consumer | Status | Notes |
|----------|--------|-------|
| [cyim](https://github.com/MacCracken/cyim) | Parser-side ready (1.2.0) | All four flavors (`bre`, `re2`, `pcre`, `fuzzy`) ready to wire. cyim ADR 0002 keeps consumer code change at zero. |
| owl | Planned | Pager / cat-class utility. |
| agnoshi | Planned | AI shell — **`fuzzy` is now ready** for shell completion (`_search_prefix`) and command lookup. The headline use case for niyama_fuzzy. |
| daimon | Planned | Agent orchestration — **`re2` ready** for DoS-safe pattern gates. **`pcre` with step-limit** for richer patterns. **`fuzzy` for fuzzy-name match.** |

## Next

M4 — `niyama_vim` engine (vim/cyim flavor: magic / nomagic / very-magic /
very-nomagic, `\<`/`\>` word boundaries, `\zs`/`\ze` match-start/end,
POSIX bracket classes `[:alpha:]`). cyim's `:s/old/new/` and ex-mode
pattern history are the consumer. After M4, M5 (hardening + freeze)
gates v1.0.0. See [`roadmap.md`](roadmap.md).
