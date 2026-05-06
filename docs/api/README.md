# niyama Public API Reference

Consolidated public-surface reference for niyama v1.0. **Frozen
per ADR 0010 — Surface freeze.** Post-v1.0 changes to anything in
this file are limited to additive extensions in cyrius stdlib's
vendored `lib/niyama.cyr` (post-fold); niyama-the-repo's v1.x
patches cannot extend the surface.

For *why* the API is shaped this way, see the per-engine ADRs
(0002-0006) and ADR 0010. For implementation details (matcher
loops, opcode encoding, internal helpers), see `docs/architecture/`.

## Conventions

- **Return value `0`** from a `_compile` function = compile failure.
  Read `niyama_<engine>_last_error()` for the error code.
- **Return value `-1`** from `_search` / `_search_at` / `_group_*`
  = no match / unparticipated group.
- **Return value `0` or `1`** from `_match` / `_search_prefix` =
  no match / match (boolean).
- All position values are **byte offsets** into the input string,
  not codepoint indices.
- All public functions take inputs as **NUL-terminated `cstring`**
  pointers; inputs are read but never written.

## niyama_bre — POSIX BRE (Basic Regular Expressions)

Pike NFA, linear-time. POSIX BRE flavor minus backref (per ADR 0002
+ ADR 0009).

| Function | Returns | Notes |
|----------|---------|-------|
| `niyama_bre_compile(pat)` | nfa pointer or `0` | error via `_last_error()` |
| `niyama_bre_match(nfa, s)` | `0` / `1` | match search starting at pos 0 |
| `niyama_bre_search(nfa, s)` | offset or `-1` | first match offset in `s` |
| `niyama_bre_search_at(nfa, s, len, from)` | offset or `-1` | first match starting at `from` |
| `niyama_bre_group_start(nfa, n)` | offset or `-1` | group n start (group 0 = whole match) |
| `niyama_bre_group_end(nfa, n)` | offset or `-1` | group n end |
| `niyama_bre_last_error()` | `BRE_E_*` | last compile error |

**Error codes**: `BRE_E_OK = 0`, `BRE_E_SYNTAX = 1`,
`BRE_E_BACKREF_UNSUPPORTED = 2` (permanent per ADR 0009),
`BRE_E_TOO_LARGE = 3`, `BRE_E_BAD_ANCHOR = 4`.

**Limits**: `BRE_MAX_INSTRS = 4096`, `BRE_MAX_CLASSES = 64`,
`BRE_MAX_SAVES = 20` (10 groups × 2). Patterns at boundary error
with `BRE_E_TOO_LARGE` or `BRE_E_SYNTAX`.

## niyama_re2 — RE2 flavor (linear-time-guaranteed)

Pike NFA, codepoint-stepped. ERE flavor + linear-time guarantee on
untrusted input (per ADR 0003).

| Function | Returns | Notes |
|----------|---------|-------|
| `niyama_re2_compile(pat)` | nfa pointer or `0` | |
| `niyama_re2_match(nfa, s)` | `0` / `1` | |
| `niyama_re2_search(nfa, s)` | offset or `-1` | |
| `niyama_re2_search_at(nfa, s, len, from)` | offset or `-1` | |
| `niyama_re2_group_start(nfa, n)` | offset or `-1` | |
| `niyama_re2_group_end(nfa, n)` | offset or `-1` | |
| `niyama_re2_group_by_name(nfa, name)` | group_idx or `-1` | for `(?<name>...)` / `(?P<name>...)` |
| `niyama_re2_last_error()` | `RE2_E_*` | |

**Error codes**: `RE2_E_OK = 0`, `RE2_E_SYNTAX = 1`,
`RE2_E_BACKREF_UNSUPPORTED = 2` (structural — guarantees linear time),
`RE2_E_LOOKAROUND_UNSUPPORTED = 3`, `RE2_E_ATOMIC_UNSUPPORTED = 4`,
`RE2_E_RECURSION_UNSUPPORTED = 5`, `RE2_E_TOO_LARGE = 6`,
`RE2_E_DUPLICATE_NAME = 7`, `RE2_E_BAD_PROPERTY = 8`.

**Limits**: same as bre, plus `RE2_MAX_NAMES = 9`,
`RE2_NAME_MAX_LEN = 31`.

## niyama_pcre — Perl-compatible (backtracking)

Backtracking matcher. Step-limit + depth-limit bounded (per ADR 0004).

| Function | Returns | Notes |
|----------|---------|-------|
| `niyama_pcre_compile(pat)` | nfa pointer or `0` | |
| `niyama_pcre_match(nfa, s)` | `0` / `1` | |
| `niyama_pcre_search(nfa, s)` | offset or `-1` | |
| `niyama_pcre_search_at(nfa, s, len, from)` | offset or `-1` | |
| `niyama_pcre_group_start(nfa, n)` | offset or `-1` | |
| `niyama_pcre_group_end(nfa, n)` | offset or `-1` | |
| `niyama_pcre_group_by_name(nfa, name)` | group_idx or `-1` | |
| `niyama_pcre_last_error()` | `PCRE_E_*` | |
| `niyama_pcre_set_step_limit(n)` | `0` | default 1_000_000; per-process |
| `niyama_pcre_last_step_count()` | step count | observability |
| `niyama_pcre_last_callout()` | callout num or `-1` | for `(?C<num>)` |

**Error codes**: `PCRE_E_OK = 0`, `PCRE_E_SYNTAX = 1`. Slots 2-5
are reserved-but-unused (former `LOOKBEHIND_UNSUPPORTED`,
`UNICODE_PROP_UNSUPPORTED` (now narrowed to inside `[...]` only),
`RECURSION_UNSUPPORTED`, `CONDITIONAL_UNSUPPORTED`); slot 3 still
emitted for `[\p{L}]`-style char-class composition. Live codes:
`PCRE_E_TOO_LARGE = 6`, `PCRE_E_DUPLICATE_NAME = 7`,
`PCRE_E_BAD_CONDITION = 8`, `PCRE_E_BAD_PROPERTY = 9`,
`PCRE_E_LOOKBEHIND_VARWIDTH = 10`, `PCRE_E_BAD_RECURSION_REF = 11`.

**Limits**: same as bre/re2, plus `PCRE_MAX_NAMES = 9`,
`PCRE_NAME_MAX_LEN = 31`. Default `_pcre_step_limit = 1_000_000`
(configurable); `_pcre_depth_limit = 256` (not configurable).

## niyama_fuzzy — Levenshtein edit-distance

Wagner-Fischer DP, two-row optimized. Not a regex (per ADR 0005).

| Function | Returns | Notes |
|----------|---------|-------|
| `niyama_fuzzy_compile(pat)` | handle or `0` | default k=2, no flags |
| `niyama_fuzzy_compile_opts(pat, max_edits, flags)` | handle or `0` | |
| `niyama_fuzzy_match(h, s)` | `0` / `1` | distance ≤ max_edits |
| `niyama_fuzzy_search(h, s)` | start_offset or `-1` | substring fuzzy |
| `niyama_fuzzy_search_prefix(h, s)` | `0` / `1` | prefix fuzzy |
| `niyama_fuzzy_distance(h, s)` | distance | full-string Levenshtein |
| `niyama_fuzzy_last_distance()` | distance | from most recent call |
| `niyama_fuzzy_last_error()` | `FUZZY_E_*` | |

**Flags**: `FUZZY_FLAG_CASE_INSENSITIVE = 1` (ASCII fold),
`FUZZY_FLAG_UNICODE_NFD = 2` (NFD-normalize pat + input via stdlib
`str_normalize`).

**Error codes**: `FUZZY_E_OK = 0`, `FUZZY_E_PATTERN_TOO_LONG = 1`,
`FUZZY_E_INVALID_THRESHOLD = 2`, `FUZZY_E_NFD_OVERFLOW = 3`.

**Limits**: `FUZZY_MAX_PAT_LEN = 256`, `FUZZY_MAX_TEXT_LEN = 4096`,
`FUZZY_DEFAULT_K = 2`.

**Note**: no `_search_at` — substring-search-from-offset requires
slicing the input externally (post-fold candidate).

## niyama_vim — vim/cyim flavor

Pike NFA, codepoint-stepped. vim/cyim flavor, 4 magicness modes
(per ADR 0006).

| Function | Returns | Notes |
|----------|---------|-------|
| `niyama_vim_compile(pat)` | nfa pointer or `0` | default mode = MAGIC |
| `niyama_vim_compile_opts(pat, mode)` | nfa pointer or `0` | mode ∈ {VERY_NOMAGIC, NOMAGIC, MAGIC, VERY_MAGIC} |
| `niyama_vim_match(nfa, s)` | `0` / `1` | |
| `niyama_vim_search(nfa, s)` | offset or `-1` | |
| `niyama_vim_search_at(nfa, s, len, from)` | offset or `-1` | |
| `niyama_vim_group_start(nfa, n)` | offset or `-1` | |
| `niyama_vim_group_end(nfa, n)` | offset or `-1` | |
| `niyama_vim_last_error()` | `VIM_E_*` | |

**Modes**: `VIM_MODE_VERY_NOMAGIC = 0`, `VIM_MODE_NOMAGIC = 1`,
`VIM_MODE_MAGIC = 2` (default), `VIM_MODE_VERY_MAGIC = 3`.

**Error codes**: `VIM_E_OK = 0`, `VIM_E_SYNTAX = 1`,
`VIM_E_BACKREF_UNSUPPORTED = 2` (per ADR 0009 — post-fold revisit
candidate via cyrius stdlib), `VIM_E_INVALID_MODE = 3`,
`VIM_E_TOO_LARGE = 4`, `VIM_E_BAD_PROPERTY = 5`.

**Limits**: same as bre.

## Cross-engine: shared modules used at runtime

The following functions live in shared modules. Consumers vendoring
`dist/niyama.cyr` get them transparently. Listed here because some
appear at the public surface as required dependencies.

### From `lib/unicode/` (cyrius stdlib ≥ 5.8.65)

- `unicode_category(cp)` — used by re2/pcre/vim's `\p{NAME}` /
  `\P{NAME}` matcher.
- `unicode_to_lower(cp)` — used by re2/pcre's `(?i)` Unicode
  case-fold for multi-byte literals.
- `str_normalize(s, form)` — used by fuzzy's
  `FUZZY_FLAG_UNICODE_NFD`.
- `_uc_decode_utf8(src, off, src_len, cp_out)` — used by re2/pcre/vim
  matchers for codepoint extraction.

Consumers must include `lib/unicode/categories.cyr`,
`lib/unicode/casefold.cyr`, `lib/unicode/normalize.cyr` (transitively
including `_decode.cyr`) and `lib/str.cyr` alongside niyama.

### From `src/posix_classes.cyr` (niyama-internal shared)

- `_posix_match_class_name(pat, pos, pat_len)` — recognizes
  `[:name:]` keywords for the 12 POSIX bracket classes. Used by all
  four regex engines.
- `_posix_apply_filler(class_base, ci, category)` — fills the
  byte-bitmap for a recognized POSIX class.

### From `src/unicode_props.cyr` (niyama-internal shared)

- `_uprops_parse_class(pat, pos, pat_len)` — recognizes `\p{NAME}`
  / `\P{NAME}` short names; returns 30-bit GeneralCategory mask.
- `_uprops_test(cp, mask)` — runtime category-membership test.

These two niyama-internal modules are part of `dist/niyama.cyr`'s
single-include bundle.

## Engine-selection guidance

See [`docs/development/state.md` § Engine-selection guidance for
consumers](../development/state.md#engine-selection-guidance-for-consumers)
for the consumer-facing flavor-picking rubric.

## Versioning post-v1.0

Per ADR 0010 § Post-freeze evolution model:

- **niyama v1.x patches** — bug fixes only. No surface changes.
- **Post-fold via cyrius stdlib** `lib/niyama.cyr` — additive
  extensions only. New error codes at next-available numeric slot,
  new opcodes at next index, new API as pure additions.
- **niyama v2.0** — hypothetical; for non-additive changes that
  exceed the post-fold model. Would be a new namespace; v1.x
  consumers are not affected.
