# niyama — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.8.0** — M4.5 completion release shipped 2026-05-05. Per ADR 0008
(Unicode-stdlib pivot + sub-release collapse), every remaining M4.5
deferral lands in one bundle: `\p{L}` for re2/pcre/vim, multi-byte
literal pattern char support, `(?i)` Unicode case-fold upgrade,
fuzzy NFD via stdlib `str_normalize`, fuzzy exact-start recovery,
vim → posix_classes refactor, pcre lookbehind (fixed-width), pcre
recursion `(?R)` / `(?P>NAME)`. All five engines still shipped.
Decision-gated v0.8.1 slot pinned for bre/vim backref.

**v0.8.x ladder rule.** Any feature deferred *out of v0.8.0 during
implementation* gets a pinned v0.8.x slot in `roadmap.md`, not a
floating "post-v1.0 / vN.0 maybe" note. Pins don't force releases —
slots that don't warrant a tag collapse to a CHANGELOG note.
Established post-v0.7.0; prevents silent roadmap shrinkage *and*
release-tag dust.

## Toolchain

- **Cyrius pin**: `5.8.65` (in `cyrius.cyml [package].cyrius`).
  Bumped from `5.8.42` post-v0.7.0 to pull in the stdlib
  `lib/unicode/` tree (categories at .49, casefold at .50, normalize
  at .51, codec lift at .55, NFKC/NFKD at .60). Required floor for
  v0.8.0 onward per ADR 0008.

## Source

- `src/main.cyr` — smoke entry (prints identity banner).
- `src/test.cyr` — top-level test entry per `[build].test`.
- **`src/posix_classes.cyr`** — shared POSIX bracket-class fillers
  + name recognizer (v0.7.0). vim folded onto this module at v0.8.0.
- **`src/unicode_props.cyr`** — shared `\p{NAME}` / `\P{NAME}`
  parser + GeneralCategory bitmask lookup (v0.8.0). Used by
  re2/pcre/vim. Backed by stdlib `lib/unicode/categories.cyr`.
  ~140 lines.
- **`src/bre.cyr`** — POSIX BRE engine (M1, +`\<\>` and POSIX
  classes from v0.7.0). Pike NFA.
- **`src/re2.cyr`** — RE2-flavor linear-time engine (M2, +named
  captures + inline flags from v0.7.0; +`\p{L}` + multi-byte
  literals + `(?i)` Unicode upgrade from v0.8.0). Pike NFA,
  codepoint-stepped matcher loop.
- **`src/pcre.cyr`** — PCRE-light backtracking engine (M3,
  +POSIX classes / inline flags / `\K` / branch-reset / conditional
  / callouts from v0.7.0; +`\p{L}` / `(?i)` Unicode / fixed-width
  lookbehind / recursion from v0.8.0).
- **`src/fuzzy.cyr`** — Levenshtein edit-distance engine (M3.5,
  +Unicode NFD + exact-start recovery from v0.8.0).
- **`src/vim.cyr`** — vim/cyim flavor engine (M4, +`\p{L}` +
  multi-byte literals + posix_classes refactor from v0.8.0). Pike
  NFA, codepoint-stepped matcher loop.

## Engines shipped

| Engine | Status | ABI prefix | Algorithm | Notes |
|---|---|---|---|---|
| **bre** | ✅ v0.7.0 (M1 + v0.7.0 catch-up) | `niyama_bre_*` | Pike NFA | POSIX BRE minus backrefs (per ADR 0002). v0.7.0 adds GNU `\<\>` boundaries + POSIX bracket classes (ADR 0007). |
| **re2** | ✅ v0.8.0 (M2 + M4.5 complete) | `niyama_re2_*` | Pike NFA, codepoint-stepped | ERE + linear-time guarantee at API (per ADR 0003). v0.7.0: named captures, inline flags, strict `^/$/.` defaults. v0.8.0: `\p{L}` Unicode props, multi-byte literal patterns, `(?i)` Unicode case-fold (ADR 0008). |
| **pcre** | ✅ v0.8.0 (M3 + M4.5 complete) | `niyama_pcre_*` | Backtracking | Perl-compat (per ADR 0004). Step-limit + depth bound. v0.7.0: POSIX classes, inline flags, `\K`, branch-reset, conditional, callouts. v0.8.0: `\p{L}`, `(?i)` Unicode, fixed-width lookbehind, recursion `(?R)` / `(?P>NAME)` (ADR 0008). |
| **fuzzy** | ✅ v0.8.0 (M3.5 + M4.5 complete) | `niyama_fuzzy_*` | Levenshtein DP | Edit-distance, three match modes (per ADR 0005). v0.8.0: `FUZZY_FLAG_UNICODE_NFD`, exact start-position recovery via reverse-DP. |
| **vim** | ✅ v0.8.0 (M4 + M4.5 complete) | `niyama_vim_*` | Pike NFA, codepoint-stepped | vim/cyim flavor, 4 magicness modes (per ADR 0006). v0.8.0: `\p{L}` Unicode props, multi-byte literal patterns, POSIX-class code folded onto `src/posix_classes.cyr`. |

## Engine-selection guidance for consumers

| Want | Use |
|---|---|
| `grep -G` / `sed` POSIX BRE compatibility | `bre` |
| ERE + provable linear-time DoS-safety on untrusted input | `re2` |
| Backref, lookahead, named captures, atomic groups | `pcre` |
| Both ERE features AND DoS-safety | `re2` (no backref/lookaround) |
| Both PCRE features AND bounded DoS-safety | `pcre` with `niyama_pcre_set_step_limit()` |
| Typo-tolerant matching, shell completion, fuzzy-name lookup | `fuzzy` |
| vim/cyim-flavor patterns (`\<`, `\>`, `\zs`/`\ze`, magicness) | `vim` |
| Case-insensitive / multi-line / dot-newline matching | `re2` or `pcre` with `(?i)`/`(?m)`/`(?s)` |
| Conditional or branch-reset patterns | `pcre` (`(?(...)...)`, `(?\|...)`) |

## Fold-ready artifact

- `dist/niyama.cyr` — single-include bundle. v0.8.0 prepends both
  shared modules (`src/posix_classes.cyr` then `src/unicode_props.cyr`)
  ahead of the five engine modules. Consumers also need stdlib
  `lib/str.cyr` and `lib/unicode/{categories,casefold,normalize}.cyr`.
  M5 surface freeze ADR will lock its public symbol set.

## Tests

- `tests/niyama.tcyr` — scaffold smoke (2 assertions).
- **`tests/bre.tcyr`** — 107 BRE assertions.
- **`tests/re2.tcyr`** — 147 RE2 assertions (was 101; +37 unicode-prop,
  +5 `(?i)` Unicode, +4 multi-byte literals).
- **`tests/pcre.tcyr`** — 185 PCRE assertions (was 140; +21 unicode-prop,
  +5 `(?i)` Unicode, +12 lookbehind, +12 recursion; -2 deferred-rejection
  tests removed since features now land).
- **`tests/fuzzy.tcyr`** — 53 fuzzy assertions (was 45; +4 NFD,
  +4 exact-start).
- **`tests/vim.tcyr`** — 104 vim assertions (was 88; +13 unicode-prop,
  +3 multi-byte).
- **`tests/{bre,re2,pcre,fuzzy,vim}.bcyr`** — per-engine bench harnesses.
- **`fuzz/{bre,re2,pcre,fuzzy,vim}.fcyr`** — per-engine fuzz harnesses.

Aggregate: `cyrius test` reports **6 files, 598 assertions** all passing.
`cyrius fuzz` reports **5 files, 1660 assertions** all passing.

## Dependencies

Direct (declared in `cyrius.cyml`):

- stdlib — string, fmt, alloc, io, vec, str, syscalls, assert, unicode.
  (`unicode` added post-v0.7.0 alongside the toolchain bump per ADR
  0008 — pulls in the `lib/unicode/{categories,casefold,normalize,
  _decode}.cyr` tree.)

## Consumers

| Consumer | Status | Notes |
|----------|--------|-------|
| [cyim](https://github.com/MacCracken/cyim) | Parser-side ready (1.2.0) | All five flavors (`bre`, `re2`, `pcre`, `fuzzy`, `vim`) ready to wire. cyim ADR 0002 keeps consumer code change at zero. |
| owl | Planned | Pager / cat-class utility. |
| agnoshi | Planned | AI shell — `fuzzy` ready for shell completion. |
| daimon | Planned | Agent orchestration — `re2` ready for DoS-safe pattern gates; `pcre` with step-limit for richer patterns; `fuzzy` for fuzzy-name match. |

## Next

Sequencing through to v1.0 (per ADR 0008's post-stdlib-unicode reshape
+ the no-fragmentation collapse):

1. **v0.8.0 — M4.5 completion release** — pcre lookbehind
   `(?<=)`/`(?<!)` (fixed-width), pcre recursion `(?R)`/`(?P>name)`,
   fuzzy exact-start recovery, `\p{L}` Unicode property classes for
   re2/pcre/vim (stdlib `unicode_category`), `(?i)` Unicode
   case-fold upgrade for re2/pcre (stdlib `unicode_fold`), fuzzy
   `FUZZY_FLAG_UNICODE_NFD` (stdlib `str_normalize`), vim →
   `src/posix_classes.cyr` refactor. **Surface bre/vim backref
   decision at ship time** — that question lands separately.
2. **v0.8.1 — bre/vim backref (decision-gated)** — pinned slot.
   Only releases if user decides "yes"; collapses to a CHANGELOG
   note if "no". Either way, "what happened to backref?" has a
   roadmap answer.
3. **M5 (post-v0.8.x)** — P(-1) hardening + closeout + surface freeze.
4. **v1.0** — fold-ready release.

See [`roadmap.md`](roadmap.md) for the full plan.
