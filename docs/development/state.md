# niyama — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.3.0** — M2 shipped 2026-05-03. niyama_re2 engine live: linear-time
ERE matcher with explicit compile-time rejection of non-regular
features (backref, lookaround, atomic groups, recursion), each with
its own error code per ADR 0003. Pike NFA dedup gives the structural
linear-time guarantee — backtracking-killer patterns like `(a|a)*b`
on 200 `a`s match in ~84μs.

## Toolchain

- **Cyrius pin**: `5.8.42` (in `cyrius.cyml [package].cyrius`)

## Source

- `src/main.cyr` — smoke entry (prints identity banner). Will become
  the dispatch surface once the M3+ engines accumulate enough that
  per-engine name-resolution gets clearer.
- `src/test.cyr` — top-level test entry per `[build].test`.
- **`src/bre.cyr`** — POSIX BRE engine (M1). ~600 lines.
- **`src/re2.cyr`** — RE2-flavor linear-time engine (M2). ~700 lines.
- Future: `src/pcre.cyr` (M3), `src/fuzzy.cyr` (M3.5), `src/vim.cyr` (M4).

## Engines shipped

| Engine | Status | ABI prefix | Notes |
|---|---|---|---|
| **bre** | ✅ v0.2.0 (M1) | `niyama_bre_*` | POSIX BRE minus backrefs (per ADR 0002). 68 unit tests pass. |
| **re2** | ✅ v0.3.0 (M2) | `niyama_re2_*` | ERE + linear-time guarantee at API (per ADR 0003). 76 unit tests pass; backref/lookaround/atomic/recursion rejected at compile with distinct error codes. |
| pcre | planned (M3) | `niyama_pcre_*` | Perl-compat including backrefs / lookaround / atomic groups. |
| fuzzy | planned (M3.5) | `niyama_fuzzy_*` | Levenshtein / typo-tolerant. |
| vim | planned (M4) | `niyama_vim_*` | magic / nomagic / very-magic / very-nomagic. |

## Fold-ready artifact

- `dist/niyama.cyr` — single-include bundle. v0.3.0 bundles
  `src/bre.cyr` + `src/re2.cyr`. Will accumulate engines through M4;
  M5 surface freeze ADR will lock its public symbol set.

## Tests

- `tests/niyama.tcyr` — scaffold smoke (2 assertions).
- **`tests/bre.tcyr`** — 68 BRE assertions across 13 groups.
- **`tests/re2.tcyr`** — 76 RE2 assertions across 11 groups, including
  4 adversarial linear-time tests.
- **`tests/bre.bcyr`** — BRE bench (4 compile + 7 search benches).
- **`tests/re2.bcyr`** — RE2 bench (3 compile + 4 search + 3 DoS-resistance benches).
- **`fuzz/bre.fcyr`** — BRE randomized fuzz harness.
- **`fuzz/re2.fcyr`** — RE2 fuzz harness with adversarial pattern
  generator (4 rejection-invariant checks + 3 backtracking-killer
  patterns × 200-byte adversarial input).

Aggregate: `cyrius test` reports 3 files (`niyama.tcyr` + `bre.tcyr` +
`re2.tcyr`), 146 assertions all passing.

## Dependencies

Direct (declared in `cyrius.cyml`):

- stdlib — string, fmt, alloc, io, vec, str, syscalls, assert
  (auto-included for `[build].test`; per-test/bench files include
  the explicit deps they need — `lib/alloc.cyr`, `lib/string.cyr`,
  `lib/fnptr.cyr`, `lib/bench.cyr`, etc.).

## Consumers

| Consumer | Status | Notes |
|----------|--------|-------|
| [cyim](https://github.com/MacCracken/cyim) | Parser-side ready (1.2.0) | Both `bre` and `re2` flavors are now ready to wire — cyim consumer integration unblocked for both. |
| owl | Planned | Pager / cat-class utility. |
| agnoshi | Planned | AI shell — `fuzzy` flavor (M3.5) is the first immediate win. |
| daimon | Planned | Agent orchestration — **`re2` is now ready** for DoS-safe pattern gates on untrusted input. The headline use case for niyama_re2. |

## Next

M3 — `niyama_pcre` engine (Perl-compatible: lookaround, atomic groups,
backrefs, named captures, etc.). Largest fuzz target by far —
diff-testing against re2 + bre on overlapping syntax becomes load-bearing.
See [`roadmap.md`](roadmap.md).
