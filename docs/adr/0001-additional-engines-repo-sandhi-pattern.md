# 0001 — niyama is the additional-engines repo, following the sandhi-pattern fold lifecycle

**Status**: Accepted
**Date**: 2026-04-28

## Context

The AGNOS-lineage Cyrius ecosystem needs more than one regex engine.
Cyrius stdlib `lib/regex.cyr` ships **one foundational engine** as of
v5.7.23: a Pike NFA / POSIX-ERE-ish matcher (engine landed at
v5.7.18, exposed via the `regex_*` ABI at v5.7.23). Engine
characteristics:

- Single flavor (POSIX ERE-ish): `\d`/`\w`/`\s`/`\b`, char classes,
  `{n,m}`, alternation, groups, lazy quantifiers.
- Pike-style multi-thread NFA execution — no catastrophic backtracking,
  but no PCRE features (no lookaround, no atomic groups, no backref
  in pattern).
- Lazy bump-init (`_re_m_lazy_init`); compile is single-shot per
  process (not reentrant). No `regex_free` — engine state lives until
  process exit.

That covers the **90% case**. The remaining 10% — Perl-compatible
features, vim-style modes, RE2's linear-time guarantee, BRE-strict
POSIX compatibility, fuzzy (Levenshtein / typo-tolerant) matching —
each have real but narrower demand.

Three placement options for the additional engines:

1. **Expand stdlib `lib/regex.cyr`** to host all engines. Bloats the
   stdlib's CVE surface, fuzz target, and per-release perf
   commitments; punishes consumers who only need the foundational
   engine with a heavier stdlib.
2. **One repo per additional engine** (`bre-cyr`, `pcre-cyr`,
   `re2-cyr`, etc.). Maximum modularity but worst discoverability —
   downstream consumers have to find each one separately, and
   shared dispatch infrastructure (option-bag struct, parser, error
   handling) gets duplicated five times.
3. **One additional-engines repo** holding all alternative flavors
   under a unified niyama-side dispatch surface. Foldable into stdlib
   later if multi-consumer demand earns it.

Cyrius ecosystem has a strong **sandhi precedent** for option 3:
[`sandhi`](https://github.com/MacCracken/sandhi) (Sanskrit सन्धि —
*junction, joining*) shipped as a standalone repo for the AGNOS
service-boundary layer (HTTP/1.1 + HTTP/2 client, JSON-RPC, service
discovery, TLS policy, HTTP server). Hit 1.0.0 with surface frozen
per its own ADR 0005, then got vendored byte-identical into Cyrius
stdlib at v5.7.0 as `lib/sandhi.cyr` per its own ADR 0002. Upstream
repo entered maintenance mode after the fold.

The sandhi-pattern proves the lifecycle works: out-of-tree means
"second-class today" only in the *discoverability* sense, not the
*architecture* sense. Stdlib promotion is a real, achievable end
state.

## Decision

**niyama is the additional-engines repo for the AGNOS-lineage Cyrius
ecosystem, following the sandhi-pattern fold lifecycle.**

Concretely:

1. niyama holds **multiple alternative regex engines** (M1: bre,
   M2: re2, M3: pcre, M3.5: fuzzy, M4: vim) under a unified
   niyama-side dispatch surface — not one engine per repo.
2. Each engine ships as a sub-module under a common `niyama_*` ABI
   (e.g. `niyama_bre_compile`, `niyama_re2_compile`). The exact
   per-engine ABI shape is finalized per engine in its own ADR.
3. The fold-ready artifact is `dist/niyama.cyr` — single-file
   include (sandhi precedent — see [`dist/sandhi.cyr`](https://github.com/MacCracken/sandhi/blob/main/dist/sandhi.cyr)).
4. **Fold-back gate to cyrius stdlib `lib/niyama.cyr`** requires
   ALL FOUR:
   - ≥2 long-horizon AGNOS-lineage consumers actively using niyama
     (per the `feedback_placement_pushback` rule recorded in cyim's
     memory).
   - niyama at 1.0.0 with public surface frozen.
   - Comprehensive test + fuzz coverage (sandhi shipped with 649
     test assertions at fold time).
   - An explicit fold ADR in this repo recording the decision
     (sandhi ADR 0002 "clean-break-fold-at-cyrius-v5-7-0" is the
     template).
5. After fold, niyama enters maintenance mode — subsequent patches
   land via the Cyrius release cycle, not this repo (sandhi precedent).
6. **Speculative cyrius 5.8.0 fold target.** If multi-consumer
   adoption is achieved by then, fold lands at 5.8.0. *This is
   conditional, not a deadline.* If consumer count isn't there,
   niyama stays out-of-tree until it earns the fold.

## Consequences

### Positive

- **Single discovery point** for additional regex engines — downstream
  consumers add one dep (`niyama`) instead of five.
- **Shared dispatch infrastructure** — option-bag struct, parser
  helpers, error handling, per-engine fuzz harness machinery all live
  once.
- **cyim's `--regex=<flavor>` parser-side picks up niyama flavors
  automatically** once linked. cyim ADR 0002 already records that
  consumer code change for new flavors is zero — extend the
  flavor-name switch in `_regex_flavor_id` and add a dispatch arm in
  `_matcher_regex`.
- **Fold path is real, not aspirational.** sandhi ADR 0002 is the
  template; the gate criteria are concrete.
- **Stdlib stays lean.** Cyrius `lib/regex.cyr` remains one engine
  with one CVE surface and one perf commitment — additional engines
  carry their own commitments inside niyama until the fold.

### Negative

- **Discoverability cost vs. stdlib placement.** A `niyama` external
  dep is one `cyrius.cyml [deps]` entry away, but downstream
  maintainers have to know it exists. Mitigated by:
  - Listing niyama in agnosticos's
    [`shared-crates.md`](https://github.com/MacCracken/agnosticos/blob/main/docs/development/applications/shared-crates.md)
    once it ships.
  - Cross-linking from cyim's CHANGELOG / ADR 0002.
  - The sandhi precedent: `lib/sandhi.cyr` was discoverable via the
    sandhi repo for nearly a year before the fold landed; in-ecosystem
    consumers found it.
- **Per-engine ABI surface design has to be done up-front.** All
  engines need to compose under one niyama-side dispatch model. If
  one engine's natural ABI is wildly different from the others
  (pcre's lookaround vs. re2's linear-time-only), the dispatch layer
  takes the cost. Acceptable — that's exactly what dispatch layers
  are for.

### Neutral

- **Per-engine ADRs follow.** Each engine gets its own ADR recording
  the flavor spec it implements, the API surface, and any
  non-obvious quirks. ADR 0002+ are reserved for the engines.
- **Fuzz / bench / audit infrastructure** — sandhi-pattern repo means
  one hardening pass at M5 covers all engines. Not zero work, but not
  five-times-the-work either.

## Alternatives considered

- **Expand stdlib `lib/regex.cyr`** — rejected. Bloats stdlib, punishes
  consumers who only need the foundational engine, and locks future
  engine work to the Cyrius release cycle (slower iteration).
- **One repo per engine** — rejected. Worst discoverability, worst
  duplication of dispatch infrastructure. Per-engine repos make sense
  for engines with wildly divergent operational requirements (e.g. a
  cloud-hosted regex service vs. a local matcher), not for cohesive
  pattern-matching primitives.
- **In-tree-in-cyim implementation** — rejected per cyim's
  `feedback_placement_pushback` rule (count downstream consumers; ≥2
  long-horizon consumers means lineage-level placement). cyim isn't
  the only consumer of additional regex engines; orphan-copying the
  same engine across cyim, owl, agnoshi, daimon would be a
  maintenance disaster.
- **Make sandhi-pattern fold-back a hard cyrius 5.8.0 deadline** —
  rejected. The fold gate is consumer-count, not a calendar date.
  5.8.0 is recorded as a *speculative* target conditional on multi-
  consumer adoption; if adoption isn't there, niyama stays
  out-of-tree until it is.

## References

- [sandhi ADR 0002 — clean-break fold at cyrius v5.7.0](https://github.com/MacCracken/sandhi/blob/main/docs/adr/0002-clean-break-fold-at-cyrius-v5-7-0.md) — template for niyama's eventual fold ADR.
- [sandhi ADR 0005 — public surface freeze at 0.9.2](https://github.com/MacCracken/sandhi/blob/main/docs/adr/0005-public-surface-freeze-at-0-9-2.md) — template for niyama's pre-1.0 surface freeze.
- [cyim ADR 0002 — `--regex=<flavor>` extensibility shape](https://github.com/MacCracken/cyim/blob/main/docs/adr/0002-regex-extensibility-shape.md) — first consumer's surface; records that new flavors land as one parser arm + one dispatch arm.
- [agnosticos `shared-crates.md`](https://github.com/MacCracken/agnosticos/blob/main/docs/development/applications/shared-crates.md) — where niyama gets listed for ecosystem discoverability once it ships engines.
- [first-party-standards](https://github.com/MacCracken/agnosticos/blob/main/docs/development/applications/first-party-standards.md) — AGNOS-lineage project conventions niyama conforms to.
