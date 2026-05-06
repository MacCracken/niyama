# 0010 — Surface freeze for v1.0

**Status**: Accepted
**Date**: 2026-05-05

## Context

v0.9.0 is the M5 P(-1) hardening + closeout release per CLAUDE.md
§ Process. With the hardening pass clean (audit
`docs/audit/2026-05-05-audit.md`, no CRITICAL/HIGH findings; bench
`docs/benchmarks.md`, no regressions; review findings
`docs/development/v0.9.0-review-findings.md`, primaries addressed),
v0.9.0 ships the **last surface change before v1.0 fold-ready
release**.

Per niyama ADR 0001's sandhi-pattern fold lifecycle, v1.0 is the
release at which `dist/niyama.cyr` becomes a byte-identical fold
candidate for cyrius stdlib `lib/niyama.cyr` (sandhi precedent at
cyrius v5.7.0). For the fold to be sound, the public API surface
must be stable: cyrius stdlib's vendoring contract is "what's in
the file at fold time is what cyrius ships, byte-identical." A
moving target wouldn't be vendorable.

This ADR locks the surface. v0.9.0 is the last release that can
add or modify the surface; v1.0 ships it as-is.

## Decision

**The public surface defined below is frozen as of v0.9.0.**
Patches (v0.9.x, post-v1.0 stdlib-side) may *extend* the surface
under strict additive rules; they may not modify, remove, or
renumber what's frozen.

### Frozen public API (per engine)

#### `niyama_bre_*` (bre — POSIX BRE, Pike NFA)

```
niyama_bre_compile(pat)               → nfa | 0
niyama_bre_match(nfa, s)              → 0 | 1
niyama_bre_search(nfa, s)             → offset | -1
niyama_bre_search_at(nfa, s, len, from) → offset | -1
niyama_bre_group_start(nfa, n)        → offset | -1
niyama_bre_group_end(nfa, n)          → offset | -1
niyama_bre_last_error()               → BRE_E_*
```

#### `niyama_re2_*` (re2 — RE2 flavor, Pike NFA, linear-time)

```
niyama_re2_compile(pat)               → nfa | 0
niyama_re2_match(nfa, s)              → 0 | 1
niyama_re2_search(nfa, s)             → offset | -1
niyama_re2_search_at(nfa, s, len, from) → offset | -1
niyama_re2_group_start(nfa, n)        → offset | -1
niyama_re2_group_end(nfa, n)          → offset | -1
niyama_re2_group_by_name(nfa, name)   → group_idx | -1
niyama_re2_last_error()               → RE2_E_*
```

#### `niyama_pcre_*` (pcre — Perl-compatible, backtracking)

```
niyama_pcre_compile(pat)              → nfa | 0
niyama_pcre_match(nfa, s)             → 0 | 1
niyama_pcre_search(nfa, s)            → offset | -1
niyama_pcre_search_at(nfa, s, len, from) → offset | -1
niyama_pcre_group_start(nfa, n)       → offset | -1
niyama_pcre_group_end(nfa, n)         → offset | -1
niyama_pcre_group_by_name(nfa, name)  → group_idx | -1
niyama_pcre_last_error()              → PCRE_E_*
niyama_pcre_set_step_limit(n)         → 0
niyama_pcre_last_step_count()         → step_count
niyama_pcre_last_callout()            → callout_num | -1
```

#### `niyama_fuzzy_*` (fuzzy — Levenshtein DP)

```
niyama_fuzzy_compile(pat)             → handle | 0
niyama_fuzzy_compile_opts(pat, max_edits, flags) → handle | 0
niyama_fuzzy_match(h, s)              → 0 | 1
niyama_fuzzy_search(h, s)             → start_offset | -1
niyama_fuzzy_search_prefix(h, s)      → 0 | 1
niyama_fuzzy_distance(h, s)           → distance
niyama_fuzzy_last_distance()          → distance
niyama_fuzzy_last_error()             → FUZZY_E_*
```

#### `niyama_vim_*` (vim — vim/cyim flavor, Pike NFA)

```
niyama_vim_compile(pat)               → nfa | 0
niyama_vim_compile_opts(pat, mode)    → nfa | 0
niyama_vim_match(nfa, s)              → 0 | 1
niyama_vim_search(nfa, s)             → offset | -1
niyama_vim_search_at(nfa, s, len, from) → offset | -1
niyama_vim_group_start(nfa, n)        → offset | -1
niyama_vim_group_end(nfa, n)          → offset | -1
niyama_vim_last_error()               → VIM_E_*
```

### Frozen error codes (numeric values immutable)

| Engine | Code | Value | Status |
|--------|------|-------|--------|
| bre | `BRE_E_OK` | 0 | live |
| bre | `BRE_E_SYNTAX` | 1 | live |
| bre | `BRE_E_BACKREF_UNSUPPORTED` | 2 | live (per ADR 0009) |
| bre | `BRE_E_TOO_LARGE` | 3 | live |
| bre | `BRE_E_BAD_ANCHOR` | 4 | live |
| re2 | `RE2_E_OK` | 0 | live |
| re2 | `RE2_E_SYNTAX` | 1 | live |
| re2 | `RE2_E_BACKREF_UNSUPPORTED` | 2 | live (structural — ADR 0003) |
| re2 | `RE2_E_LOOKAROUND_UNSUPPORTED` | 3 | live |
| re2 | `RE2_E_ATOMIC_UNSUPPORTED` | 4 | live |
| re2 | `RE2_E_RECURSION_UNSUPPORTED` | 5 | live |
| re2 | `RE2_E_TOO_LARGE` | 6 | live |
| re2 | `RE2_E_DUPLICATE_NAME` | 7 | live |
| re2 | `RE2_E_BAD_PROPERTY` | 8 | live |
| pcre | `PCRE_E_OK` | 0 | live |
| pcre | `PCRE_E_SYNTAX` | 1 | live |
| pcre | `PCRE_E_LOOKBEHIND_UNSUPPORTED` | 2 | **reserved** (no longer emitted; v0.7.0 narrowed) |
| pcre | `PCRE_E_UNICODE_PROP_UNSUPPORTED` | 3 | **narrowed** (only inside `[...]` per v0.8.0) |
| pcre | `PCRE_E_RECURSION_UNSUPPORTED` | 4 | **reserved** (no longer emitted; v0.8.0 added recursion) |
| pcre | `PCRE_E_CONDITIONAL_UNSUPPORTED` | 5 | **reserved** (no longer emitted; v0.7.0 added conditionals) |
| pcre | `PCRE_E_TOO_LARGE` | 6 | live |
| pcre | `PCRE_E_DUPLICATE_NAME` | 7 | live |
| pcre | `PCRE_E_BAD_CONDITION` | 8 | live |
| pcre | `PCRE_E_BAD_PROPERTY` | 9 | live |
| pcre | `PCRE_E_LOOKBEHIND_VARWIDTH` | 10 | live |
| pcre | `PCRE_E_BAD_RECURSION_REF` | 11 | live |
| fuzzy | `FUZZY_E_OK` | 0 | live |
| fuzzy | `FUZZY_E_PATTERN_TOO_LONG` | 1 | live |
| fuzzy | `FUZZY_E_INVALID_THRESHOLD` | 2 | live |
| fuzzy | `FUZZY_E_NFD_OVERFLOW` | 3 | live |
| vim | `VIM_E_OK` | 0 | live |
| vim | `VIM_E_SYNTAX` | 1 | live |
| vim | `VIM_E_BACKREF_UNSUPPORTED` | 2 | live (per ADR 0009 — vim post-fold revisit only) |
| vim | `VIM_E_INVALID_MODE` | 3 | live |
| vim | `VIM_E_TOO_LARGE` | 4 | live |
| vim | `VIM_E_BAD_PROPERTY` | 5 | live |

**Reserved-but-unused** slots are part of the frozen surface — they
keep their numeric values forever, even though the engine no longer
emits them. Future code reading old NFA artifacts (e.g. for
diagnostics) sees unchanged numbers.

### Frozen capacity limits

| Limit | Value | Engines |
|-------|-------|---------|
| `MAX_INSTRS` | 4096 | bre, re2, pcre, vim |
| `MAX_CLASSES` | 64 | bre, re2, pcre, vim |
| `MAX_SAVES` | 20 (= 10 groups × 2) | bre, re2, pcre, vim |
| `MAX_NAMES` | 9 | re2, pcre |
| `NAME_SLOT_SIZE` | 40 (32-byte NUL-padded name + 8-byte group_idx) | re2, pcre |
| `NAME_MAX_LEN` | 31 (excluding NUL terminator) | re2, pcre |
| `FUZZY_MAX_PAT_LEN` | 256 | fuzzy |
| `FUZZY_MAX_TEXT_LEN` | 4096 | fuzzy |
| `FUZZY_DEFAULT_K` | 2 | fuzzy |
| pcre step-limit default | 1_000_000 | pcre (configurable) |
| pcre depth-limit | 256 | pcre (not configurable in v1.0) |

Patterns hitting any limit error cleanly with `*_E_TOO_LARGE` (or
the appropriate engine code). v0.9.0 boundary tests (per-engine)
verify the contract.

### Frozen semantic invariants

1. **re2 cannot accept backref.** Compile-time rejection is
   structural per ADR 0003. The linear-time guarantee depends on
   it. This invariant is permanent through v1.0 and beyond.
2. **bre cannot accept backref.** Per ADR 0009, permanent for v1.0
   and post-fold.
3. **vim cannot accept backref through v1.0.** Post-fold revisit
   open via cyrius stdlib per ADR 0009; niyama-side ABI is locked.
4. **pcre is the *only* backtracking engine.** Step-limit + depth
   bound the worst case.
5. **Pike NFA engines (bre, re2, vim) are linear-time-guaranteed**
   for accepted patterns. No catastrophic-backtracking path exists.
6. **fuzzy is byte-Levenshtein** (with optional NFD normalization
   pre-processing via `FUZZY_FLAG_UNICODE_NFD`).
7. **Greedy quantifiers by default.** Lazy via `?` suffix
   (`*?` `+?` `??` `{n,m}?`) where the engine supports it.
8. **`^` and `$` are strict-by-default in re2 and pcre** (per
   v0.7.0 ADR 0007). `(?m)` opts into multiline. bre and vim keep
   their flavor-traditional loose semantics.

### Frozen opcode IDs

Internal but part of the dist artifact ABI (consumers of
`dist/niyama.cyr` see the encoded NFAs). Listed in source as
`<ENGINE>_OP_*` constants. Frozen values:

- bre: 0–11
- re2: 0–19 (v0.8.0 added UPROP=16, NUPROP=17, UCHAR=18, UCHAR_CI=19)
- pcre: 0–30 (v0.8.0 added UPROP=24, NUPROP=25, UCHAR_CI=26,
  LOOKBEHIND=28, NLOOKBEHIND=29, RECURSE=30; opcode 27 unused)
- vim: 0–14 (v0.8.0 added UPROP=12, NUPROP=13, UCHAR=14)
- fuzzy: no opcodes (DP-based)

Reserved-and-unused opcode slots (e.g. pcre 27) keep their
slots; new opcodes go at next-available indices.

## Post-freeze evolution model

The frozen surface above ships as v1.0. Three release paths exist
post-v1.0:

### v1.x niyama patch releases

Bug fixes only. **Cannot** add error codes, opcodes, or API entry
points. **Cannot** change behavior. Bug-fix releases ship to
consumers who haven't picked up the cyrius stdlib fold yet.

### Post-fold via cyrius stdlib

Once cyrius stdlib vendors `lib/niyama.cyr` (per ADR 0001's
sandhi-pattern fold lifecycle), the stdlib version *can* extend the
surface under additive-only rules:

- **New error codes** at next-available numeric slot per engine.
  Existing values immutable.
- **New opcodes** at next-available index per engine. Existing
  values immutable.
- **New API entry points** as pure additions (e.g. a
  hypothetical `niyama_fuzzy_search_at` per the v0.9.0 review's
  A1 finding). Existing signatures immutable.
- **Behavior changes** require an ADR and a major-version bump on
  the cyrius-stdlib-side `lib/niyama.cyr`. niyama-the-repo's
  v1.x doesn't pick them up.

Example pinned candidates for post-fold extension (per ADR 0009 +
v0.9.0 review):

- vim backref `\1`-`\9` — pinned in ADR 0009 with containment
  design.
- fuzzy `niyama_fuzzy_search_at` — A1 finding from review.
- Long property names + Unicode scripts — G3 from review.

### Major-version v2.0 (speculative)

If a consumer ecosystem ever needs surface changes that exceed
additive-only rules, niyama v2.0 is the vehicle. v2.0 would be a
new repository or a new toplevel namespace within niyama; the
frozen v1.0 surface remains accessible to consumers that pin to
v1.x.

## Consequences

- **Positive** — `dist/niyama.cyr` is a stable vendor target.
  cyrius stdlib can fold without versioning drama. Consumers
  pinning to `niyama_<engine>_*` get the API contract for v1.x.
- **Positive** — Security audit cadence simplifies. Any future
  audit checks "did the surface change?" — if no, the v0.9.0 audit
  generally still applies.
- **Positive** — Fork-and-extend is structurally possible. A
  consumer needing post-fold features that haven't landed can
  vendor `dist/niyama.cyr` directly and apply local extensions
  under additive-only rules.
- **Negative** — Mistakes in the surface are now permanent (or
  post-fold-only fixable). The v0.9.0 hardening pass exists
  exactly to surface mistakes before this lock; the audit found
  none CRITICAL/HIGH.
- **Negative** — Some asymmetries are now permanent: fuzzy lacks
  `_search_at` (A1), `compile_opts` is only on fuzzy/vim (A2),
  pcre is the only engine with `set_step_limit`. These are
  flavor-driven and documented in `state.md`'s engine-selection
  rubric.
- **Neutral** — Post-fold extensions land in cyrius stdlib's
  vendored copy, not in niyama's repo. niyama enters maintenance
  mode at v1.0; releases beyond v1.x are bug-fix only.

## Alternatives considered

- **No surface freeze; let v1.0 be a flag day.** Rejected — the
  fold needs a stable target. Without freeze, every cyrius release
  vendoring `lib/niyama.cyr` would risk drift.
- **Wider surface freeze including internal helpers.** Rejected —
  internal `_<engine>_*` symbols are implementation details.
  Freezing them prevents post-fold optimization (e.g. the B4
  saves-pool refactor). Public surface only is the right scope.
- **Narrower surface freeze excluding error codes.** Rejected —
  error codes ARE consumer-visible (via `niyama_*_last_error()`).
  cyim's flavor-selection logic switches on them. Freezing them
  is mandatory.
- **Defer freeze to v1.0 itself.** Rejected — that means v1.0
  ships under freeze for the first time, with no pre-release
  shake-out window. v0.9.0 ship + v1.0 ship = two opportunities
  to catch a freeze mistake before it's permanent.

## Open questions (resolved at v0.9.0 ship)

- ~~Should `compile_opts` be added to bre/re2/pcre for symmetry?~~
  Rejected per A2 review finding. Frozen as-is.
- ~~Should fuzzy gain `_search_at`?~~ Rejected for v1.0 per A1.
  Pinned for post-fold cyrius-stdlib extension.
- ~~Should pcre's depth limit be configurable?~~ Rejected — 256 is
  conservative for niyama's use cases; making it configurable
  trades attack-surface for one consumer's edge case.
