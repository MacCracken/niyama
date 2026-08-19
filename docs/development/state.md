# niyama — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**1.0.7** — maintenance patch shipped 2026-08-19. Toolchain pin
bumped 6.4.64 → 6.5.29 (wrapper-drift catch-up); no engine source
changes, `dist/niyama.cyr` byte-identical but for its version
header. Fixed a pre-existing 2-byte `.rodata` over-read in the
`src/main.cyr` smoke banner (write length 87 vs an 85-byte
literal, LOW severity, outside the frozen engine surface) and
un-pinned that banner's hardcoded `1.0.0` version string.
The smoke binary grew 4,544 B → 401,240 B across the bump —
**not** a regression: cyrius ≤ 6.5.15 silently ignored
`[deps] stdlib` auto-include, and 6.5.16 fixed it, so the declared
stdlib (incl. the ~307 KB `lib/unicode/*_data.cyr` tables) is now
actually linked. niyama was never affected functionally — all
`.tcyr` files and engine consumers use explicit `include` lines.
Tests/fuzz/bench all green. See CHANGELOG § 1.0.7.

**1.0.1** — fold-trigger release shipped 2026-05-06. ADR 0011
**triggered**: cyrius v5.9.0 vendored `dist/niyama.cyr` from this
tag byte-identical as `lib/niyama.cyr`. v1.0.1 also corrected the
v1.0.0 dist defect (the manifest-scaffold `include` shape) by
wiring `[lib] modules = [...]` in `cyrius.cyml` and regenerating
via `cyrius distlib`. niyama-the-repo enters fold-maintenance:
v1.x patches still land here, propagate via cyrius update;
post-fold extensions land in cyrius stdlib's vendored copy.

**1.0.0** — fold-ready release shipped 2026-05-05. Public surface
locked per ADR 0010 + ADR 0011. niyama-the-repo enters maintenance
mode; v1.x patches are bug-fix only. Post-fold extensions land in
cyrius stdlib's vendored `lib/niyama.cyr` once the fold trigger
fires (consumer #2 materializes; cyim is #1).

v1.0 sweeps the v0.9.0 review remainders: `(?i)ß` semantics-lock
test, recursion+lookaround nesting tests, empty-pattern tests
across all 5 engines, fold ADR 0011, public API reference at
`docs/api/`, the 5 deferred CLAUDE.md sections, working-doc
cleanup. No engine code changes vs v0.9.0.

**0.9.0** — M5 P(-1) hardening + surface freeze release shipped
2026-05-05. Last release before v1.0 fold-ready. Per ADR 0010,
the public `niyama_<engine>_*` API surface locked. The 9-step
P(-1) pass ran clean: no CRITICAL/HIGH security findings, no
bench regressions (all 47 measurements within ±2.5% of v0.8.0
baseline).

**v0.8.0** — M4.5 completion release shipped 2026-05-05. Per ADR 0008
(Unicode-stdlib pivot + sub-release collapse), every remaining M4.5
deferral landed: `\p{L}` for re2/pcre/vim, multi-byte literal
pattern chars, `(?i)` Unicode case-fold upgrade, fuzzy NFD via
stdlib `str_normalize`, fuzzy exact-start recovery, vim →
posix_classes refactor, pcre lookbehind (fixed-width), pcre
recursion `(?R)` / `(?P>NAME)`. v0.8.1 collapsed (no release);
backref question moved to v0.9.0 review (resolved as ADR 0009).

**v0.8.x ladder rule.** Any feature deferred *out of v0.8.0 during
implementation* gets a pinned v0.8.x slot in `roadmap.md`, not a
floating "post-v1.0 / vN.0 maybe" note. Pins don't force releases —
slots that don't warrant a tag collapse to a CHANGELOG note.
Established post-v0.7.0; prevented silent roadmap shrinkage and
release-tag dust through the v0.8.0 → v0.9.0 → v1.0 sequence.

## Toolchain

- **Cyrius pin**: `6.5.29` (in `cyrius.cyml [package].cyrius`).
  Floor remains `5.8.65` for stdlib `lib/unicode/` per ADR 0008
  (categories at .49, casefold at .50, normalize at .51, codec
  lift at .55, NFKC/NFKD at .60). Bump history: `5.8.42`
  post-v0.7.0 → `5.8.65` for v0.8.0 (per ADR 0008) → `5.11.4`
  at v1.0.2 (for `: i64` return-type syntax) → `6.0.1` at v1.0.3
  → `6.1.27` at v1.0.4 → `6.2.1` at v1.0.5 → `6.4.64` at v1.0.6
  → `6.5.29` at v1.0.7 (each matches the installed wrapper at
  the time; zero engine source changes across all of them).
- **`[deps] stdlib` auto-include only works from cyrius 6.5.16.**
  Every toolchain through 6.5.15 silently ignored manifest-declared
  stdlib deps — a probe calling `unicode_category()` with no
  explicit `include` drew `warning: undefined function` (a warning,
  **not** an error) on 6.4.64 / 6.5.10 / 6.5.14 / 6.5.15, and links
  correctly on 6.5.16 onward. This is why the v1.0.7 smoke binary
  went 4,544 B → 401,240 B: the declared stdlib is now genuinely
  linked, ~307 KB of it the `lib/unicode/*_data.cyr` tables.
  **niyama never relied on the broken path** — all `tests/*.tcyr`
  and engine consumers name their stdlib modules with explicit
  `include` lines, and `src/main.cyr` references no stdlib at all.
  Consequence to remember: 6.5.16+ links every declared module
  whether or not the entry point uses it, so the smoke binary
  carries tables it never calls. Library consumers are unaffected —
  the shipped artifact is the `dist/niyama.cyr` source bundle.

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
| `grep -G` / `sed` POSIX BRE compatibility (no backref) | `bre` |
| ERE + provable linear-time DoS-safety on untrusted input | `re2` |
| **Backref `\1`-`\9`** in any flavor | `pcre` (per ADR 0009 — bre/vim do not implement backref through v1.0; vim *may* gain it post-fold via cyrius stdlib) |
| Lookahead, lookbehind, named captures, atomic groups, recursion | `pcre` |
| Both ERE features AND DoS-safety | `re2` (no backref/lookaround) |
| Both PCRE features AND bounded DoS-safety | `pcre` with `niyama_pcre_set_step_limit()` |
| Typo-tolerant matching, shell completion, fuzzy-name lookup | `fuzzy` |
| vim/cyim-flavor patterns (`\<`, `\>`, `\zs`/`\ze`, magicness) | `vim` |
| Unicode property classes `\p{L}` etc. | `re2`, `pcre`, or `vim` (v0.8.0+) |
| Case-insensitive / multi-line / dot-newline matching | `re2` or `pcre` with `(?i)`/`(?m)`/`(?s)` |
| Conditional or branch-reset patterns | `pcre` (`(?(...)...)`, `(?\|...)`) |
| Linear-time *family* (DoS-safe by construction) | `re2`, `bre`, `vim` — backref structurally rejected |
| Backtracking *family* (full PCRE feature set, step-limit guarded) | `pcre` |

### Known asymmetries / scope notes (v1.0)

Per ADR 0010 (Surface freeze) + the v0.9.0 review remainders:

- **`niyama_fuzzy_search_at` does not exist** (A1). bre/re2/pcre/vim
  all expose `_search_at(nfa, s, len, from)` for substring search
  from offset; fuzzy doesn't. Workaround: slice the input externally
  (`s + from` with appropriate `len - from`) and call
  `niyama_fuzzy_search`. Pinned for post-fold cyrius-stdlib
  extension.
- **Long Unicode property names not supported** (G3). niyama
  implements `\p{NAME}` with short names only — the 7 single-letter
  aggregates (`L M N P S Z C`) and 30 two-letter leaves (`Lu Ll Lt
  Lm Lo Mn ... Cn`). Long forms `\p{Letter}`, `\p{Uppercase_Letter}`
  not supported. Script properties `\p{Greek}`, `\p{Cyrillic}` not
  supported (different table, post-fold candidate). Block properties
  `\p{InGreek}` not supported.
- **Absolute anchors `\A` / `\z` / `\Z` not implemented** (G4).
  niyama's `^` and `$` are strict-by-default in re2/pcre (per
  v0.7.0): `^` matches only at pos 0, `$` only at pos len, both
  unaffected by `\n` unless `(?m)` is set. This makes `^/$` (default)
  semantically equivalent to PCRE2's `\A`/`\z` for niyama purposes.
  Consumers porting PCRE2 patterns can substitute `\A` → `^` and
  `\z` → `$`. `\Z` (string-end-or-trailing-newline) has no exact
  equivalent; in practice `$\n?$` or `(?m)$` covers most uses.
- **`(?i)` is 1:1 fold only**, not full Unicode case folding. ß
  matches ß but not SS; İ matches İ but not i̇. Per ADR 0008 +
  v1.0 semantics-lock test (`tests/re2.tcyr` + `tests/pcre.tcyr`).
  Post-fold candidate.
- **`compile_opts` asymmetry**: only fuzzy and vim expose a
  `_compile_opts` entry point. bre/re2/pcre have only `_compile`.
  Consumers needing flag-on-compile for the linear-time engines
  would need a post-fold ABI extension.

## Fold-ready artifact

- `dist/niyama.cyr` — single-include bundle. v0.8.0 prepends both
  shared modules (`src/posix_classes.cyr` then `src/unicode_props.cyr`)
  ahead of the five engine modules. Consumers also need stdlib
  `lib/str.cyr` and `lib/unicode/{categories,casefold,normalize}.cyr`.
  M5 surface freeze ADR will lock its public symbol set.

## Tests

- `tests/niyama.tcyr` — scaffold smoke (2 assertions).
- **`tests/bre.tcyr`** — 112 BRE assertions (v1.0: +3 empty-pattern).
- **`tests/re2.tcyr`** — 175 RE2 assertions (v1.0: +3 ß
  semantics-lock, +3 empty-pattern).
- **`tests/pcre.tcyr`** — 206 PCRE assertions (v1.0: +3 ß, +12
  recursion-nesting, +3 empty-pattern).
- **`tests/fuzzy.tcyr`** — 57 fuzzy assertions (v1.0: +4 empty).
- **`tests/vim.tcyr`** — 109 vim assertions (v1.0: +3 empty).
- **`tests/{bre,re2,pcre,fuzzy,vim}.bcyr`** — per-engine bench harnesses.
- **`fuzz/{bre,re2,pcre,fuzzy,vim}.fcyr`** — per-engine fuzz harnesses.

Aggregate: `cyrius test` reports **6 files, 661 assertions** all passing
(was 627 at v0.9.0; +34 v1.0 sweep).
`cyrius fuzz` reports **5 files, 1689 assertions** all passing
(unchanged from v0.9.0; no engine code changes).

Bench history captured in [`../benchmarks.md`](../benchmarks.md).
Security audit history in [`../audit/`](../audit/).
Architecture invariants in [`../architecture/`](../architecture/).

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

niyama-the-repo enters maintenance mode at v1.0. Going forward:

1. **Bug-fix patches (v1.0.x)** — only fixes; no surface changes.
   Reported issues land here.
2. **Fold trigger** — when consumer #2 materializes (per ADR 0011's
   gate), cyrius stdlib vendors `dist/niyama.cyr` as
   `lib/niyama.cyr` byte-identical. ADR 0011 status updates to
   "Triggered" at that point.
3. **Post-fold extensions** — additive-only changes (per ADR 0010)
   land in cyrius stdlib's vendored copy, not in niyama-the-repo.
   Pinned candidates: vim backref (ADR 0009), fuzzy `_search_at`,
   Unicode long property names, `(?i)` full case folding.
4. **niyama v2.0** — speculative; only if a future ecosystem need
   exceeds the additive-only post-fold model. Would be a new
   namespace; v1.x consumers unaffected.

See [`roadmap.md`](roadmap.md) for the complete milestone history;
[ADR 0010](../adr/0010-surface-freeze.md) for the freeze contract;
[ADR 0011](../adr/0011-fold-readiness-and-trigger.md) for the
fold trigger checklist.
3. **M5 (post-v0.9.0)** — P(-1) hardening + closeout + surface freeze.
4. **v1.0** — fold-ready release.

See [`roadmap.md`](roadmap.md) for the full plan.
