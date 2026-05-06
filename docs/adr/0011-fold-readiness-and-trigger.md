# 0011 — Fold readiness and post-v1.0 fold trigger

**Status**: Triggered: 2026-05-06 via cyrius v5.9.0
**Date**: 2026-05-05 (Accepted) / 2026-05-06 (Triggered)

## Context

Per ADR 0001 (niyama is the additional-engines repo, sandhi-pattern
fold lifecycle), niyama is foldable into cyrius stdlib as
`lib/niyama.cyr` once a multi-consumer adoption gate is met. The
gate, restated:

1. **≥2 long-horizon AGNOS-lineage consumers** actively using niyama.
2. **v1.0.0 reached** with full feature set shipped.
3. **Public surface frozen** (no breaking changes after the
   freeze ADR).
4. **Explicit fold ADR** documenting the decision.

This ADR is item 4. It also records whether items 1-3 are met as
of v1.0 ship.

## Status of each gate item at v1.0 ship

### 1 — Multi-consumer adoption: NOT YET MET

Per `docs/development/state.md` § Consumers as of v1.0 ship:

| Consumer | Status |
|----------|--------|
| **cyim** | Active. v1.2.0+ has `--regex=<flavor>` threaded for all five niyama flavors. cyim ADR 0002 keeps cyim-side code change at zero. cyim is **consumer #1**. |
| owl | Planned. Pager / cat-class utility; intended uses are filtering and search. Not yet wired. |
| agnoshi | Planned. AI shell; fuzzy flavor (M3.5) for shell completion. Not yet wired. |
| daimon | Planned. Agent orchestration; re2 flavor for DoS-safe pattern gates. Not yet wired. |

**One confirmed long-horizon consumer** (cyim). The fold gate
requires two. v1.0 ships fold-**ready** but does not actually
fold.

### 2 — v1.0.0 reached: ✓ MET (this release)

v1.0 is the release this ADR ships in. Five engines (bre, re2,
pcre, fuzzy, vim) all live. Test surface 661 assertions, fuzz
surface 1689 assertions (final v1.0 numbers are within ±0.5%
expected drift from these). Bench history captured in
`docs/benchmarks.md`.

### 3 — Public surface frozen: ✓ MET (per ADR 0010)

ADR 0010 (Surface freeze) locked the public `niyama_<engine>_*`
API at v0.9.0 ship. v1.0 is the first ship under freeze; no
breaking changes have landed since. The frozen contract:

- All public function signatures.
- All error code numeric values (including reserved-but-unused
  slots).
- All capacity limits (`MAX_*`).
- Semantic invariants (re2/bre/vim no-backref; pcre as the only
  backtracking engine; Pike NFA linear-time guarantee).
- Opcode IDs per engine (encoded in `dist/niyama.cyr`).

Post-v1.0 changes are restricted to additive-only extensions in
cyrius stdlib's vendored `lib/niyama.cyr` once the fold trigger
fires. niyama-the-repo enters maintenance mode at v1.0; v1.x
patches are bug-fix only.

### 4 — Fold ADR: ✓ THIS DOCUMENT

## Triggered consequences (2026-05-06)

The fold trigger fired at cyrius v5.9.0 ship. AGNOS-lineage
consumer gate met by:

1. **cyim** (consumer #1, active — v1.2.0+ has `--regex=<flavor>`
   threaded for all five niyama flavors).
2. **AGNOS bare-metal kernel** (consumer #2 — long-horizon
   confirmed pin queued for cyrius v5.10.0 bare-metal target).

Cyrius v5.9.0 vendored `dist/niyama.cyr` (from this repo's
v1.0.1 tag) byte-identical as `lib/niyama.cyr`. niyama-the-repo
enters fold-maintenance mode: v1.x patches still land here and
propagate via cyrius update; post-fold extensions land in
cyrius stdlib's vendored copy per the original ADR plan.

The v1.0.0 ship had a defect — `dist/niyama.cyr` was the
108-line `include`-manifest scaffold, not a true bundled
artifact. v1.0.1 corrected this by wiring `[lib] modules = [...]`
in `cyrius.cyml` and regenerating via `cyrius distlib` (the
canonical bundling tool). The fold operation vendored from
v1.0.1, not v1.0.0.

## Decision

**v1.0 ships fold-ready but does not fold at v1.0 ship time.**

The fold trigger is **the materialization of a second long-horizon
AGNOS-lineage consumer** wiring niyama at the same level cyim
does (full engine usage, `cyrius.cyml [deps.niyama]` declaration,
shipping a release that depends on niyama). When that happens:

1. The second consumer's release notes (or a follow-up niyama
   patch) adds the consumer to `state.md` § Consumers as
   "Active."
2. A cyrius stdlib release vendors `dist/niyama.cyr` as
   `lib/niyama.cyr` byte-identical (sandhi precedent at cyrius
   v5.7.0).
3. niyama's CHANGELOG records the fold event under
   `[Unreleased]` → next patch release; ADR 0011 status changes
   to "Triggered: <date>" with the cyrius stdlib version that
   vendored.

Until then, niyama remains an out-of-tree dependency. Consumers
needing niyama vendor `dist/niyama.cyr` directly via
`[deps.niyama]` in their `cyrius.cyml`. This is the same model
sandhi used pre-fold.

### Speculative target: cyrius 5.9.0

The v0.9.0 surface freeze + v1.0 ship aligns with the cyrius 5.9.x
release window (current cyrius is 5.8.65 with 5.9.x in planning).
If the second-consumer gate is met during 5.9.x, cyrius 5.9.0 (or
a 5.9.x patch) is the natural fold vehicle.

This is **speculative, not a deadline**. Fold can happen any
cyrius release after the gate is met; if the gate isn't met
during 5.9.x, niyama stays out-of-tree through 5.10.x and beyond.
The decision is consumer-count-driven, not calendar-driven.

## Consequences

- **Positive** — niyama v1.0 is *available* for consumer #2
  whenever it materializes. The artifact (`dist/niyama.cyr`),
  surface (ADR 0010), and audit (`docs/audit/2026-05-05-audit.md`)
  are all in place. The fold itself is a one-line change in
  cyrius stdlib.
- **Positive** — Consumer #1 (cyim) gets a stable v1.0 to pin
  against. The freeze means cyim's `--regex=<flavor>` parser
  doesn't need to track moving niyama versions.
- **Positive** — niyama-the-repo enters maintenance mode at v1.0.
  Future v1.x patches are bug-fix only; the cognitive load drops
  to "respond to bug reports."
- **Negative** — If consumer #2 never materializes, niyama
  doesn't fold. The fold-ready artifact lives in `dist/niyama.cyr`
  forever, vendored directly by cyim and any future consumer, but
  cyrius stdlib never picks it up. Acceptable: niyama is still
  *useful* out-of-tree; folding is a bonus, not a requirement.
- **Negative** — The "trigger" model defers a release decision.
  Some readers prefer "fold at v1.0 by fiat regardless of
  consumer count." Rejected per ADR 0001's explicit
  multi-consumer requirement; folding before the gate is met
  would burden cyrius stdlib with niyama maintenance for one
  consumer.
- **Neutral** — Post-fold extensions (per ADR 0010) land in
  cyrius stdlib's `lib/niyama.cyr`, not in niyama. Examples
  pinned for post-fold:
  - vim backref `\1`-`\9` per ADR 0009 (decision-gated on cyim
    asking).
  - fuzzy `_search_at` per A1 review finding.
  - Long Unicode property names per G3.
  - `(?i)` full case folding per C2 lock.

## Trigger checklist (for future-me / future-reader)

When consumer #2 materializes, the fold operation is:

1. **Update `state.md` § Consumers**: add the consumer with
   "Active" status, link to its repo, note the niyama flavor(s)
   it uses.
2. **Verify v1.0 surface unchanged**: the cyrius stdlib version
   doing the fold reads `dist/niyama.cyr` from the niyama
   v1.0.0 git tag. Any surface drift is a niyama bug, not a
   fold issue.
3. **Cyrius stdlib PR**: vendor `dist/niyama.cyr` as
   `lib/niyama.cyr` byte-identical. Add to `cyrius.cyml`
   `[deps]` resolver mapping. Bump cyrius stdlib version (X.Y.0
   if introducing the new lib; X.Y.Z if backporting).
4. **Update ADR 0011**: status → "Triggered: <date> via cyrius
   <version>". Add a "Triggered consequences" section noting
   what changed at fold time.
5. **niyama CHANGELOG entry** under `[Unreleased]` or next patch
   release: "niyama folded into cyrius stdlib `lib/niyama.cyr`
   at cyrius `<version>`. Out-of-tree consumers may continue
   vendoring `dist/niyama.cyr` from this repo or migrate to the
   stdlib path."
6. **niyama-the-repo enters deeper maintenance**: post-fold
   extensions live in cyrius stdlib's vendored copy, not here.
   Bug fixes still land here and propagate via cyrius update.

## Alternatives considered

- **Fold at v1.0 by fiat, ignoring consumer count.** Rejected
  per ADR 0001's explicit gate. Cyrius stdlib should not carry
  vendored libs for one-consumer use cases; fold is a
  multi-consumer optimization.
- **Defer the fold ADR to "the release that actually folds."**
  Rejected. The fold ADR exists to *unblock* fold once the
  consumer gate is met. Writing it at v1.0 means consumer #2's
  arrival can immediately trigger fold without a delay for ADR
  authoring.
- **Auto-fold post-v1.0 on any second-consumer signal.**
  Rejected. The trigger should be a deliberate cyrius stdlib
  PR, not automatic. The fold ADR's checklist is the manual
  gate.
- **Fold but keep niyama-the-repo active for post-fold
  development.** Rejected. Once stdlib vendors, the stdlib copy
  is the canonical one; niyama-the-repo stays as a maintenance
  fallback for non-stdlib consumers. Active development moves
  to cyrius stdlib per the sandhi precedent.

## Related ADRs

- ADR 0001 — niyama-as-additional-engines-repo, fold lifecycle.
- ADR 0008 — Unicode-stdlib pivot (informs fold dependency on
  cyrius `lib/unicode/`).
- ADR 0009 — bre/vim backref review (post-fold candidate per
  this ADR's trigger).
- ADR 0010 — Surface freeze (gate item 3).
