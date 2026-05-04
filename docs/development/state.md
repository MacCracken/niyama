# niyama — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.6.0** — M4 shipped 2026-05-03. niyama_vim engine live: Pike NFA
matcher with vim/cyim flavor (all 4 magicness modes, `\<`/`\>` word
boundaries, `\zs`/`\ze` match-position markers, POSIX bracket classes)
per ADR 0006. **All five engines now shipped**; v0.9.0 (M4.5
catch-up) is next, then M5 hardening + freeze, then v1.0 fold-ready.

## Toolchain

- **Cyrius pin**: `5.8.42` (in `cyrius.cyml [package].cyrius`)

## Source

- `src/main.cyr` — smoke entry (prints identity banner).
- `src/test.cyr` — top-level test entry per `[build].test`.
- **`src/bre.cyr`** — POSIX BRE engine (M1). ~600 lines, Pike NFA.
- **`src/re2.cyr`** — RE2-flavor linear-time engine (M2). ~700 lines, Pike NFA.
- **`src/pcre.cyr`** — PCRE-light backtracking engine (M3). ~1100 lines.
- **`src/fuzzy.cyr`** — Levenshtein edit-distance engine (M3.5). ~300 lines.
- **`src/vim.cyr`** — vim/cyim flavor engine (M4). ~1100 lines, Pike NFA.

## Engines shipped

| Engine | Status | ABI prefix | Algorithm | Notes |
|---|---|---|---|---|
| **bre** | ✅ v0.2.0 (M1) | `niyama_bre_*` | Pike NFA | POSIX BRE minus backrefs (per ADR 0002). |
| **re2** | ✅ v0.3.0 (M2) | `niyama_re2_*` | Pike NFA | ERE + linear-time guarantee at API (per ADR 0003). |
| **pcre** | ✅ v0.4.0 (M3) | `niyama_pcre_*` | Backtracking | Perl-compat (per ADR 0004). Step-limit + depth bound. |
| **fuzzy** | ✅ v0.5.0 (M3.5) | `niyama_fuzzy_*` | Levenshtein DP | Edit-distance, three match modes (per ADR 0005). |
| **vim** | ✅ v0.6.0 (M4) | `niyama_vim_*` | Pike NFA | vim/cyim flavor, 4 magicness modes (per ADR 0006). |

## Engine-selection guidance for consumers

| Want | Use |
|---|---|
| `grep -G` / `sed` POSIX BRE compatibility | `bre` |
| ERE + provable linear-time DoS-safety on untrusted input | `re2` |
| Backref, lookahead, named captures, atomic groups | `pcre` |
| Both ERE features AND DoS-safety | `re2` (no backref/lookaround) |
| Both PCRE features AND bounded DoS-safety | `pcre` with `niyama_pcre_set_step_limit()` |
| Typo-tolerant matching, shell completion, fuzzy-name lookup | `fuzzy` |
| vim/cyim-flavor patterns (`\<`, `\>`, `\zs`/`\ze`, magicness) | `vim` |

## Fold-ready artifact

- `dist/niyama.cyr` — single-include bundle. v0.6.0 bundles all five
  engines. M5 surface freeze ADR will lock its public symbol set.

## Tests

- `tests/niyama.tcyr` — scaffold smoke (2 assertions).
- **`tests/bre.tcyr`** — 68 BRE assertions.
- **`tests/re2.tcyr`** — 76 RE2 assertions, 4 adversarial linear-time tests.
- **`tests/pcre.tcyr`** — 83 PCRE assertions, 7 deferred-feature codes,
  step-limit-guard verification.
- **`tests/fuzzy.tcyr`** — 45 fuzzy assertions.
- **`tests/vim.tcyr`** — 88 vim assertions across 13 groups (all 4 modes,
  `\<`/`\>`, `\zs`/`\ze`, all 12 POSIX bracket classes, anchors,
  lazy brace, DoS-resistance).
- **`tests/{bre,re2,pcre,fuzzy,vim}.bcyr`** — per-engine bench harnesses.
- **`fuzz/{bre,re2,pcre,fuzzy,vim}.fcyr`** — per-engine fuzz harnesses.

Aggregate: `cyrius test` reports 6 files, 362 assertions all passing.
`cyrius fuzz` reports 5 files, 1636 assertions all passing.

## Dependencies

Direct (declared in `cyrius.cyml`):

- stdlib — string, fmt, alloc, io, vec, str, syscalls, assert.

## Consumers

| Consumer | Status | Notes |
|----------|--------|-------|
| [cyim](https://github.com/MacCracken/cyim) | Parser-side ready (1.2.0) | All five flavors (`bre`, `re2`, `pcre`, `fuzzy`, `vim`) ready to wire. cyim ADR 0002 keeps consumer code change at zero. |
| owl | Planned | Pager / cat-class utility. |
| agnoshi | Planned | AI shell — `fuzzy` ready for shell completion. |
| daimon | Planned | Agent orchestration — `re2` ready for DoS-safe pattern gates; `pcre` with step-limit for richer patterns; `fuzzy` for fuzzy-name match. |

## Next

Sequencing through to v1.0:

1. **M4.5 — v0.9.0 catch-up release** — consolidates 11 deferred
   features that ADRs 0003/0004/0005 previously labeled "post-v1.0",
   plus the bre/vim backref decision flagged from M1 + M4.
2. **M5 (post-v0.9.0)** — P(-1) hardening + closeout + surface freeze.
3. **v1.0** — fold-ready release.

See [`roadmap.md`](roadmap.md) for the full plan.
