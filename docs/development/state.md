# niyama — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**1.0.12** — toolchain and dependency refresh, shipped 2026-09-23. No
engine source changes; `dist/niyama.cyr` changed only in its version
header.
- **Pin and `lib/`:** pin `6.6.2` → `6.6.6`. `lib/` was wiped and
  re-vendored with `cyrius deps` (110 → 30 files, byte-identical to the
  6.6.6 snapshot).
- **Lock and CI:** `cyrius.lock` is now a real lock, and CI/release check it
  with `cyrius deps` + `cyrius deps --verify` (see § Toolchain).
- **Actions:** moved to their Node 24 majors, because GitHub removed the
  Node 20 runtime on 2026-09-23.
- **aarch64:** the release aarch64 binary is restored. It was silently
  skipped from 1.0.3 through 1.0.11 because the gate named the pre-6.0
  `cc5_aarch64`.
- **Gates:** tests 773 and fuzz 1689 are unchanged, all green. The bench A/B
  (6.6.2 vs 6.6.6) has no regression: 58 rows, mean −0.16%, none beyond
  ±5%. DCE binary 323,432 → 323,624 B.
- **Roadmap:** rewritten to be forward-looking only.

See CHANGELOG § 1.0.12.

**1.0.11** — toolchain pin `6.6.0` → `6.6.2`, shipped 2026-09-12. No
engine source changes. `tests/{bre,re2,pcre,vim}.tcyr` continuation lines
were reflowed to `cyrius fmt`'s style, which cleared the one `audit` fmt
finding.

**1.0.10** — refactor release shipped 2026-09-08. Lands the
cross-engine duplication v1.0.9's audit identified but deferred. New
shared module **`src/nfa_edit.cyr`** holds the NFA-blob edit primitives
every engine had its own byte-identical copy of; six primitives went
from four implementations to one, two from four to two (pcre keeps
extended versions — it has five more branch-carrying opcodes plus
COND, whose arg1 is a group index that must not relocate). The
per-engine `_<engine>_*` entry points remain thin delegates, so all
~183 call sites and the per-engine naming are untouched. **No
behaviour change**: no engine semantics, no error codes, no public
surface. Code lines 5293 → 5178 (−115); raw lines +37 because the
shared module carries the layout/ordering documentation four anonymous
copies never had. Tests 741 → **773** (recorded at the time as
747 → 779 — see § Tests) — `tests/niyama.tcyr` is now the
cross-engine invariant suite, pinning `OP_JMP`/`OP_SPLIT` numbering
across all four engines because the shared relocation code identifies
branches by those numbers. Bench: 53 rows, **0 over ±5%**, mean
+0.68%. See CHANGELOG § 1.0.10.

**1.0.9** — P(-1) hardening pass shipped 2026-09-08. Audit,
refactor, optimization and security sweep. **One CRITICAL and eight
HIGH findings, all fixed** — a heap out-of-bounds write via a crafted
nested-`?` pattern (all four engines' shift-based growth paths bypassed
the `MAX_INSTRS` ceiling), three unbounded unreclaimable heap-growth
sites (per-start-position and per-backtrack `alloc()` on a bump
allocator that never frees — measured 6.4 MB and 45.7 MB retained,
both now 0), a catastrophic-backtracking guard that was `(len+1)x`
weaker than documented, and `{n,m}` compiled by re-parsing the atom
(corrupting capture numbering and falsely rejecting `(a){10}` /
`(?<x>a){2}` in all four engines). Plus 6 MEDIUM and 6 LOW. Every
finding was reproduced with a compiled probe before repair.
**All 53 bench rows faster or neutral, 0 regressions, mean −14.29%** —
a side effect of removing locked `alloc()` calls from the matcher hot
loop. Tests 661 → **741 assertions** (recorded at the time as 747 —
see § Tests). Two confirmed findings were
rejected on measurement. See CHANGELOG § 1.0.9 and
[`../audit/2026-09-08-audit.md`](../audit/2026-09-08-audit.md).

**Known limitation (documented, not fixed)**: `_pcre_match_run`
recurses natively per input position, so `PCRE_MAX_DEPTH = 256` caps
consuming quantifiers at ~250 positions — `a*$` over 300 bytes reports
no match. Pre-existing and **not a tunable**: frames measure ~20 KB and
the process SIGSEGVs past ~400 frames on an 8 MB stack. The fix is an
explicit heap backtrack stack (matcher-core rewrite), deferred.
Mitigated by `PCRE_E_DEPTH_EXCEEDED = 12` so the condition is
observable rather than a silent false negative.

**1.0.8** — maintenance patch shipped 2026-09-07. Toolchain pin
bumped 6.5.29 → 6.6.0 (wrapper-drift catch-up) and the vendored
`lib/` re-synced from the 6.6.0 snapshot. No engine source changes;
`dist/niyama.cyr` byte-identical but for its version header.
9 stdlib files changed content (`fmt`, `io`, `unicode/casefold`,
`unicode/normalize`, 5 × `syscalls_*`); the two `unicode` ones are
comment/whitespace only. Three undeclared stale files
(`atomic`, `fnptr`, `result`) dropped from `lib/` — they now resolve
from the snapshot, as `lib/bench.cyr` always has. Fixed
`tests/niyama.bcyr`, which had never compiled (called a `bench()`
that stdlib does not expose). DCE binary 323,416 B → 323,432 B
(+16 B). Tests/fuzz/bench all green, no regressions.
See CHANGELOG § 1.0.8.

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

- **Cyrius pin**: `6.6.6` (in `cyrius.cyml [package].cyrius`).
  Floor remains `5.8.65` for stdlib `lib/unicode/` per ADR 0008
  (categories at .49, casefold at .50, normalize at .51, codec
  lift at .55, NFKC/NFKD at .60). Bump history: `5.8.42`
  post-v0.7.0 → `5.8.65` for v0.8.0 (per ADR 0008) → `5.11.4`
  at v1.0.2 (for `: i64` return-type syntax) → `6.0.1` at v1.0.3
  → `6.1.27` at v1.0.4 → `6.2.1` at v1.0.5 → `6.4.64` at v1.0.6
  → `6.5.29` at v1.0.7 → `6.6.0` at v1.0.8 → `6.6.2` at v1.0.11
  → `6.6.6` at v1.0.12. Each matched the installed wrapper at the
  time, and none required an engine source change.
- **The pin selects the compiler.** When the pin differs from the
  running wrapper, the wrapper re-execs `~/.cyrius/versions/<pin>/bin/cyrius`,
  which builds with its own sibling `cycc`. That has held for pinned
  wrappers from 6.5.44 on (cyrius CHANGELOG 6.5.47), and was measured at
  v1.0.12: under the 6.6.2 pin, `cyrius build -v` reports
  `versions/6.6.2/bin/cycc`. A bump is therefore a real compiler change.
  The "advisory" behaviour recorded at v1.0.8 was true only because its
  6.5.29 pin predated that fix.
- **Re-vendor `lib/` by wiping it: `rm -rf lib && cyrius deps`.** On 6.6.6
  that vendors 30 files: the declared leaves plus their transitive
  includes (`atomic`, `fnptr`, `result`, `args_*`). It is also exactly
  what CI vendors. The alternatives differ:
  - `cyrius lib sync` copies only the 25 files behind the declared leaves,
    and syncing over an existing `lib/` leaves transitive modules stale.
    At v1.0.8 that would have mixed two `Result` ABIs.
  - `cyrius update` copies the whole 111-file snapshot. That is how
    `lib/` held 110 files before v1.0.12.

  Undeclared modules such as `lib/bench.cyr` resolve from the pinned
  snapshot.
- **`cyrius.lock` is a real integrity record (v1.0.12).** It holds one
  sha256 per vendored file plus a `cyrius <pin>` trailer, committed.
  - `cyrius deps` re-locks silently when the pin changes.
  - Under an unchanged pin, it refuses any file whose snapshot disagrees
    with the lock (the cyrius 6.6.4 guard). Only `--relock` accepts that.
  - `cyrius deps --verify` checks `lib/` against the lock. CI and release
    run `cyrius deps` followed by `cyrius deps --verify`.
  - The pre-1.0.12 comment-only stub and CI's `--no-lock` are retired. The
    0-byte-truncation bug they worked around was fixed in cyrius 6.0.2, and
    on 6.6.x every `build` / `test` / `bench` / `fuzz` / `distlib` /
    `audit` run rewrites a stub lock anyway.
- **`cyrius audit` runs but exits 1.** On 6.6.6 its default mode sweeps
  fmt, lint, docs, tests and bench over `src/`, `tests/` and `fuzz/`. Any
  lint warning or any undocumented public function sets exit code 1, and
  niyama has two blockers:
  - **Lint:** 23 warnings (11 in `src/`, 11 in `tests/`, 1 in `fuzz/`);
    18 are over-long lines and 5 are consecutive blank lines.
  - **Docs:** 150 undocumented public fns. It counted 141 under 6.6.2
    because cyrdoc used to stop reading a file at 64 KB, and
    `src/pcre.cyr` is 89.7 KB.

  fmt has been clean since 1.0.11. Tracked in `roadmap.md` § Maintenance.
- **The `description`-truncation bug did not reproduce on 6.6.6.** At
  v1.0.8, `cyrius build` / `cyrius deps` were seen cutting `cyrius.cyml`'s
  `description` at the first `;`. At v1.0.12, every command run in this
  tree left the field byte-identical. A `git diff cyrius.cyml` after
  toolchain runs is still cheap insurance.
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
- **`src/nfa_edit.cyr`** — shared NFA-blob edit primitives (v1.0.10).
  `_nfa_shift_right`, `_nfa_patch_arg1/2`, `_nfa_shift_targets_one`,
  `_nfa_tmpl_save`, `_nfa_tmpl_reloc`, `_nfa_zero_class`,
  `_nfa_lit_byte`. Takes `instr_base` explicitly rather than reading an
  engine global, which is what makes sharing possible. **Must come first**
  in `[lib] modules` and in every test/fuzz/bench include list — Cyrius is
  single-pass. ~188 lines.
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

- `dist/niyama.deps` — sidecar emitted by `cyrius distlib` from 6.6.0
  (v1.0.8), listing the 9 stdlib leaf requirements for downstream
  `cyrius deps`. Checked in alongside the bundle.
- `dist/niyama.cyr` — single-include bundle. v1.0.10 prepends
  `src/nfa_edit.cyr` ahead of the two other shared modules; consumers are
  unaffected because the bundle is still one file and `dist/niyama.deps`
  is unchanged. v0.8.0 prepends both
  shared modules (`src/posix_classes.cyr` then `src/unicode_props.cyr`)
  ahead of the five engine modules. Consumers also need stdlib
  `lib/str.cyr` and `lib/unicode/{categories,casefold,normalize}.cyr`.
  ADR 0010 locks its public symbol set.
- **Fold copy.** cyrius 6.6.6 vendors niyama **1.0.11** as
  `lib/niyama.cyr`, with a body byte-identical to this repo's `dist/`.
  1.0.12 changes only the header, so the fold is current in substance.
- **The fold is never checked for agnos.** cyrius's folds-parity gate
  skips niyama on Linux: its preamble lacks `lib/unicode`, so the fold's
  `str_normalize(…, NFD)` fails with `undefined variable 'NFD'`. That is a
  gap in cyrius's gate, not a niyama defect, and is recorded here only.

## Tests

- `tests/niyama.tcyr` — **cross-engine invariants** (34 assertions,
  was a 2-assertion scaffold). Pins the assumptions no single-engine suite
  can see: every engine's `OP_JMP`/`OP_SPLIT` must equal `NFA_OP_JMP`/
  `NFA_OP_SPLIT` (the shared relocation code identifies branches by those
  numbers, so renumbering an engine would silently emit a corrupt program),
  `MAX_CLASSES == 64` everywhere, and the shared escape decoder and
  blob-edit paths behaving identically across engines.
- **`tests/bre.tcyr`** — 123 BRE assertions (v1.0.9: +11).
- **`tests/re2.tcyr`** — 199 RE2 assertions (v1.0.9: +24).
- **`tests/pcre.tcyr`** — 225 PCRE assertions (v1.0.9: +19).
- **`tests/fuzzy.tcyr`** — 70 fuzzy assertions (v1.0.9: +13).
- **`tests/vim.tcyr`** — 122 vim assertions (v1.0.9: +13).
- **`tests/{bre,re2,pcre,fuzzy,vim}.bcyr`** — per-engine bench harnesses.
- `tests/niyama.bcyr` — scaffold smoke bench (1 row). Repaired at
  v1.0.8; it had never compiled, calling a `bench()` entry point
  stdlib does not expose. 6 runnable bench harnesses total.
- **`fuzz/{bre,re2,pcre,fuzzy,vim}.fcyr`** — per-engine fuzz harnesses.

Aggregate: `cyrius test` reports **6 files, 773 assertions**, all
passing. The history is 661 at v1.0.8 → 741 at v1.0.9 (the P(-1)
regression coverage) → 773 at v1.0.10 (the cross-engine invariant suite),
unchanged since.

**Correction (v1.0.12):** v1.0.9 and v1.0.10 recorded 747 and 779. The
per-file counts above were right; the aggregates also added the
`6 passed, 0 failed` line that `cyrius test` prints last, which counts
files, not assertions. When totalling, sum only the lines ending
`(N total)`.

`cyrius fuzz` runs 6 files, all passing: the 5 per-engine harnesses
(**1689 assertions**, unchanged since v0.9.0) plus the zero-assertion
`tests/niyama.fcyr` scaffold.

Bench history captured in [`../benchmarks.md`](../benchmarks.md).
Security audit history in [`../audit/`](../audit/).
Architecture invariants in [`../architecture/`](../architecture/).

## Dependencies

Direct (declared in `cyrius.cyml`):

- stdlib — string, fmt, alloc, io, vec, str, syscalls, assert, unicode.
  (`unicode` added post-v0.7.0 alongside the toolchain bump per ADR
  0008 — pulls in the `lib/unicode/{categories,casefold,normalize,
  _decode}.cyr` tree.)

`cyrius deps` vendors those 9 leaves as 30 files including transitive
includes. Since v1.0.12 each is locked by sha256 in `cyrius.lock`, stamped
with the pin, and CI checks it with `cyrius deps --verify`. niyama has no
git `[deps.*]` entries, so `cyrius update` has nothing to move.

## Consumers

| Consumer | Status | Notes |
|----------|--------|-------|
| [cyim](https://github.com/MacCracken/cyim) | Active (consumer #1) | Includes cyrius stdlib's folded `lib/niyama.cyr` (cyim 1.10.x). `--regex=<flavor>` covers all five flavors; per cyim ADR 0002, cyim needs no code changes. |
| AGNOS kernel | Consumer #2 per ADR 0011 | Long-horizon pin, queued for the bare-metal target. No `niyama_*` references in `agnos/src` as of 2026-09-23. |
| owl | Planned | Pager / cat-class utility. No `niyama_*` references in `src/` as of 2026-09-23. |
| agnoshi | Planned | AI shell — `fuzzy` for shell completion. No `niyama_*` references in `src/` as of 2026-09-23. |
| daimon | Planned | Agent orchestration — `re2` for DoS-safe pattern gates, `pcre` with a step limit for richer patterns, `fuzzy` for fuzzy-name match. No `niyama_*` references in `src/` as of 2026-09-23. |

## Next

niyama-the-repo is in fold-maintenance. The fold triggered on 2026-05-06
at cyrius v5.9.0 (ADR 0011). Going forward:

1. **v1.0.x patches** — bug fixes, hardening and toolchain pin bumps; no
   surface changes (ADR 0010). Each tag reaches cyrius stdlib's
   `lib/niyama.cyr` on its next refold.
2. **v1.1.0** — the pcre explicit heap backtrack stack, which removes the
   `PCRE_MAX_DEPTH` false negative described under § Version, 1.0.9.
3. **Post-fold extensions** — additive-only, landing in cyrius stdlib's
   vendored copy, not here. The pinned candidates are vim backref
   (ADR 0009), fuzzy `_search_at`, long Unicode property names and full
   `(?i)` case folding.
4. **niyama v2.0** — speculative, only if a need exceeds the additive-only
   model. It would be a new namespace; v1.x consumers are unaffected.

See [`roadmap.md`](roadmap.md) for the open work (forward-looking only;
shipped history is the CHANGELOG), [ADR 0010](../adr/0010-surface-freeze.md)
for the freeze contract, and
[ADR 0011](../adr/0011-fold-readiness-and-trigger.md) for the fold record.
