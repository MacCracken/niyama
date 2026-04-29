# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

## [0.1.0] — 2026-04-28

Initial scaffold. Repo positioning, doc-tree, ADR 0001 (sandhi-pattern
positioning + fold lifecycle), roadmap to v1.0. No engines shipped
yet — engine work begins at M1 (POSIX BRE).

### Added

- Project scaffold via `cyrius init niyama` (first-party-standards
  conformance: VERSION, cyrius.cyml, CLAUDE.md, CHANGELOG.md,
  README.md, LICENSE, CI workflows, doc-tree).
- README.md positioning: additional-regex-engines repo for the
  AGNOS-lineage Cyrius ecosystem, sandhi-pattern lifecycle, M0–v1.0
  engine roadmap, planned consumers (cyim, owl, agnoshi, daimon).
- ADR 0001 — niyama as the additional-engines repo following the
  sandhi-pattern fold lifecycle. Records:
  - Why one foundational engine in cyrius stdlib + additional engines
    in niyama (rather than per-engine repos or stdlib expansion).
  - The fold-back gate (≥2 long-horizon consumers + 1.0.0 + frozen
    surface + explicit fold ADR per sandhi ADR 0002 template).
  - Speculative cyrius 5.8.0 fold target (CONDITIONAL on consumer
    count — not a deadline).
- `docs/development/roadmap.md` — M0 (scaffold, done), M1 (bre — first
  engine, picked to shake out dispatch surface), M2 (re2, DoS-safe),
  M3 (pcre, largest fuzz target), M3.5 (fuzzy, Levenshtein), M4 (vim
  flavor), M5 (hardening + freeze), v1.0 (fold-ready).
- `docs/development/state.md` — initial scaffold snapshot.
- `dist/niyama.cyr` — placeholder for the fold-ready single-file
  artifact (sandhi precedent: `dist/sandhi.cyr` is what stdlib
  vendored byte-identical at the fold).
