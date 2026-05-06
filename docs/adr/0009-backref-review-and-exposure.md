# 0009 — bre / vim backref `\1`-`\9`: review + exposure surface

**Status**: Accepted
**Date**: 2026-05-05

## Context

niyama_bre and niyama_vim have rejected backreferences `\1`-`\9` at
compile time since they shipped:

- **niyama_bre** (M1 / ADR 0002): `BRE_E_BACKREF_UNSUPPORTED = 2`,
  documented as "potentially post-v1.0" — POSIX BRE *does* spec
  backrefs, so this is a deliberate scope cut, not a feature gap.
- **niyama_vim** (M4 / ADR 0006): `VIM_E_BACKREF_UNSUPPORTED = 2`,
  same shape. Pike NFA can't natively handle backrefs (they're not
  regular), and adding backtracking to vim's M4 carve was out of
  scope.
- **niyama_re2** (M2 / ADR 0003): `RE2_E_BACKREF_UNSUPPORTED = 2`
  by *guarantee* — re2's "linear-time on untrusted input" promise
  forbids backrefs structurally. This stays.
- **niyama_pcre** (M3 / ADR 0004): backref **is** supported via the
  backtracking matcher; consumers needing backref already have it.

The question pinned at v0.9.0 (per ADR 0008's review-not-implement
framing): given that pcre provides backref, does bre/vim need it?
And if it lands, what's the exposure surface that ships with it?

This ADR is the review. It does not commit to implementation; it
lays out the options, costs, and a recommendation. Acceptance of
this ADR is what decides the path.

## Options

Three viable paths, plus the documented "no" path.

### Option A — Implement at v0.9.0 via per-engine backtracking fallback

bre and vim each gain a new backref opcode + matcher extension.
When a pattern contains `\1`-`\9`, the engine switches from its
Pike NFA matcher to a backtracking variant. Patterns without
backref keep the linear-time path.

**What changes per engine:**
- New opcode `BRE_OP_BACKREF` / `VIM_OP_BACKREF` (mirrors
  `PCRE_OP_BACKREF = 12`).
- Compile-time flag set when any backref opcode is emitted.
- New backtracking matcher path (likely ~150 lines per engine,
  borrowing pcre's matcher structure).
- Step-limit infrastructure (currently pcre-only) extends to
  bre/vim — `niyama_bre_set_step_limit()`,
  `niyama_vim_set_step_limit()`, plus `last_step_count` accessors.
- The rejection error code (`*_E_BACKREF_UNSUPPORTED = 2`) becomes
  reserved-but-unused per the ABI freeze rule (same shape as
  `PCRE_E_CONDITIONAL_UNSUPPORTED = 5` after v0.7.0).

**Pros:**
- Honors the M1 / M4 "potentially post-v1.0; document, don't skip"
  language. Closes the question with code rather than a deferral.
- POSIX BRE conformance — `niyama_bre` can now compile every
  POSIX-spec pattern.
- vim flavor parity with vim itself (vim supports `\1`-`\9` in
  search patterns and substitution).

**Cons:**
- Adds a *second* matcher per engine. bre and vim become two-matcher
  engines (Pike NFA + backtracking), doubling the audit surface
  right before M5 freeze.
- New runtime API surface (`set_step_limit`, `last_step_count`)
  lands per engine — a non-trivial ABI extension shipping right
  before v1.0 lock.
- Catastrophic-backtracking patterns now compile in bre/vim —
  consumers must opt into step-limit guards explicitly. Default
  step-limit semantics need to be decided.
- Code duplication across bre + vim + pcre for backtracking
  fundamentals. No shared "Pike NFA kernel" or "backtracking
  kernel" exists today; introducing one is itself a project.

### Option B — Document permanently out-of-scope for v1.0

Backref stays rejected in bre and vim through v1.0 fold. The
rejection error codes are *not* reserved-and-renamed — they remain
as-named (`*_E_BACKREF_UNSUPPORTED`), correctly describing the
permanent state. ADR 0002 / ADR 0006 get amended footnotes pointing
to this ADR.

**What changes:**
- ADR 0002 footnote: "v0.9.0 review concluded backref is permanently
  out of scope for niyama_bre. Consumers needing BRE-flavor backref
  must use niyama_pcre with a BRE-emulating pattern, or wait for
  post-fold."
- ADR 0006 footnote: same for vim.
- The flavor-selection rubric in `state.md` gets an explicit
  "want backref + bre/vim flavor → use pcre with the equivalent
  pattern" row.
- No code changes. `*_E_BACKREF_UNSUPPORTED = 2` keeps its name and
  meaning.

**Pros:**
- Smallest surface area before M5 freeze. Zero new code, zero new
  API extensions, zero new runtime risk.
- Honest about niyama's positioning: bre/vim are linear-time-only;
  backref users → pcre. The flavor split becomes more meaningful.
- re2's guarantee preservation is *structurally* identical to
  bre/vim's, which is conceptually clean.
- Doesn't preclude post-fold addition. cyrius stdlib (post-fold)
  can add backref later without breaking niyama's frozen surface
  — niyama's freeze just locks "what shipped at v1.0".

**Cons:**
- Walks back the "potentially post-v1.0" language. The
  "potentially" had a hopeful edge; this ADR replaces it with
  "no, never (within v1.0 scope)".
- Loses POSIX BRE conformance. `niyama_bre` ships as
  POSIX-BRE-minus-backrefs forever (or at least until post-fold).
- Some consumers (cyim's `--regex=bre` users with backref-heavy
  patterns) get a worse error message: "use pcre" rather than
  "your pattern works."

### Option C — Defer to post-fold (v1.x or via cyrius stdlib)

Same as Option B for v1.0 — backref stays rejected — but the
permanent-out-of-scope claim is replaced with "post-fold revisit".
Once cyrius stdlib vendors `lib/niyama.cyr` (sandhi precedent),
backref support can be added there without disrupting niyama's
v1.0 surface.

**What changes:**
- ADR 0002 / 0006 footnote: "v0.9.0 review deferred backref to
  post-fold. cyrius stdlib's `lib/niyama.cyr` may add backref
  support in a post-vendor minor version; the niyama v1.0 ABI
  remains backref-rejecting."
- The fold ADR (template: sandhi ADR 0002) carries a "post-fold
  capability gaps" section listing backref.

**Pros:**
- Preserves the "potentially" hope without committing to v1.0.
- Defers complexity past the freeze gate, when the surface is
  more flexible (stdlib evolves more freely than niyama-the-repo).
- Honest pre-v1.0 state: "we considered, we declined for v1.0,
  the door isn't shut."

**Cons:**
- Post-fold path is speculative — niyama might never fold
  (v1.0 fold gate requires ≥2 long-horizon consumers; not yet met).
  If fold doesn't happen, "post-fold backref" never happens either.
- The footnote-then-defer pattern is what ADR 0007 originally
  warned against ("floating deferrals"). The v0.8.x ladder rule
  exists specifically to avoid this. Option C re-introduces a
  floating defer.

### Option D — Don't decide; keep the v0.9.0 pin open longer

Move the review pin to v0.9.x or beyond, keep the question
floating with new context.

**Pros:**
- Buys time to consult with consumers (cyim especially — does it
  actually want bre-flavor backref?).
- No ADR commitment now.

**Cons:**
- Defeats the purpose of pinning. If v0.9.0 also collapses without
  a decision, we're at v0.9.x with the same unresolved question
  going into M5 freeze.
- The decision has to land before v1.0 surface freeze regardless.
  Procrastinating just shifts which release does the work.

## Exposure surface (if Option A lands)

Captured here in detail because it's the load-bearing comparison
across options. Options B/C/D ship none of this; Option A ships
all of it.

### ABI extensions

**Per engine** (bre + vim, each):

```
# New opcode (compile + match):
*_OP_BACKREF                    # arg1: capture index 1..9

# New compile-time flag (internal):
_<engine>_has_backref           # set when any backref is emitted

# New runtime ABI (mirrors pcre):
niyama_<engine>_set_step_limit(n)       # default 1_000_000
niyama_<engine>_last_step_count()       # observability
```

**New error codes** (next-available slot per engine; existing
codes unchanged per ABI freeze rule):

```
BRE_E_BACKREF_INDEX_OOB         # \5 in a 3-group pattern
VIM_E_BACKREF_INDEX_OOB
```

`*_E_BACKREF_UNSUPPORTED = 2` becomes reserved-but-unused; comment
in source records this. (Consumers checking `last_error == 2` for
backref-rejection now get a different signal: compile succeeds
but a runtime path triggers — the error semantics shift.)

### Match-time semantics

Decisions to make explicit (PCRE2-compatible defaults shown):

- **Unparticipated group**: `(a)|(b)\2` against "a" — group 2
  never captured. PCRE2 semantics: backref to unparticipated group
  fails the alternative. Same here.
- **Self-reference inside the group**: `(\1foo)` — backref to
  group while group is being captured. PCRE2 semantics: matches
  empty (since save-end hasn't fired). Same here.
- **Out-of-range**: `\5` when only 3 groups defined. **Reject at
  compile** with `*_E_BACKREF_INDEX_OOB`. (PCRE2 also rejects, but
  with a different error name.)
- **Case-insensitive interaction**: bre has no `(?i)`; vim's
  `\c`/`\C` is per-pattern not implemented in niyama. So this
  doesn't surface in v0.9.0. Future extension if `\c` lands.

### Step-limit semantics (new for bre/vim)

Currently pcre owns step-limit. Adding to bre/vim raises questions:

- **Default step-limit**: 1_000_000 (matching pcre).
- **Behavior on exceed**: pcre returns "no match" (per ADR 0004's
  "honest mismatch" framing). bre/vim should match.
- **Patterns without backref**: are still Pike-NFA-matched and
  thus step-limit-irrelevant. Step counter only increments on the
  backtracking path. Documented invariant.

### Consumer impact

- **cyim** (`--regex=bre`, `--regex=vim`): backref patterns now
  compile and match. cyim's parser-side wiring already exists
  (one elif arm per cyim ADR 0002); the engine-side change is
  transparent.
- **owl, agnoshi, daimon**: backref usage unclear. Probably no
  immediate impact (these consumers haven't asked for backref).
  daimon specifically wants re2 — re2 stays no-backref, so
  daimon is unaffected.
- **The flavor-selection rubric** in `state.md` simplifies one
  row — "Backref" no longer routes to pcre exclusively. (But
  pcre still has named backrefs, branch-reset, conditional, etc.
  that bre/vim don't.)

## re2 guarantee preservation (all options)

Whatever this ADR decides, **re2 keeps rejecting backref**. The
linear-time guarantee is a load-bearing API contract per ADR 0003.
This is enforced by per-engine policy: re2's parser keeps the
explicit `RE2_E_BACKREF_UNSUPPORTED = 2` rejection branch. There
is no shared kernel today; the rejection is structurally local to
re2.cyr.

If/when a "Pike NFA kernel" gets refactored out (post-fold, maybe),
the kernel itself can grow optional backref support — re2 just
keeps its parser-level rejection. cyim ADR 0002's "capability
flag" pattern works here: kernel says "I can do backref"; engine
says "but I won't expose it."

## Recommendation

**Asymmetric split: bre Option B (permanently out), vim Option C
scoped to post-fold-via-cyrius-stdlib.**

The two engines have different consumer profiles and the decision
honors that.

### bre — permanently out of scope for v1.0 *and* post-fold

Direct `niyama_bre` users wanting backref migrate to `niyama_pcre`
with the equivalent pattern. POSIX BRE minus backref is bre's
permanent position. ADR 0002 gets a footnote pointing here.

Reasoning:
1. No identified bre consumer with a hard backref need.
2. POSIX BRE → PCRE pattern translation is mechanical for backref;
   no syntax-quirk loss the way vim has.
3. "POSIX BRE minus backref" is an honest niyama positioning point —
   bre is the no-frills linear-time POSIX-compatible flavor. Adding
   backref dilutes that.

### vim — out of scope for v1.0; explicit post-fold revisit via cyrius stdlib

`niyama_vim` ships v1.0 without backref. Post-fold (cyrius stdlib
vendors `lib/niyama.cyr` per the sandhi precedent), the stdlib
version *may* add backref as a post-fold extension. ADR 0006 gets a
footnote pointing here. The fold ADR (template: sandhi ADR 0002)
carries a "post-fold capability gaps" entry listing this.

Reasoning:
1. **cyim asymmetry.** cyim is "vim-in-cyrius with potential
   enhancements over time" — a load-bearing long-horizon consumer
   whose product positioning genuinely benefits from vim-flavor
   backref. Telling cyim users "use pcre flavor" loses the muscle
   memory that motivated picking vim flavor in the first place.
2. **vim flavor isn't pcre-translatable.** vim metachars (`\zs`,
   `\ze`, magicness modes, `\<`/`\>`) don't all map cleanly to
   pcre. A vim → pcre rewrite for backref users isn't free.
3. **Complexity is guardable** — see Containment Design below.
4. **The fold gate is the natural release vehicle** for this
   addition. Post-fold means the cyrius stdlib `lib/niyama.cyr`
   is the artifact carrying the extension; niyama's v1.0 frozen
   ABI stays narrow.

### re2 — structurally backref-free, permanent

re2 keeps rejecting `\1`-`\9` at parse time. The linear-time
guarantee is a load-bearing API contract per ADR 0003 (DoS-safety
on untrusted input). re2.cyr documents this as a permanent
invariant; no future change reverses it. Enforced by per-engine
policy — cyim ADR 0002's "capability flag" pattern.

## Containment design (post-fold vim extension shape)

If/when vim backref lands post-fold via cyrius stdlib, the
implementation must satisfy these constraints — they bound the
long-term maintenance surface:

1. **vim-only.** bre stays rejecting per the bre decision above.
   No code in bre.cyr changes ever for backref reasons.
2. **Either-or matcher dispatch in vim.** Parser sets
   `_vim_has_backref` flag when emitting `VIM_OP_BACKREF`. Matcher
   checks once at run start: flag clear → existing Pike NFA path,
   untouched. Flag set → backtracking path. The two paths don't
   share state mid-run, just data structures (capture array).
3. **No new shared kernel.** Don't try to extract a "Pike NFA
   kernel" or "backtracking kernel" module across engines.
   Per-engine duplication of step-limit infra (vim copy-pastes
   pcre's API shape) is the lesser evil — bounded surface, no
   cross-engine coupling. cross-engine coupling was the explicit
   anti-goal in ADR 0001.
4. **Step-limit ABI mirrors pcre exactly.**
   `niyama_vim_set_step_limit(n)` (default 1_000_000),
   `niyama_vim_last_step_count()`. New error code at next-available
   slot. `VIM_E_BACKREF_UNSUPPORTED = 2` becomes reserved-but-unused
   per ABI freeze rules (consumer-visible meaning shifts; document
   in CHANGELOG of the post-fold release).
5. **re2 stays structurally backref-free.** Per the bre/re2
   sections — re2 invariant is permanent regardless of vim's path.

Match-time semantics decisions (PCRE2-compatible defaults):
- Unparticipated group → backref fails the alternative.
- Self-reference inside its own group → matches empty.
- Out-of-range (`\5` when 3 groups defined) → reject at compile
  with the new index-OOB error code.

These are the answers a future implementer of the post-fold
extension reads to know what shape the niyama-side review locked
in. Not commitments to ship.

## Consequences

- **Positive** — niyama v1.0 freezes with a smaller, more
  auditable surface. Zero new matcher code, zero new ABI extensions,
  zero new attack surface before M5 hardening.
- **Positive** — Flavor positioning sharpens. re2 + bre + vim are
  the linear-time family; pcre is the dedicated backtracking
  engine. Honest split.
- **Positive** — cyim's potential vim-flavor backref need has a
  documented future path (post-fold via stdlib) without forcing
  niyama v1.0 to carry it.
- **Negative** — Walks back the "potentially post-v1.0" language
  from ADR 0002 / ADR 0006 *partially*. ADR 0006's "potentially"
  survives via the post-fold path; ADR 0002's doesn't.
- **Negative** — bre direct users wanting backref must switch to
  pcre flavor. One config change in cyim; mechanical pattern
  translation for raw consumers.
- **Negative** — Post-fold path is contingent on fold happening,
  which gates on ≥2 long-horizon consumers (cyim is #1; #2 not
  yet identified). If fold doesn't happen, vim never gets backref.
  This is disclosed up front, not a broken promise.
- **Neutral** — `BRE_E_BACKREF_UNSUPPORTED = 2` and
  `VIM_E_BACKREF_UNSUPPORTED = 2` keep their current names and
  meanings. No ABI churn from this ADR.

## Alternatives considered

Covered as Options A / C-symmetric / D above. Beyond those, two
patterns were considered and discarded outright:

- **"Borrow pcre's matcher" cross-engine** — bre/vim with backref
  dispatch into pcre.cyr's matcher. Discarded because it couples
  engines architecturally; pcre.cyr would need to be
  partially-imported as a sub-module by bre.cyr / vim.cyr, which
  the per-engine-ABI principle (ADR 0001) rejects. Engines stay
  independent.
- **"Compile bre/vim backref patterns to pcre at compile time"** —
  rewrite the user's pattern into an equivalent pcre pattern and
  match with pcre. Discarded because it leaks the rewrite to
  error messages (consumer sees pcre errors for a bre/vim pattern),
  breaks the per-engine error-code ABI, and the vim → pcre
  rewrite isn't always clean.

The asymmetric bre/vim split is itself an alternative considered
and chosen — symmetric Option B (both bre and vim permanently out)
was the original recommendation; the split landed after surfacing
that vim has a load-bearing consumer (cyim) bre doesn't.

## Resolved questions

- **Does any consumer need bre-flavor or vim-flavor backref?**
  cyim has the strongest potential need for vim-flavor (muscle
  memory + product positioning). bre has no identified consumer
  need beyond POSIX-spec conformance. Resolution: asymmetric split
  as above.
- **POSIX BRE conformance positioning?** "POSIX BRE minus
  backref" is the chosen honest position. niyama_bre's flavor
  identity is linear-time-only.
- **Post-fold door for vim?** Explicitly open for vim, explicitly
  closed for bre. Asymmetric on purpose; the fold ADR carries the
  "post-fold capability gaps" entry.
