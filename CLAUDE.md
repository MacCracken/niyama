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

