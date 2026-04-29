# niyama — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.1.0** — scaffolded 2026-04-28 via `cyrius init`. Doc-tree, ADR
0001 (positioning + sandhi-pattern fold lifecycle), README, roadmap,
CHANGELOG seeded. No engines shipped yet — engine work begins at M1
(POSIX BRE).

## Toolchain

- **Cyrius pin**: `5.7.24` (in `cyrius.cyml [package].cyrius`)

## Source

Initial scaffold only — `src/main.cyr` and `src/test.cyr` are the
`cyrius init` defaults. Per-engine source files (`src/bre.cyr`,
`src/re2.cyr`, etc.) land at their respective milestones.

## Fold-ready artifact

- `dist/niyama.cyr` — placeholder. Single-file include is the
  fold-ready shape (sandhi precedent: `dist/sandhi.cyr` is what
  cyrius stdlib vendored byte-identical at v5.7.0). Becomes
  load-bearing at M5 / v1.0.

## Tests

- `tests/niyama.tcyr` — primary suite (smoke + math; passes on `cyrius test`)
- `tests/niyama.bcyr` — benchmark stub (no-op)
- `tests/niyama.fcyr` — fuzz stub

## Dependencies

Direct (declared in `cyrius.cyml`):

- stdlib — string, fmt, alloc, io, vec, str, syscalls, assert

## Consumers

| Consumer | Status | Notes |
|----------|--------|-------|
| [cyim](https://github.com/MacCracken/cyim) | Parser-side ready (1.2.0) | `--regex=<flavor>` already threaded; new flavors land as one elif arm in `_regex_flavor_id` + one dispatch arm in `_matcher_regex` per cyim ADR 0002. Will start consuming niyama once M1 (`bre`) ships. |
| owl | Planned | Pager / cat-class utility. Filtering and search are the obvious fit. |
| agnoshi | Planned | AI shell — `fuzzy` flavor (M3.5) is the first immediate win. |
| daimon | Planned | Agent orchestration — `re2` flavor (M2) for DoS-safe pattern gates on untrusted input. |

## Next

See [`roadmap.md`](roadmap.md).
