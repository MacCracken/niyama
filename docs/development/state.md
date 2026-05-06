# niyama — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.7.0** — first M4.5 catch-up release shipped 2026-05-03. Eight
features land across bre / re2 / pcre under one shared
`src/posix_classes.cyr` module per ADR 0007 (the no-Unicode-dep
slice). All five engines still shipped.

**Unreleased (post-v0.7.0)** — toolchain bumped to Cyrius 5.8.65,
which ships `lib/unicode/` (categories, casefold, normalize, UTF-8
codec). ADR 0008 reshapes M4.5 around the new stdlib + a user
direction to not fragment a coherent milestone: **v0.8.0 = M4.5
completion release**. Custom `src/unicode.cyr` deleted from the
plan. No engine code touched yet.

Next:

1. **v0.8.0 — M4.5 completion** — pcre lookbehind, pcre recursion,
   fuzzy exact-start, `\p{L}` for re2/pcre/vim, `(?i)` Unicode
   case-fold upgrade for re2/pcre, fuzzy NFD, vim →
   `src/posix_classes.cyr` refactor.
2. **v0.8.1 — bre/vim backref (decision-gated)** — pinned slot;
   only releases if the user says "yes" on backref. Otherwise
   collapses to a `[Unreleased]` CHANGELOG note.
3. **M5** — hardening + closeout + surface freeze.
4. **v1.0** — fold-ready release.

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
  + name recognizer (v0.7.0). ~200 lines.
- **`src/bre.cyr`** — POSIX BRE engine (M1, +`\<\>` and POSIX
  classes from v0.7.0). ~930 lines, Pike NFA.
- **`src/re2.cyr`** — RE2-flavor linear-time engine (M2, +named
  captures and inline flags from v0.7.0). ~1330 lines, Pike NFA.
- **`src/pcre.cyr`** — PCRE-light backtracking engine (M3,
  +POSIX classes / inline flags / `\K` / branch-reset / conditional
  / callouts from v0.7.0). ~1660 lines.
- **`src/fuzzy.cyr`** — Levenshtein edit-distance engine (M3.5).
  ~320 lines.
- **`src/vim.cyr`** — vim/cyim flavor engine (M4). ~1480 lines,
  Pike NFA.

## Engines shipped

| Engine | Status | ABI prefix | Algorithm | Notes |
|---|---|---|---|---|
| **bre** | ✅ v0.7.0 (M1 + catch-up) | `niyama_bre_*` | Pike NFA | POSIX BRE minus backrefs (per ADR 0002). v0.7.0 adds GNU `\<\>` boundaries + POSIX bracket classes (ADR 0007). |
| **re2** | ✅ v0.7.0 (M2 + catch-up) | `niyama_re2_*` | Pike NFA | ERE + linear-time guarantee at API (per ADR 0003). v0.7.0 adds named captures + inline flags + spec-strict `^/$/.` defaults (ADR 0007). |
| **pcre** | ✅ v0.7.0 (M3 + catch-up) | `niyama_pcre_*` | Backtracking | Perl-compat (per ADR 0004). Step-limit + depth bound. v0.7.0 adds POSIX classes / inline flags / `\K` / branch-reset / conditional / callouts + spec-strict `^/$/.` defaults (ADR 0007). |
| **fuzzy** | ✅ v0.5.0 (M3.5) | `niyama_fuzzy_*` | Levenshtein DP | Edit-distance, three match modes (per ADR 0005). |
| **vim** | ✅ v0.6.0 (M4) | `niyama_vim_*` | Pike NFA | vim/cyim flavor, 4 magicness modes (per ADR 0006). |

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

- `dist/niyama.cyr` — single-include bundle. v0.7.0 prepends
  `src/posix_classes.cyr` ahead of the five engine modules. M5
  surface freeze ADR will lock its public symbol set.

## Tests

- `tests/niyama.tcyr` — scaffold smoke (2 assertions).
- **`tests/bre.tcyr`** — 107 BRE assertions (was 68 pre-v0.7.0;
  +9 word-boundary, +28 POSIX-class + 2 supporting).
- **`tests/re2.tcyr`** — 101 RE2 assertions (was 76; +12 named,
  +12 inline-flag, +3 strict-default + supporting).
- **`tests/pcre.tcyr`** — 140 PCRE assertions (was 83; +8
  POSIX-class, +12 inline-flag, +3 strict-default, +4 `\K`,
  +8 branch-reset, +10 conditional, +8 callout + supporting).
- **`tests/fuzzy.tcyr`** — 45 fuzzy assertions.
- **`tests/vim.tcyr`** — 88 vim assertions across 13 groups.
- **`tests/{bre,re2,pcre,fuzzy,vim}.bcyr`** — per-engine bench harnesses.
- **`fuzz/{bre,re2,pcre,fuzzy,vim}.fcyr`** — per-engine fuzz harnesses.

Aggregate: `cyrius test` reports 6 files, 483 assertions all passing.
`cyrius fuzz` reports 5 files, 1658 assertions all passing.

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
