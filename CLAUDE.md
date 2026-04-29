# niyama — Claude Code Instructions

> **Core rule**: this file is **preferences, process, and procedures** —
> durable rules that change rarely. Volatile state (current version,
> module line counts, supported backends, test counts, dep-gap status,
> consumers) lives in [`docs/development/state.md`](docs/development/state.md).
> Do not inline state here.

## Project Identity

**niyama** (Sanskrit नियम — *rule, regulation, restraint*) — additional regex
engines (`bre`, `re2`, `pcre`, `fuzzy`, `vim`) for the AGNOS-lineage Cyrius
ecosystem. Out-of-tree home, foldable into Cyrius stdlib once consumer count
earns it (sandhi-pattern lifecycle).

- **Type**: Library (engine modules) + small CLI wrapper for testing
- **License**: GPL-3.0-only
- **Language**: Cyrius (toolchain pinned in `cyrius.cyml [package].cyrius`)
- **Version**: `VERSION` at the project root is the source of truth — do not inline the number here
- **Standards**: [First-Party Standards](https://github.com/MacCracken/agnosticos/blob/main/docs/development/applications/first-party-standards.md) · [First-Party Documentation](https://github.com/MacCracken/agnosticos/blob/main/docs/development/applications/first-party-documentation.md)
- **Sister projects**: [cyrius stdlib `lib/regex.cyr`](https://github.com/MacCracken/cyrius) (foundational Pike NFA engine — ERE flavor, the universal floor), [sandhi](https://github.com/MacCracken/sandhi) (fold-pattern precedent), [vyakarana](https://github.com/MacCracken/vyakarana) (sister Sanskrit-grammar lib)

## Goal

niyama owns the **additional-regex-engines** slot in the AGNOS Cyrius
ecosystem. Cyrius stdlib `lib/regex.cyr` ships one foundational engine (Pike
NFA, POSIX-ERE-ish — the 90% case). Engines that don't fit that case (PCRE
features, vim-flavor modes, RE2's linear-time guarantee, BRE-strict, fuzzy
matching) live here, behind a unified niyama-side dispatch surface.
Foldable into Cyrius stdlib at v5.8.0 (speculative, conditional on
consumer count) per niyama ADR 0001.

## Consumers

- **[cyim](https://github.com/MacCracken/cyim)** v1.2.0+ — `--regex=<flavor>` already threaded; new flavors land as one elif arm in `_regex_flavor_id` per cyim ADR 0002.
- **[owl](https://github.com/MacCracken/owl)** — pager / cat-class utility; filtering and search.
- **[agnoshi](https://github.com/MacCracken/agnoshi)** — AI shell; `fuzzy` flavor (M3.5) for completion.
- **[daimon](https://github.com/MacCracken/daimon)**-orchestrated agents — `re2` flavor (M2) for DoS-safe pattern gates on untrusted input.

## Current State

> Volatile state lives in [`docs/development/state.md`](docs/development/state.md) —
> current version, surface area, in-flight work, consumers, dep gaps.
> Refreshed every release.

This file (`CLAUDE.md`) is durable rules.

## Scaffolding

Project was scaffolded with `cyrius init` (greenfield) or `cyrius port` (Rust → Cyrius migration). **Do not manually create project structure** — use the tools. If a tool is missing something, fix the tool.

## Quick Start

```sh
cyrius deps                          # resolve sibling deps
cyrius build src/main.cyr build/niyama
cyrius test                          # run [build].test + tests/*.tcyr
```

## Key Principles

- **Correctness over cleverness** — regex engines are CVE-rich territory; bugs own you fast.
- **Reference, don't mimic.** PCRE / RE2 / vim are references for flavor specs, not implementation templates. niyama is what each flavor looks like designed today, in sovereign Cyrius, without 30 years of accumulated bug-compatible quirks.
- **Lineage-level placement, not in-tree-in-cyim.** Per cyim's `feedback_placement_pushback` rule: count downstream consumers; ≥2 long-horizon consumers means lineage-level placement. niyama is that lineage-level home; the multi-consumer argument is the load-bearing reason it's a separate repo.
- **Sandhi-pattern fold-readiness from day one.** Every engine ships under the unified niyama-side dispatch model so the single-file `dist/niyama.cyr` artifact stays vendor-ready.
- Test + fuzz after EVERY engine change, not after the engine is "done".
- ONE engine at a time per milestone — never bundle.
- Build with `cyrius build`, not raw `cat file | cc5` — the manifest auto-resolves deps.
- Every buffer declaration is a contract: `var buf[N]` = N **bytes**, not N entries.

## Rules (Hard Constraints)

- **Read the [first-party-standards](https://github.com/MacCracken/agnosticos/blob/main/docs/development/applications/first-party-standards.md) first** — niyama conforms to AGNOS-lineage project conventions.
- **Do not commit or push** — the user handles all git operations.
- **NEVER use `gh` CLI** — use `curl` to the GitHub API if needed.
- **Sandhi-pattern fold model is load-bearing.** Don't propose folding niyama into cyrius stdlib before the fold gate is met (≥2 long-horizon consumers + 1.0.0 + frozen surface + explicit fold ADR per niyama ADR 0001).
- **Per-engine ABI naming is `niyama_<flavor>_*`** — e.g. `niyama_bre_compile`, `niyama_re2_search`. Mirrors cyrius stdlib's `regex_*` naming pattern. Each engine adds an ADR finalizing its specific ABI shape.
- Do not add unnecessary dependencies — niyama is stdlib-only through M5 (no external Cyrius deps).
- Do not skip tests/fuzz before claiming changes work.
- Do not use `sys_system()` with unsanitized input — command injection.
- Do not trust external data (file / network / args) without validation.
- Do not modify `lib/` files (vendored stdlib).
- Do not hardcode toolchain versions in CI YAML — `cyrius = "X.Y.Z"` in `cyrius.cyml` is the source of truth.

## Documentation

- [`docs/adr/`](docs/adr/) — Architecture Decision Records (*why X over Y?*)
- [`docs/architecture/`](docs/architecture/) — Non-obvious constraints (*what's true about the code?*)
- [`docs/guides/`](docs/guides/) — Task-oriented how-tos
- [`docs/examples/`](docs/examples/) — Runnable examples
- [`docs/development/state.md`](docs/development/state.md) — Live state snapshot
- [`docs/development/roadmap.md`](docs/development/roadmap.md) — Milestones through v1.0

## Process

1. **Work phase** — features, roadmap items, bug fixes
2. **Build check** — `cyrius build`
3. **Test + benchmark additions** for new code
4. **Internal review** — performance, memory, correctness, edge cases
5. **Documentation** — update CHANGELOG, `docs/development/state.md`, any ADR the change earned
6. **Version sync** — `VERSION`, `cyrius.cyml`, CHANGELOG header

