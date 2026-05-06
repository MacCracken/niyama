# 0008 — Unicode-stdlib pivot + M4.5b/c reshape

**Status**: Accepted
**Date**: 2026-05-05

## Context

Cyrius 5.8.65 ships a Unicode stdlib at `lib/unicode/`:

- `lib/unicode/categories.cyr` — `unicode_category(cp)` returning a
  GeneralCategory leaf (0..29) backed by a binary-search range index
  over Unicode 17.0.0 DerivedGeneralCategory data (4144 ranges,
  ~12 iterations max).
- `lib/unicode/normalize.cyr` — `str_normalize(s, form)` for
  NFC / NFD / NFKC / NFKD, plus low-level `unicode_canonical_decomp`,
  `unicode_canonical_combining_class`, `unicode_compose`.
- `lib/unicode/casefold.cyr` — simple `unicode_to_lower / _upper /
  _title` plus full-fold `unicode_fold(cp, out_buf, cap)` per UCD
  CaseFolding.txt status C+F (handles ß → "ss", İ → "i̇").
- `lib/unicode/_decode.cyr` — shared UTF-8 codec
  `_uc_emit_utf8` / `_uc_decode_utf8` and hex-pair decoders for the
  baked UCD data tables.

ADR 0007's M4.5c plan (v0.9.0) was carved around building this exact
infrastructure ourselves: "a single ~25KB infrastructure addition" —
a new shared `src/unicode.cyr` housing decomposition + property
tables that re2, pcre, fuzzy, and vim would all hang off of. That
carve was load-bearing for ADR 0007's three-bucket sequencing —
the reason `\p{L}` and `FUZZY_FLAG_UNICODE_NFD` were grouped together
in v0.9.0 is that they shared the table-building cost, not the
feature work itself.

That premise is gone. The table exists; consumers just call into it.
What remains in M4.5c is per-engine wiring — picking opcodes,
threading parser support, writing tests. Each piece is small and
independent.

A second observation: `unicode_fold` makes a ~10-line `(?i)` upgrade
viable in re2 and pcre. v0.7.0's inline-flag work (ADR 0007) shipped
ASCII-only case folding because the alternative was either bake our
own fold table or block the v0.7.0 carve on infrastructure that
hadn't been planned. With stdlib `unicode_fold` available, the cost
of the upgrade is trivial; the only question is sequencing.

## Decision

**Stdlib unicode replaces the planned `src/unicode.cyr` module
entirely. All remaining M4.5 feature work consolidates into a single
v0.8.0 = M4.5 completion release.**

### Sequencing history (for the next reader)

This ADR went through two revisions before settling. The original
draft proposed a four-release ladder (v0.8.0 expanded with `\p{L}`,
new v0.8.1 for `(?i)` Unicode upgrade, v0.8.2 decision-gated for
backref, v0.9.0 slimmed for NFD + vim refactor). User direction
during planning collapsed that to **two**: v0.8.0 carrying every
non-decision-gated piece, v0.8.1 pinned but only releasing if the
backref decision comes back "yes". The reasoning: don't fragment a
coherent milestone (M4.5 completion) into a dust of patch-tagged
releases just because the carve was once useful at planning time.
Both shapes were technically valid; the consolidated shape is what
ships. The four-release intermediate is preserved here only so a
future reader understands why early commits / drafts may reference
v0.8.1 as "(?i) upgrade" or v0.9.0 as "NFD + vim refactor."

### v0.8.0 (M4.5b) — M4.5 completion release

Carries every M4.5 deferral that isn't decision-gated:

- `niyama_pcre` lookbehind `(?<=...)` `(?<!...)` — fixed-width
  compile-time width analysis (PCRE2 10.43 model). Variable-width
  lookbehind stays out of scope.
- `niyama_pcre` recursion `(?R)`, `(?P>name)`, `(?N)` — subroutine
  call into the same compiled NFA.
- `niyama_fuzzy` exact start-position recovery in
  `niyama_fuzzy_search` — reverse-DP pass to recover the precise
  start offset of the best fuzzy substring match.
- **`\p{L}` Unicode property classes for `niyama_re2`,
  `niyama_pcre`, and `niyama_vim`** — wired onto stdlib
  `unicode_category()`. One shared parser helper (likely
  `src/unicode_props.cyr`, parallel to `src/posix_classes.cyr`)
  for parsing `\p{Name}` / `\P{Name}` and resolving leaf vs.
  aggregate categories.
- **`(?i)` Unicode case-fold upgrade for `niyama_re2` and
  `niyama_pcre`** — replaces ASCII `cyr_to_lower` with stdlib
  `unicode_fold(cp, ...)`. Handles ß↔SS, İ↔i̇, Greek σ↔ς, Cyrillic.
  Same opcodes (`*_OP_CHAR_CI`); matcher comparison logic swap only.
  niyama_vim excluded — vim's case-insensitive matching is `\c`/`\C`
  toggles, not `(?i)`, an ADR 0006 escape-menagerie item.
- **`niyama_fuzzy` Unicode NFD normalization** —
  `FUZZY_FLAG_UNICODE_NFD` calls stdlib `str_normalize(s, NFD)` on
  both pattern and input before the Levenshtein DP runs.
- **vim → posix_classes refactor** — fold `niyama_vim`'s in-engine
  POSIX bracket-class code onto `src/posix_classes.cyr` (deliberate
  duplication left in v0.7.0 to keep risk low). Zero behavior
  change; cleans up before M5 freeze.

The new shared `src/unicode.cyr` module is **deleted from the plan**.
A small `src/unicode_props.cyr` (parser helper for `\p{...}` syntax,
not a data table) lands in v0.8.0 instead — orders of magnitude
smaller than the planned 25 KB.

### v0.8.1 (M4.5b.1) — collapsed at v0.8.0 ship time

Originally pinned as the decision-gated slot for bre/vim backref.
At v0.8.0 ship the user direction landed: **slot collapses (no
release); review pin moves to v0.9.0 with broader scope.** The
ladder rule's "pinning ≠ shipping" branch fired exactly as intended;
no v0.8.1 tag exists.

### v0.9.0 — bre / vim backref review (decision + exposure)

Pinned for a fuller review pass than v0.8.1 contemplated. Question
isn't just "implement yes/no" — it's the **exposure surface**: ABI
shape (kernel vs. per-engine), error-code reuse vs. new slots
(BRE_E_BACKREF_UNSUPPORTED = 2 and VIM_E_BACKREF_UNSUPPORTED = 2
already burn the obvious slots), re2's no-backref linear-time
guarantee preservation strategy, and consumer-side impact (cyim,
owl, agnoshi, daimon).

Three outcome paths:

- **Implement at v0.9.0** with documented exposure (new ADR records
  the kernel/policy split + the consumer-facing API).
- **Document permanently out-of-scope for v1.0** with a fold-time
  ADR explaining why.
- **Further-defer** with refined scope (slot moves to v0.9.x or
  post-fold).

The review itself is the v0.9.0 deliverable; whether code lands
depends on what the review concludes. Pin exists so "what happened
to bre/vim backref?" has a roadmap answer — visibility, not
commitment to ship.

### v0.8.x ladder rule (companion policy)

Adopted alongside this reshape: **any feature deferred out of v0.8.0
during implementation lands at a pinned v0.8.x slot** (v0.8.1,
v0.8.2, ...) with a one-line scope note in `roadmap.md`. No floating
"post-v1.0 unless asked" or "vN.0 maybe" deferrals — every slipped
item gets a version number and stays visible.

**Pinning ≠ committing to a release.** Slots that don't warrant a
tag (e.g. backref says "no") collapse to a CHANGELOG note without
a version number. The pin is about visibility; the release decision
is downstream. This is what keeps the ladder from fragmenting M4.5
into a dust of patch-tagged releases — the carve we just collapsed
in the consolidation above.

The first concrete consequence: the bre / vim backref re-decision
question is pinned at **v0.8.1** as a decision-gated slot.

The ladder rule isn't strictly part of the unicode-stdlib pivot
that motivates this ADR, but it lands in the same docs-update sweep
because the reshape itself created the right moment to fix the
"floating deferrals" anti-pattern. CLAUDE.md's "do not unilaterally
defer / descope" rule already covers Claude's behavior; the ladder
covers the *artifact* — what's written down.

### ADR 0007 status

ADR 0007 is **not superseded** — it correctly described the v0.7.0
carve as it was implemented. ADR 0008 supersedes only the M4.5b/c
sequencing notes in ADR 0007's "Deferrals confirmed for v0.8.0 /
v0.9.0" section, and adds the v0.8.x ladder policy on top.

## Consequences

- **Positive** — ~25 KB of UCD data is no longer niyama's
  responsibility. Cyrius stdlib owns the version cadence (Unicode
  17.0.0 today; future updates land via toolchain bump). niyama's
  fold-ready artifact (`dist/niyama.cyr`) shrinks accordingly.
- **Positive** — `\p{L}` arrives one release earlier than originally
  planned. agnoshi / daimon consumers needing Unicode-aware patterns
  unblock at v0.8.0 instead of v0.9.0.
- **Positive** — `(?i)` semantics catch up to spec at v0.8.1. The
  v0.7.0 inline-flag work shipped a known-incomplete fold; v0.8.1
  closes the gap before M5 freeze.
- **Negative** — niyama now depends on `lib/unicode/`. The dep gate
  is `cyrius >= 5.8.65` (categories landed at .49, casefold at .50,
  normalize at .51, NFKC/NFKD at .60, codec lift at .55). Older
  toolchains can't build niyama post-v0.8.0. Acceptable: the
  toolchain pin in `cyrius.cyml` is the contract, and AGNOS-lineage
  consumers track the same pin.
- **Negative** — three sub-releases (v0.8.0, v0.8.1, v0.9.0) means
  three CHANGELOG entries, three state.md refreshes, three bench
  passes. Slightly more release overhead than the original two-slot
  v0.8.0 / v0.9.0 plan. The trade is a smaller per-release surface
  per slot, which is actually easier to review.
- **Neutral** — pulling `\p{L}` into v0.8.0 expands that release's
  scope from 3 items to 4. Within ADR 0007's "carve to keep things
  reviewable" spirit; the extra item is a wiring-only change with
  no new infrastructure.
- **Neutral** — niyama's `lib/unicode/_decode.cyr` UTF-8 codec
  helpers (`_uc_emit_utf8`, `_uc_decode_utf8`) are now available to
  the engines too. The four engines that hand-rolled UTF-8 decode
  in their own bodies (vim, fuzzy at minimum) could deduplicate onto
  the stdlib helpers in M5 cleanup. Not in scope here; flagged for
  M5.

## Alternatives considered

- **Keep ADR 0007's plan unchanged — build `src/unicode.cyr`
  ourselves.** Rejected: re-implementing 25 KB of UCD data when the
  toolchain already ships an audited, version-tracked copy is
  duplicate work for zero correctness benefit. The "vendored
  forever" argument that justified custom tables in some other
  contexts doesn't apply here — stdlib `lib/unicode/` is byte-for-byte
  vendored into `lib/` by `cyrius update` exactly like every other
  stdlib module.
- **Pull `\p{L}` AND NFD into v0.8.0, leave only the vim refactor
  for v0.9.0.** Rejected: NFD wiring touches the fuzzy DP setup,
  not just a parser helper — it's a real matcher change. Keeping it
  in v0.9.0 alongside the vim refactor (also a refactor pass)
  matches v0.9.0's character as a cleanup release.
- **Defer `(?i)` upgrade to v0.9.0 alongside NFD.** Rejected:
  `(?i)` is independent work — different engines (re2 + pcre, no
  fuzzy/vim involvement), different mechanism (matcher comparison
  swap, not normalization). Its own patch release (v0.8.1) gives it
  a clean changelog, a focused review surface, and the option to
  ship before v0.9.0's Unicode infra work is fully baked.
- **Ship `(?i)` upgrade as part of v0.8.0.** Rejected: v0.8.0 is
  already four items including `\p{L}`. Adding a fifth — even small —
  pushes the carve past the "reviewable in one session" line that
  ADR 0007 set as the v0.7.0 / v0.8.0 / v0.9.0 sizing principle.
- **Lift niyama-side UTF-8 helpers onto stdlib `_uc_*` codec now.**
  Rejected: in-scope cleanup but not in-scope feature. The engines
  work; M5 is the right window per first-party-standards' "audit
  before freeze" cadence.
