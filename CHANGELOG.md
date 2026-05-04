# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

## [0.4.0] — 2026-05-03

M3 — third engine: pcre (`niyama_pcre_*`). Backtracking matcher (the
first non-Pike-NFA engine in niyama) bringing the features re2
deliberately rejects: backreferences, lookahead, atomic groups, named
captures, possessive quantifiers. Catastrophic-backtracking risk is
mitigated by an explicit step-limit guard with a configurable budget
(default 1M steps) and a hard recursion-depth bound (256). Per ADR
0004.

### Added

- `src/pcre.cyr` — backtracking matcher with PCRE-flavor parser,
  ~1100 lines. New opcodes: `PCRE_OP_BACKREF`, `PCRE_OP_LOOKAHEAD`,
  `PCRE_OP_NLOOKAHEAD`, `PCRE_OP_LOOKAHEAD_END`, `PCRE_OP_ATOMIC`,
  `PCRE_OP_ATOMIC_END`. Same instruction layout as bre/re2 for
  shared opcodes.
- Public ABI mirroring niyama_bre / niyama_re2 plus pcre-specific
  extensions:
  - `niyama_pcre_compile` / `_match` / `_search` / `_search_at`
  - `niyama_pcre_group_start` / `_group_end` (groups 0..9)
  - `niyama_pcre_group_by_name(nfa, name)` — named-capture lookup
  - `niyama_pcre_last_error()` — frozen error code set
  - `niyama_pcre_set_step_limit(n)` — configurable step budget
  - `niyama_pcre_last_step_count()` — observability hook
- **PCRE feature set in M3**: ERE base (literals, `.`, anchors,
  `\d`/`\w`/`\s`/`\b`, classes, `*` `+` `?` `{n,m}`, lazy,
  alternation, capturing + non-capturing), plus:
  - **Backreferences `\1`-`\9`** — the headline PCRE feature.
  - **Lookahead `(?=...)` and `(?!...)`** — variable-width.
  - **Atomic groups `(?>...)`** — no internal backtracking.
  - **Possessive quantifiers `*+`, `++`, `?+`, `{n,m}+`** —
    desugared to atomic-wrapping at compile.
  - **Named captures `(?<name>...)` and `(?P<name>...)`** — both
    syntaxes; lookup by name. Up to 9 named (shared with positional).
  - Step-limit + depth-limit catastrophic-backtracking guard.
- ADR 0004 — niyama_pcre engine ABI shape, matcher architecture,
  and scope. Records the backtracking-vs-Pike-NFA decision, the
  step-limit semantics, and the deferral list.
- `tests/pcre.tcyr` — 83 unit tests across 13 groups, including
  backref correctness, lookahead semantics, atomic-blocks-backtracking
  demo, named-capture lookup, all 7 deferred-feature rejection codes,
  and the step-limit guard kicking in on `(a+)+b` against 25 'a's.
- `fuzz/pcre.fcyr` — 229-assertion harness with adversarial pattern
  generator, every rejection invariant, and 4 catastrophic-backtracking
  patterns under tight step limit.
- `tests/pcre.bcyr` — bench harness covering compile + search +
  PCRE-specific features + the bounded-DoS bench (`(a+)+b` on 30
  'a's with `step_limit=50k` terminates in ~2.4ms).

### Performance floor (M3, x86_64, cyrius 5.8.42)

- `pcre_compile_*` — 4-5 μs avg (literal, email-class, backref,
  lookahead).
- `pcre_search_literal` (256-byte input): ~17 μs.
- `pcre_search_email`: ~9 μs.
- `pcre_search_alt` (3-way alt): ~61 μs.
- `pcre_backref` (`(\w+) \1` on `hello hello world`): ~2 μs.
- `pcre_lookahead` (`\w+(?=@)` on 256-byte input): ~180 μs (largest
  search bench — lookahead validates against multiple match start
  positions).
- `pcre_atomic` (`(?>a*)b` on `aaaaaaaaaaaaab`): ~3 μs.
- `pcre_named_captures`: ~3 μs.
- **Catastrophic-backtracking guard**: `(a+)+b` against 30 'a's with
  no terminator, `step_limit=50000` — ~2.4 ms. Without the limit
  this pattern would explore ~2^30 paths (would never terminate in
  practice).

### Changed

- `dist/niyama.cyr` — bundle now includes `src/pcre.cyr` alongside
  `src/bre.cyr` and `src/re2.cyr`. `NIYAMA_VERSION` → `"0.4.0"`.
- `src/main.cyr` smoke banner reflects M3 status.

### Deferred (not in M3 — see ADR 0004 for rationale)

- Lookbehind `(?<=...)` `(?<!...)` — needs fixed-width analysis;
  M3.5 candidate if a consumer asks. Compile rejects with
  `PCRE_E_LOOKBEHIND_UNSUPPORTED`.
- Unicode property classes `\p{L}` — needs Unicode database. Rejects
  with `PCRE_E_UNICODE_PROP_UNSUPPORTED`.
- POSIX bracket classes `[:alpha:]` — deferred to M4 (vim flavor
  inherits the same semantics).
- Recursion `(?R)` `(?P>name)` — `PCRE_E_RECURSION_UNSUPPORTED`.
- Conditional patterns `(?(...)...)` — `PCRE_E_CONDITIONAL_UNSUPPORTED`.
- Inline flags `(?i)` `(?m)` `(?s)` — generic `PCRE_E_SYNTAX`.

## [0.3.0] — 2026-05-03

M2 — second engine: re2 (`niyama_re2_*`). Linear-time Pike NFA matcher
with ERE-flavor parser and **explicit compile-time rejection of every
feature that would break the linear-time guarantee**. Each
non-regular construct gets its own error code so consumers know
exactly which engine to fall back to (niyama_pcre at M3). Per ADR
0003.

### Added

- `src/re2.cyr` — Pike NFA matcher with ERE syntax: `\d` `\D` `\w`
  `\W` `\s` `\S` `\b` `\B`, alternation `|`, `(...)` capturing
  groups, `(?:...)` non-capturing, greedy AND lazy quantifiers
  (`*` `+` `?` `{n,m}` and lazy variants). Same Pike NFA matcher
  kernel as niyama_bre — the linear-time guarantee is structural.
- Public ABI: `niyama_re2_compile`, `niyama_re2_match`,
  `niyama_re2_search`, `niyama_re2_search_at`,
  `niyama_re2_group_start`, `niyama_re2_group_end`,
  `niyama_re2_last_error`. Mirrors `niyama_bre_*` shape per ADR
  0002.
- **Compile-time rejection** of features that would break linear
  time, each with its own error code:
  - `RE2_E_BACKREF_UNSUPPORTED` (= 2) — `\1`-`\9` backreferences.
  - `RE2_E_LOOKAROUND_UNSUPPORTED` (= 3) — `(?=...)` `(?!...)`
    `(?<=...)` `(?<!...)` lookaround.
  - `RE2_E_ATOMIC_UNSUPPORTED` (= 4) — `(?>...)` atomic groups.
  - `RE2_E_RECURSION_UNSUPPORTED` (= 5) — `(?R)` `(?P>name)`
    recursion / subroutine calls.
  - `RE2_E_TOO_LARGE` (= 6) — pattern compile exceeds instruction
    or class limits.
- ADR 0003 — niyama_re2 engine ABI shape and scope. Records the
  ERE feature set, the linear-time guarantee, the rejection
  contract, and the deferred-to-post-M3 named-capture decision.
- `tests/re2.tcyr` — 76 unit tests across 11 feature groups,
  including each rejection error code AND adversarial linear-time
  patterns (`(a|a)*b`, `(a*)*b`, Cox's `a?{30}a{30}` adversary,
  deep alternation × repetition).
- `fuzz/re2.fcyr` — 221-assertion harness, including the four
  rejection-invariant checks (every pattern matching `\1`/`(?=`/
  `(?>`/`(?R` MUST yield the corresponding error code).
- `tests/re2.bcyr` — bench harness with **dedicated DoS-resistance
  benches**: `(a|a)*b` and `(a*)*b` against 200-`a` inputs that
  would DoS a backtracking engine.

### Performance floor (M2, x86_64, cyrius 5.8.42)

- `re2_compile_*` — 3-5 μs avg (literal, alt, email-class).
- `re2_search_literal` (256-byte input): ~44 μs.
- `re2_search_alt` (3-way alt over 256-byte input): ~73 μs.
- `re2_search_email`: ~15 μs.
- **DoS-resistance** (the headline numbers — these would never
  terminate on a backtracking engine):
  - `(a|a)*b` against 200 `a`s, no match: ~84 μs.
  - `(a*)*b` against 200 `a`s, no match: ~60 μs.
  - Cox's `a?{30}a{30}` adversary, match: ~201 μs.

### Changed

- `dist/niyama.cyr` — bundle now includes `src/re2.cyr` alongside
  `src/bre.cyr`. `NIYAMA_VERSION` → `"0.3.0"`.
- `src/main.cyr` smoke banner reflects M2 status.

### Deferred (not in M2)

- Named captures `(?P<name>...)` / `(?<name>...)` — deferred to
  post-M3. Both re2 and pcre will share the named-capture API
  surface; designing it once with both engines in hand avoids
  shipping it twice.
- Unicode property classes `\p{L}` — M2 is ASCII-only.
- Inline flags `(?i)` `(?m)` `(?s)` — post-v1.0 unless asked.

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
