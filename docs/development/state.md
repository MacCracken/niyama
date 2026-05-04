# niyama — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.2.0** — M1 shipped 2026-05-03. POSIX BRE engine
(`niyama_bre_*`) is live; ADR 0002 records the per-engine ABI shape
that M2-M4 will mirror. Backref rejection policy locked: `\1`-`\9`
returns `BRE_E_BACKREF_UNSUPPORTED` at compile (potentially-post-v1.0
work).

## Toolchain

- **Cyrius pin**: `5.8.42` (in `cyrius.cyml [package].cyrius`)

## Source

- `src/main.cyr` — smoke entry (prints identity banner; will become
  the dispatch surface once M2+ lands and we have multiple engines).
- `src/test.cyr` — top-level test entry per `[build].test`.
- **`src/bre.cyr`** — POSIX BRE engine (M1). ~600 lines. Forked from
  cyrius stdlib `lib/regex.cyr` Pike NFA; all globals `_bre_*`-prefixed
  for collision-free coexistence.
- Future: `src/re2.cyr` (M2), `src/pcre.cyr` (M3), `src/fuzzy.cyr`
  (M3.5), `src/vim.cyr` (M4).

## Engines shipped

| Engine | Status | ABI prefix | Notes |
|---|---|---|---|
| **bre** | ✅ v0.2.0 (M1) | `niyama_bre_*` | POSIX BRE minus backrefs (per ADR 0002). 68 unit tests pass. |
| re2 | planned (M2) | `niyama_re2_*` | Linear-time guarantee, compile-time rejection of non-regular features. |
| pcre | planned (M3) | `niyama_pcre_*` | Perl-compat including backrefs / lookaround / atomic groups. |
| fuzzy | planned (M3.5) | `niyama_fuzzy_*` | Levenshtein / typo-tolerant. |
| vim | planned (M4) | `niyama_vim_*` | magic / nomagic / very-magic / very-nomagic. |

## Fold-ready artifact

- `dist/niyama.cyr` — single-include bundle. v0.2.0 bundles
  `src/bre.cyr`. Will accumulate engines through M4; M5 surface
  freeze ADR will lock its public symbol set per the sandhi ADR
  0005 template.

## Tests

- `tests/niyama.tcyr` — scaffold smoke (2 assertions; `cyrius test`
  pass).
- **`tests/bre.tcyr`** — 68 BRE assertions across 13 groups (literals,
  dot, star, literal-by-default `+`/`?`, anchors, brackets, groups +
  brace quantifiers, literal metachars, escapes, backref rejection,
  syntax errors, search, DoS-resistance).
- **`tests/bre.bcyr`** — bench harness (4 compile + 7 search benches).
- **`fuzz/bre.fcyr`** — randomized fuzz harness (200-iter sweep + smoke
  corpus + backref-rejection invariant).
- `tests/niyama.fcyr` — scaffold stub (transitional; superseded by
  per-engine harnesses in `fuzz/`).

## Dependencies

Direct (declared in `cyrius.cyml`):

- stdlib — string, fmt, alloc, io, vec, str, syscalls, assert
  (auto-included for `[build].test`; per-test/bench files include
  the explicit deps they need — `lib/alloc.cyr`, `lib/string.cyr`,
  `lib/fnptr.cyr`, `lib/bench.cyr`, etc.).

## Consumers

| Consumer | Status | Notes |
|----------|--------|-------|
| [cyim](https://github.com/MacCracken/cyim) | Parser-side ready (1.2.0) | `--regex=<flavor>` already threaded; new flavors land as one elif arm in `_regex_flavor_id` + one dispatch arm in `_matcher_regex` per cyim ADR 0002. **`bre` is now ready to wire** — cyim consumer integration is unblocked. |
| owl | Planned | Pager / cat-class utility. |
| agnoshi | Planned | AI shell — `fuzzy` flavor (M3.5) is the first immediate win. |
| daimon | Planned | Agent orchestration — `re2` flavor (M2) for DoS-safe pattern gates. |

## Next

M2 — `niyama_re2` engine (Thompson NFA + linear-time guarantee +
compile-time rejection of non-regular features). See
[`roadmap.md`](roadmap.md).
