# niyama — Roadmap

> **Forward-looking only.** Nothing below has shipped. The record of what
> has lives in [`CHANGELOG.md`](../../CHANGELOG.md) — complete from 0.1.0;
> milestones M0–M5 are the 0.1.0 → 0.9.0 entries — with the decisions in
> [`../adr/`](../adr/) and live state (version, toolchain pin, counts,
> consumers) in [`state.md`](state.md). **When an item ships, delete it
> here**; its CHANGELOG entry is the record.
>
> Last swept: 2026-09-23, at v1.0.12.

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

## Next minor — v1.1.0

### pcre: explicit heap backtrack stack

**The bug.** `_pcre_match_run` recurses natively on SPLIT and SAVE, so
recursion depth grows with **input position**, not pattern nesting. With
`PCRE_MAX_DEPTH = 256`, any quantifier that must consume more than ~250
positions exhausts the bound and the match fails — a false negative:

```
a*$        over 300 × 'a'   → no match   (correct: match)
(a){200}   over 200 × 'a'   → no match   (correct: match)
(a){100}   over 100 × 'a'   → match
```

**Why it is not a tunable.** Frames measure ~20 KB and the process
SIGSEGVs past ~400 of them on an 8 MB stack, so 256 is already near the
ceiling; raising it trades a false negative for a crash. Since v1.0.9 the
condition is at least observable — `PCRE_E_DEPTH_EXCEEDED = 12` via
`niyama_pcre_last_error()` — which is the interim mitigation, not the fix.
Full analysis: [`../audit/2026-09-08-audit.md`](../audit/2026-09-08-audit.md)
§ Known limitation.

**The fix** replaces native recursion with an explicit heap-allocated
backtrack stack — a matcher-core rewrite, which is why it was held out of
the v1.0.9 security patch. Done when:

- The probes above return the correct answer, and matching depth no longer
  scales with input length.
- The step limit (`niyama_pcre_set_step_limit`, default 1M) remains the
  only DoS bound, with its documented semantics unchanged.
- Backtrack storage is process-lifetime scratch initialised in
  `_pcre_lazy_init` — never an `alloc()` per call or per backtrack point.
  `alloc()` is a bump allocator that never frees; v1.0.9 measured 45.7 MB
  retained from exactly that pattern.
- `PCRE_E_DEPTH_EXCEEDED` keeps its frozen value and still reports, rather
  than crashes, if the new stack has a ceiling.
- Regression tests for the probes, long-input cases in `fuzz/pcre.fcyr`,
  and a same-boot bench A/B showing no regression on the 13 pcre rows.
- An ADR recording the stack design.

## Maintenance — v1.0.x open items

- **Toolchain tracking.** Bump the `cyrius` pin with each cyrius release:
  wipe and re-vendor `lib/` (`rm -rf lib && cyrius deps`), commit the
  regenerated `cyrius.lock`, regenerate `dist/` with `cyrius distlib`, and
  run test + fuzz + a same-boot bench A/B against the old pin (method:
  [`../benchmarks.md`](../benchmarks.md)). Check that cyrius's vendored
  `lib/niyama.cyr` matches the latest tag.
- **CI does less than CLAUDE.md says.** `ci.yml` runs build + test only;
  CLAUDE.md § CI/Release describes lint and fuzz steps that do not exist.
  Either add the steps (lint advisory, fuzz gating) or correct the doc.
- **CI installs the toolchain unverified.** Both workflows fetch the cyrius
  tarball with `curl` and unpack it without checking the published
  `.sha256` or the signed `SHA256SUMS`. That is the gap cyrius's CVE-21
  names. The committed `cyrius.lock` now catches a changed *stdlib*
  (`cyrius deps` refuses it), but nothing checks the compiler binaries.
  Verify the checksum, or install through cyrius's own `install.sh`,
  which refuses a mismatch.
- **`cyrius audit` exits 1** on accepted findings, so it cannot gate
  anything. It lints `src/` + `tests/` + `fuzz/` — 23 warnings (11 / 11 / 1):
  18 lines over 120 characters and 5 runs of consecutive blank lines — and
  counts 150 public functions without doc comments (6.6.6's count; 6.6.2
  counted 141 on identical source). Document the functions and clear the
  lint set, or record which findings are accepted so a non-zero exit means
  something new.

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
