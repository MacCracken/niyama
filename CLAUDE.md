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
- **Do not unilaterally defer / descope features.** The roadmap is the user's commitments, not yours. If a milestone says "Initial scope includes X, Y, Z," ship X, Y, Z — or stop and ask before cutting any of them. "Lets keep it moving" is permission for momentum, not for scope decisions. Asking once at the start does NOT cover later cuts; every "this is bigger than fits cleanly" moment is a separate decision to surface. Do not use an ADR as the asking-mechanism — the deferral has to be agreed-to before it lands in the ADR.
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

Process structure follows the AGNOS-lineage convention from
[example_claude.md § Process](https://github.com/MacCracken/agnosticos/blob/main/docs/development/applications/example_claude.md#process)
with niyama-specific notes (per-engine bench harnesses, PCRE2 CVE
corpus as the highest-risk surface, cyim as consumer #1).

### P(-1): Scaffold / Project Hardening (before any new features)

niyama runs P(-1) before each major-band closeout. **v0.9.0 IS the
P(-1) for v1.0 fold-ready freeze** — see roadmap.md §
"M5 / v0.9.0 — P(-1) hardening + closeout + surface freeze".

1. **Cleanliness baseline** — `cyrius build`, `cyrius lint`,
   `cyrius audit`; all `tests/*.tcyr` pass, all `fuzz/*.fcyr`
   pass.
2. **Benchmark baseline** — `cyrius bench tests/<engine>.bcyr`
   for each of the 5 per-engine harnesses (`bre`, `re2`, `pcre`,
   `fuzzy`, `vim`); CSV recorded for regression detection through
   the rest of the hardening pass and into v1.0.
3. **Internal deep review** — gaps, optimizations, correctness,
   edge cases, ABI consistency across engines, parallel-codepath
   identification (matcher loops, opcode-emission helpers,
   codepoint-decode duplication accreted across v0.7.0/v0.8.0).
4. **External research** — domain completeness check (PCRE2
   feature coverage, vim flavor coverage, RE2 spec coverage),
   best-practices review, **PCRE2 CVE corpus cross-check**
   against `niyama_pcre` — backtracker, step-limit guard,
   recursion handling, lookbehind/UCHAR_CI additions.
5. **Security audit** — per first-party-standards § Security
   Hardening (see below). File findings in
   `docs/audit/YYYY-MM-DD-audit.md`.
6. **Additional tests / fuzz / benchmarks** from findings.
7. **Post-review benchmarks** — prove wins (or document neutral
   refactors) against step-2 baseline.
8. **Documentation audit** — ADRs for hardening decisions,
   public API guides, surface freeze ADR (template: sandhi
   ADR 0005). Roll deferred CLAUDE.md sections (Cyrius
   Conventions, CI/Release, Documentation Structure, .gitignore,
   CHANGELOG Format) into this step at v1.0 closeout.
9. **Repeat if heavy** — keep drilling until clean.

### Work Loop (continuous)

1. **Work phase** — features, roadmap items, bug fixes.
2. **Build check** — `cyrius build`.
3. **Test + fuzz + bench additions** for new code; existing
   harnesses must pass.
4. **Internal review** — performance, memory, correctness,
   edge cases, per-engine ABI shape preserved.
5. **Documentation** — CHANGELOG, `docs/development/state.md`,
   any ADR the change earned.
6. **Version sync** — `VERSION`, `cyrius.cyml`,
   CHANGELOG header.

### Security Hardening (before every release)

Per [first-party-standards § Security Hardening](https://github.com/MacCracken/agnosticos/blob/main/docs/development/applications/first-party-standards.md#security-hardening-new--required-before-every-release).
Minimum:

1. **Input validation** — every public `niyama_<engine>_compile()`
   / `_match()` / `_search()` validates pattern + input bounds.
2. **Buffer safety** — every `var buf[N]` audited; opcode-encoded
   instruction buffers, class bitmaps, name tables, save arrays,
   step counters all sized correctly. N is **bytes**, not entries.
3. **Syscall review** — niyama's syscall surface is small (alloc,
   load/store via stdlib); each one validated.
4. **Pointer validation** — every `load8` / `load64` / `store8`
   / `store64` against pattern or input pointers has an upstream
   bounds check.
5. **No command injection** — N/A; niyama doesn't shell out.
6. **No path traversal** — N/A; niyama doesn't touch the
   filesystem.
7. **Known-CVE review** — **PCRE2 CVE corpus is the highest-risk
   surface.** Cross-check `niyama_pcre`'s backtracker, step-limit
   guard, recursion handling, and the v0.8.0 additions
   (lookbehind, recursion, UCHAR_CI) against known PCRE2 issue
   patterns.
8. **Document findings** — `docs/audit/YYYY-MM-DD-audit.md`.

Severity levels: **CRITICAL** (remote / privilege escalation —
e.g. heap overflow via crafted pattern), **HIGH** (moderate effort
— e.g. unbounded backtracking that escapes the step limit),
**MEDIUM** (specific conditions — e.g. UTF-8 decode at unaligned
position with adjacent buffer leak), **LOW** (defense in depth).

### Closeout Pass (before every minor/major bump)

Run before tagging X.Y.0 or X.0.0. Ship as the last patch of the
current minor (e.g. 0.8.5 before 0.9.0). For niyama right now,
**v0.9.0 is the closeout for v1.0**.

1. **Full test suite** — all `tests/*.tcyr` pass, zero failures.
   `cyrius fuzz` passes too.
2. **Benchmark baseline** — `cyrius bench tests/*.bcyr`; CSV
   compared against the prior closeout (v0.7.0 → v0.8.0 → v0.9.0).
3. **Dead code audit** — remove unused functions; record floor
   in CHANGELOG.
4. **Refactor pass** — consolidate parallel codepaths from the
   minor's additions. v0.8.0 carved 7 features across 5 engines
   and added a codepoint-stepped matcher loop in re2 + vim;
   look for accreted matcher patterns + opcode-emission helpers
   + UTF-8 decode duplication that should fold.
5. **Code review pass** — walk diffs end-to-end for missed
   guards, ABI leaks, off-by-ones, silently-ignored errors.
6. **Cleanup sweep** — stale comments, dead branches, unused
   includes, orphaned helpers.
7. **Security re-scan** — quick grep for new unchecked writes,
   buffer size mismatches, unsanitized input paths.
8. **Downstream check** — cyim builds + tests against the new
   niyama version (cyim is consumer #1 per state.md). When a
   second long-horizon consumer materializes (owl / agnoshi /
   daimon), the list grows.
9. **Doc sync** — CHANGELOG, roadmap, `docs/development/state.md`,
   CLAUDE.md (if durable content changed).
10. **Version verify** — `VERSION`, `cyrius.cyml`, CHANGELOG
    header, intended git tag all match.
11. **Full build from clean** —
    `rm -rf build && cyrius deps && CYRIUS_DCE=1 cyrius build`
    passes clean.

### Task Sizing

- **Low/Medium effort**: batch freely — multiple items per work
  loop cycle. v0.7.0 batched 8 features; v0.8.0 batched 7.
- **Large effort**: small bites only — break into sub-tasks,
  verify each before moving on. The v0.8.0
  `\p{L}` → multi-byte literals → `(?i)` Unicode → fuzzy NFD
  chain ran this way; each step's tests went green before the
  next started.
- **If unsure**: treat as large.

### Refactoring Policy

- Refactor when the code tells you to — duplication, unclear
  boundaries, measured bottlenecks. The v0.8.0 vim →
  posix_classes refactor followed this; duplication was visible
  and the test suite covered the move.
- Never refactor speculatively. Wait for the third instance.
- Every refactor must pass the same test + fuzz + benchmark
  gates as new code.
- 3 failed attempts = defer and document — don't burn time in
  a rabbit hole.

## Cyrius Conventions

Lifted from
[agnosticos example_claude.md § Cyrius Conventions](https://github.com/MacCracken/agnosticos/blob/main/docs/development/applications/example_claude.md#cyrius-conventions).
These are toolchain-level invariants — true for every Cyrius
project, not niyama-specific.

- All struct fields are 8 bytes (`i64`), accessed via `load64` /
  `store64` with offset.
- Heap allocation via `fl_alloc()` / `fl_free()` (freelist) for
  data with individual lifetimes.
- Bump allocation via `alloc()` for long-lived data (vec, str
  internals, niyama's NFA structures, class tables, name tables).
- Enum values for constants — don't consume `gvar_toks` slots
  (4 096 initialized globals limit).
- Heap-allocate large buffers — `var buf[256000]` bloats the
  binary by 256 KB.
- `break` in while loops with `var` declarations is unreliable —
  use flag + `continue`. (See niyama's `_<engine>_parse_class`
  loops for the canonical pattern.)
- No negative literals — write `(0 - N)` not `-N`. niyama uses
  `0 - 1` for "no match" / "not found" returns throughout.
- No mixed `&&` / `||` in one expression — nest `if` blocks
  instead.
- `match` is reserved — don't use as a variable name.
- `return;` without value is invalid — always `return 0;`.
- All `var` declarations are function-scoped — no block scoping.
- Max limits per compilation unit: 4 096 variables, 1 024
  functions, 4 096 initialized globals. niyama's largest engine
  (pcre at ~2 000 lines) is well under all three; the dist
  artifact stays comfortably under after bundling.
- Counting rule: only a top-level `var NAME = <non-literal>;`
  (call / identifier / expression initializer) consumes an
  initialized-globals slot; a bare integer-literal init
  (`var x = 42;`) takes the static-init fast path and enum
  members are const-folded, so neither counts. See the cyrius
  guide's **Global Initializers** section
  (`docs/guides/cyrius-guide.md` in the cyrius repo).

## CI / Release

- **Toolchain pin**: `cyrius = "X.Y.Z"` field in `cyrius.cyml
  [package]`. **No separate `.cyrius-toolchain` file.** CI and
  release both read from `cyrius.cyml`. Currently pinned to
  `6.1.27` (floor is `5.8.65` per ADR 0008 for stdlib `lib/unicode/`;
  bumped to track the installed wrapper).
- **Dead code elimination**: every `cyrius build` in CI and
  release runs with `CYRIUS_DCE=1`. Binary size is a release
  metric — track in CHANGELOG.
- **Tag filter**: release workflow triggers on
  `tags: ['[0-9]*']` — semver-only. Non-numeric tags do not ship
  a release.
- **Version-verify gate**: release asserts `VERSION ==
  cyrius.cyml version == git tag` before building. Mismatch fails
  the run.
- **Lint step**: CI runs `cyrius lint` per source file across
  `src/*.cyr`. Advisory; v0.9.0 audit notes 11 long-line cosmetic
  warnings remain (cosmetic only).
- **Workflow layout** (`.github/workflows/`):
  - `ci.yml` — build, lint, test, fuzz; reusable via
    `workflow_call`.
  - `release.yml` — version gate → CI gate → DCE build →
    artifacts (source tarball, bundled `dist/niyama.cyr`, DCE
    binary, SHA256SUMS).
- **Concurrency**: CI uses `cancel-in-progress: true` keyed on
  workflow + ref — only the latest push is tested.
- **State sync**: release post-hook bumps
  `docs/development/state.md`. If the hook doesn't, fix the hook
  — don't hand-maintain state.
- **`cyrius audit`**: known-broken from 5.8.65 onward (missing
  `~/.cyrius/bin/check.sh`); verify against the live 6.1.27
  toolchain before relying on it. Run constituents individually
  (`cyrius lint` + `cyrius test` + `cyrius fuzz` + clean DCE
  build) until the toolchain bug is fixed.

## Documentation Structure

```
Root files (required):
  README.md, CHANGELOG.md, CLAUDE.md, LICENSE,
  VERSION, cyrius.cyml

Root files (recommended for first-party):
  CONTRIBUTING.md, SECURITY.md, CODE_OF_CONDUCT.md

docs/ (minimum):
  adr/       — Architecture Decision Records (README + template.md
               + NNNN-*.md). Numbered, never renumber. niyama has
               0001-0010 as of v0.9.0.
  architecture/ — Non-obvious invariants (README + NNN-*.md).
                  Numbered, never renumber. niyama has 001-004 as
                  of v0.9.0.
  guides/    — Task-oriented how-tos (currently empty; reserved).
  examples/  — Runnable examples (currently empty; reserved).
  development/
    roadmap.md — completed, backlog, future, v1.0 criteria.
    state.md   — live state snapshot (volatile; release-hook-bumped).

docs/ (when earned — niyama has all of these post-v0.9.0):
  audit/     — Security audit reports (YYYY-MM-DD-audit.md).
               niyama has `2026-05-05-audit.md` (v0.9.0 P(-1)).
  api/       — Curated public-surface reference. niyama has
               `README.md` mirroring ADR 0010's freeze contract.
  benchmarks.md — Per-release bench history. niyama has v0.8.0
                  baseline + v0.9.0 step-7 diff.
  development/v0.9.0-review-findings.md — working doc for the
                  v0.9.0 P(-1) review pass; deleted at v1.0.

docs/ (when earned — niyama doesn't have yet):
  sources.md          — academic/domain citations (mostly N/A;
                        regex theory is well-known).
  proposals/          — pre-ADR design drafts.
  standards/, compliance/, faq.md — as applicable.
```

## .gitignore (Required)

niyama-specific .gitignore — note divergence from the example
template: niyama **keeps `dist/niyama.cyr` and `lib/*.cyr`
checked in** (vendored stdlib + fold-ready artifact, both
load-bearing).

```gitignore
# Build
/build/

# Release / toolchain artifacts (post-v0.9.0 added)
cyrius-*.tar.gz
*.tar.gz
SHA256SUMS

# Crash dumps
*.core

# IDE / editors
.claude/
.idea/
.vscode/
*.swp
*.swo
*~

# OS
.DS_Store
Thumbs.db

# Secrets (defense-in-depth — niyama doesn't deal with secrets,
# but the gitignore guards anyway)
.env
.env.*
*.pem
*.key
```

**Note**: unlike the agnosticos template, niyama does NOT ignore:
- `/dist/` — `dist/niyama.cyr` is the fold-ready bundled artifact,
  byte-identical to what cyrius stdlib will vendor as
  `lib/niyama.cyr` post-fold. Checked in.
- `lib/*.cyr` — niyama vendors the cyrius stdlib subset it needs
  (alloc, string, fmt, vec, str, syscalls, assert, unicode/*).
  `cyrius update` re-vendors from the toolchain. Checked in to
  freeze the stdlib version with the niyama version.

## CHANGELOG Format

Follow [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
niyama-specific guidance:

- **Performance claims must include benchmark numbers** with the
  bench harness name. Reference `docs/benchmarks.md` rows where
  applicable. Don't hand-wave "faster than"; cite the data.
- **Breaking changes** get a `### Breaking` section with a
  migration guide. **niyama has no breaking changes through v1.0**
  — the v0.7.0 strict-default `^/$/.` shift was pre-v1.0; ADR 0010
  freeze locks the surface so post-v1.0 patches cannot break.
- **Security fixes** get a `### Security` section with CVE
  references where applicable. niyama has had no security fixes
  yet; the v0.9.0 audit found no CRITICAL/HIGH issues.
- **Per-release section structure** — niyama uses:
  - `### Changed` — modifications to existing surface.
  - `### Added` — new public symbols / features.
  - `### Deferred` — items pinned for later (with target slot).
  - `### Tests / fuzz` — assertion count diffs vs. prior release.
  - `### Bench` — regression check, ±% from prior baseline.
  - `### ABI summary` (at major / freeze releases) — opcode and
    error-code numeric counts.
- **Frozen surface notes**: post-v1.0 entries call out
  reserved-but-unused error codes / opcode slots so a future
  reader sees what's locked vs. live.
