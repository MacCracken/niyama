# niyama — Roadmap

> **Forward-looking only.** Nothing below has shipped. The record of what
> has lives in [`CHANGELOG.md`](../../CHANGELOG.md) — complete from 0.1.0;
> milestones M0–M5 are the 0.1.0 → 0.9.0 entries — with the decisions in
> [`../adr/`](../adr/) and live state (version, toolchain pin, counts,
> consumers) in [`state.md`](state.md). **When an item ships, delete it
> here**; its CHANGELOG entry is the record.
>
> Last swept: 2026-10-08, at v1.1.0.

## Where niyama is

Post-v1.0 and post-fold. All five engines have shipped, the public
`niyama_<engine>_*` surface is frozen
([ADR 0010](../adr/0010-surface-freeze.md)), and cyrius stdlib has vendored
`dist/niyama.cyr` as `lib/niyama.cyr` since cyrius v5.9.0
([ADR 0011](../adr/0011-fold-readiness-and-trigger.md)). That decides where
each kind of change can land:

| Change | Lands in | Rule |
|---|---|---|
| Bug fix, hardening, toolchain pin bump | **This repo** (v1.x) — then `dist/niyama.cyr`, then cyrius stdlib's next refold | No new entry points, error codes or opcodes; no behaviour change beyond the fix |
| Additive surface growth | **cyrius stdlib's `lib/niyama.cyr`**, not this repo | New symbols at the next free slot; existing values immutable (ADR 0010 § Post-fold) |
| Anything additive-only cannot express | niyama **v2.0** — new namespace | Speculative; v1.x consumers unaffected |

## Maintenance — v1.x open items

- **Toolchain tracking.** Bump the `cyrius` pin with each cyrius release:
  wipe and re-vendor `lib/` (`rm -rf lib && cyrius deps`), commit the
  regenerated `cyrius.lock`, regenerate `dist/` with `cyrius distlib`, and
  run test + fuzz + a same-boot bench A/B against the old pin (method:
  [`../benchmarks.md`](../benchmarks.md)). Check that cyrius's vendored
  `lib/niyama.cyr` matches the latest tag.
- **Make `cyrius audit` gateable.** It exits 1 on accepted findings, so it
  cannot gate anything. On the 6.7.5 pin it lints `src/` + `tests/` +
  `fuzz/` — 21 warnings (9 / 11 / 1): 16 lines over 120 characters and 5
  runs of consecutive blank lines — and counts 150 public functions
  without doc comments. Document the functions and clear the lint set, or
  record which findings are accepted so a non-zero exit means something
  new; then CI's advisory lint step can gate too.
- **Every engine's compile trusts its allocations.** `niyama_<engine>_compile`
  dereferences its `alloc()`'d NFA unchecked in all four regex engines
  (`bre.cyr:695`, `re2.cyr:1095`, `pcre.cyr:1751`, `vim.cyr:1153`), and
  bre / re2 / vim do not check that their lazy init succeeded (pcre does
  since 1.1.0). Heap exhaustion becomes a write to the NULL page instead of
  `*_E_TOO_LARGE`, the 1.0.9 C10 class. Found during 1.1.0; the same
  one-line refusal in each engine, with a test under a capped allocator.
- **pcre backreference inside its own group.** A `\N` read while group N
  is open sees the current iteration's start with the previous iteration's
  end; when that span is negative the backreference moves the position
  backwards. `(?:x(a\1?))+` over `"xaxaa"`: niyama answers (0,3) with
  group 1 = (3,3); PCRE2 answers (0,5), group 1 = (3,5), because it keeps a
  group's previous capture until the group closes. Pre-existing (1.0.13
  answers the same); fixing it changes capture bookkeeping, so it needs
  its own design note. Found by the 1.1.0 differential run.
- **Flag loops → `loop` / `break`.** The engines' scanners and matchers
  use a flag + `continue` (CLAUDE.md § Cyrius Conventions) from an era
  when `break` past a `var` was unreliable; it is not on 6.7.5. Adopting
  cyrius 6.7.x `loop` raises the toolchain floor, so it is a deliberate
  release of its own, measured with the bench A/B, not a drive-by edit.
- **Public constants stay `var`.** The `PCRE_*` / `RE2_*` / … opcode,
  error-code and limit names are frozen surface (ADR 0010); turning them
  into 6.7.x `const` changes what kind of name a consumer sees, and a
  `const` beside a same-name `var` anywhere in a consumer's unit is a hard
  error. Not recommended for v1.x; revisit only with a v2.0 namespace.

## Post-fold extension candidates

These belong to **cyrius stdlib's `lib/niyama.cyr`, not this repo**
(ADR 0010 § Post-fold via cyrius stdlib). They are recorded here so the
question does not have to be re-derived, and so nobody lands them in
niyama-the-repo by accident.

**Pinned** — named in ADR 0010 / ADR 0011:

| Candidate | Source | Gate |
|---|---|---|
| vim backref `\1`–`\9` | [ADR 0009](../adr/0009-backref-review-and-exposure.md) — containment design already written (vim-only, either-or matcher dispatch, pcre-style step limit) | cyim asks |
| `niyama_fuzzy_search_at` | v0.9.0 review A1 — every other engine has `_search_at` | A consumer asks |
| Long `\p{...}` names, Unicode scripts and blocks | v0.9.0 review G3 — short names only today | A consumer asks |
| Full `(?i)` case folding (ß ↔ SS, İ ↔ i̇) | v1.0 semantics-lock test (C2) — 1:1 fold only today | A consumer asks |

**Unpinned** — only if a consumer asks:

- A distinct match-time code for a recursion loop (PCRE2's
  `PCRE2_ERROR_RECURSELOOP`). Since 1.1.0 a recursion that re-enters its
  group at the same position fails that branch
  ([ADR 0012](../adr/0012-pcre-explicit-backtrack-stack.md)); a new error
  code is additive surface, so it would land in cyrius stdlib.
- `_compile_opts` for bre / re2 / pcre (review A2; only fuzzy and vim have
  it).
- `\A` / `\z` / `\Z` absolute anchors (review G4). Strict-default `^` / `$`
  already behave as `\A` / `\z`; `\Z` has no exact equivalent.
- Numeric `(?N)` recursion syntax (pcre has `(?R)` and `(?P>NAME)`).
- vim mid-pattern mode switching (`\v` `\m` `\M` `\V`) and vim's `\X`
  escape set ([ADR 0006](../adr/0006-vim-engine-abi-and-scope.md)).

**Never** — bre backref is permanently out of scope, pre- and post-fold
(ADR 0009); re2 is structurally backref-free (ADR 0003).

## niyama v2.0 — speculative

Only if an ecosystem need exceeds the additive-only post-fold model. It
would be a new repository or a new top-level namespace; the frozen v1.x
surface stays available to consumers that pin it.

## Deferral rule

Anything deferred during implementation gets a named slot in this file
with a one-line scope note — never a floating "later" (the v0.8.x ladder
rule, [ADR 0008](../adr/0008-unicode-stdlib-pivot-and-reshape.md)).
Pinning is not shipping: a slot with nothing to ship collapses to a
CHANGELOG note.

## Out of scope

- **Cloud / hosted regex services.** niyama is in-process matchers only.
- **Regex-DSL extensions** (e.g. structural regex à la Sam). Possible if a
  consumer asks; not a commitment.
- **Regex generation / inverse regex** (synthesising strings from
  patterns). Distinct enough to warrant its own repo (`viparyaya`?) if
  ever needed.
- **GUI / TUI regex-testing tools.** A consumer concern — cyim could build
  a `:RegexTest` mode.
- **Engine-on-engine recursion** (e.g. a pcre recursive pattern calling
  into re2). Engines stay independent.
- **Replacement-language helpers** (`~`, `&`, `\u` / `\l` / `\U` / `\L`).
  Substitution is the consumer's job; cyim handles its own.
- **Backwards-compat shims for non-AGNOS regex APIs** (e.g. wrapping
  PCRE2's C API). niyama is sovereign Cyrius — a consumer that needs
  PCRE2's wire API wraps it themselves.
