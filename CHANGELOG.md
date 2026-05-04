# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

## [0.2.0] — 2026-05-03

M1 — first engine shipped: POSIX BRE (`niyama_bre_*`). Per ADR 0002,
ABI mirrors stdlib `regex_*` shape; `\1`-`\9` backreferences are
rejected at compile time with `BRE_E_BACKREF_UNSUPPORTED` (deferred
to potentially post-v1.0 — explicit reject + document, never silent
skip).

### Added

- `src/bre.cyr` — POSIX BRE engine. Forks the cyrius stdlib
  `lib/regex.cyr` Pike NFA / Thompson construction (instruction
  model, class bitmap, matcher) with a BRE-flavor parser. All
  globals `_bre_*`-prefixed for collision-free coexistence with
  `lib/regex.cyr` in the same program.
- Public ABI: `niyama_bre_compile`, `niyama_bre_match`,
  `niyama_bre_search`, `niyama_bre_search_at`,
  `niyama_bre_group_start`, `niyama_bre_group_end`,
  `niyama_bre_last_error`. Error codes frozen at M1: `BRE_E_OK` (0),
  `BRE_E_SYNTAX` (1), `BRE_E_BACKREF_UNSUPPORTED` (2),
  `BRE_E_TOO_LARGE` (3), `BRE_E_BAD_ANCHOR` (4 — reserved).
- POSIX BRE features: literals, `.`, `*`, `\(...\)` capturing groups
  (1..9), `\{n,m\}` / `\{n,\}` / `\{n\}` quantifiers, `^` start
  anchor (only at byte 0), `$` end anchor (only at end), `[...]` /
  `[^...]` bracket expressions with ranges, common backslash
  escapes (`\.` `\*` `\\` `\n` `\t` etc.).
- POSIX-faithful literal-by-default for `+`, `?`, `(`, `)`, `{`,
  `}`, `|`. `^`/`$` are literal in mid-pattern positions.
- ADR 0002 — niyama_bre engine ABI shape and scope. Records ABI
  surface, error-code numbering (frozen ABI at M1), backref
  rejection policy + post-v1.0 reconsideration path.
- `tests/bre.tcyr` — 68 unit tests covering literals, dot, star,
  POSIX literal-by-default for `+`/`?`, anchor placement, bracket
  expressions, groups + brace quantifiers, escapes, backref
  rejection, syntax error paths, search semantics, and a
  catastrophic-backtracking-class adversarial pattern (asserts
  linear-time DoS-resistance).
- `fuzz/bre.fcyr` — randomized fuzz harness, 200-iter sweep over
  metachar-heavy random patterns; smoke corpus drawn from past
  stdlib regex bug-fix history (3-way alt regression at v5.7.18,
  body-pc re-entry regression).
- `tests/bre.bcyr` — bench harness across compile + search paths.

### Performance floor (M1, x86_64, cyrius 5.8.42)

- `bre_compile_*` — ~3 μs avg (literal, dot-star, quantifier, group).
- `bre_search_literal_hit` (256-byte input, 6-byte needle): ~44 μs.
- `bre_search_literal_miss` (22-byte input): ~9 μs.
- `bre_search_dot_star` (256-byte input, `.*needle.*`): ~112 μs.
- `bre_search_anchored` (`^needle`): ~2 μs (anchor short-circuits).
- `bre_search_group` (256-byte input, two captures): ~16 μs.

Floor recorded for M5 hardening regression detection.

### Changed

- Cyrius pin bumped: `5.7.24` → `5.8.42` (`cyrius.cyml [package].cyrius`).
- `dist/niyama.cyr` — placeholder replaced with the M1 bundle
  (single-include over `src/bre.cyr`). `NIYAMA_VERSION` now `"0.2.0"`.

### Deferred (not in M1)

- `\1`-`\9` backreferences — rejected at compile per ADR 0002.
  Patterns needing backrefs should use `niyama_pcre` (M3); BRE
  backref support is potentially post-v1.0 work, gated on a real
  consumer ask.
- GNU `\<` / `\>` word boundaries — deferred to M4 (vim flavor
  inherits the same semantics).
- `[:alpha:]` POSIX bracket character classes — deferred to M4.

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
