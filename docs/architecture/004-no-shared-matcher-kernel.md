# 004 — Each engine owns its full matcher; no shared kernel

niyama's four regex engines (bre, re2, pcre, vim) plus fuzzy each
implement their entire matcher independently. There is no shared
"Pike NFA kernel" or "backtracking kernel" module that multiple
engines call into.

## What's true

Each `src/<engine>.cyr` file contains:

- Its own opcode set (`<ENGINE>_OP_*`).
- Its own emit helpers (`_<engine>_emit_raw`, `_emit_op`, `_emit_char`,
  etc.).
- Its own matcher main function (`_<engine>_match_run` or
  `_<engine>_pike_run`).
- Its own class-bitmap helpers (`_<engine>_class_set`,
  `_class_range`, etc.).
- Its own boundary helpers (`_<engine>_at_word_char`).

The only cross-engine sharing is via two utility modules:

- **`src/posix_classes.cyr`** — POSIX bracket-class fillers
  (alpha, digit, etc.) and the `[[:name:]]` keyword recognizer.
  Used by all four regex engines as of v0.9.0.
- **`src/unicode_props.cyr`** — `\p{NAME}` / `\P{NAME}` parser
  and 30-bit GeneralCategory mask lookup. Used by re2, pcre, vim.

These are *utility-level* sharing — they don't run the matcher
loop, they just provide compile-time helpers and runtime
membership tests.

## Why no shared kernel

This was an explicit decision per ADR 0001 (per-engine ABI shape)
and reaffirmed at ADR 0009 ("no new shared kernel" was load-bearing
in the bre/vim backref review).

The reasoning:

1. **Per-engine ABIs are the contract.** Each engine has its own
   `niyama_<engine>_*` API surface and its own error code
   numbering, frozen at v1.0. A shared kernel that the engines
   call into would couple their versioning.
2. **The matcher's complexity is in dispatch + thread management.**
   Extracting "Pike NFA kernel" and giving it to bre, re2, vim to
   share would require a virtual method-table for opcode dispatch
   per engine — lots of indirection cost for code that's already
   thoroughly tested.
3. **The duplication is small in absolute terms.** Each engine
   matcher is 100-300 lines of dispatch code; total cross-engine
   duplication is bounded.
4. **Future-fold compatibility.** Once niyama folds into cyrius
   stdlib `lib/niyama.cyr` (post-v1.0), the stdlib can choose to
   refactor; until then, each `niyama_<engine>_*` surface is
   independently locked. Refactoring inside niyama would risk
   breaking that lock prematurely.

## Consequences for the implementer

- **Bug fixes are per-engine.** A buffer-bounds fix in
  `_pcre_match_run` doesn't automatically apply to re2 or vim; the
  bug needs to be checked separately per engine, and fixed
  separately. The v0.9.0 P(-1) audit (`docs/audit/2026-05-05-audit.md`)
  walked all four engines independently.
- **New features are per-engine.** v0.8.0's `\p{L}` shipped as
  three separate UPROP/NUPROP opcode pairs (one per engine).
  Tedious; bounded.
- **Per-engine optimizations are isolated.** A future post-fold
  optimization in pcre's matcher (e.g. the saves-pool refactor
  flagged at v0.9.0) only changes pcre.

## What's specifically NOT extractable

- **Matcher main loops** — locked.
- **Opcode emit helpers** — locked.
- **Per-engine class-bitmap helpers** (despite duplication
  identified in v0.9.0 review item B1) — locked.
- **Boundary helpers** (B2) — locked.

These all stay duplicated through v1.0 by design.

## What IS extractable (already done)

- POSIX bracket classes → `src/posix_classes.cyr` (v0.7.0).
- `\p{NAME}` parser → `src/unicode_props.cyr` (v0.8.0).

Pattern: utility modules that supply *data tables* and *parsers*
are fine to share. Modules that own *matcher control flow* are
not.

## What changed when

- M0 — ADR 0001 set the per-engine ABI principle.
- M1-M4 — each engine implemented independently.
- v0.7.0 — first utility extraction (POSIX classes).
- v0.8.0 — second utility extraction (unicode_props).
- v0.8.0 — vim folded onto `posix_classes.cyr`. Per-engine
  *duplication* of POSIX class fillers reduced; matcher
  independence preserved.
- ADR 0009 — reaffirmed "no shared kernel" in the context of
  potential post-fold backref support for vim.
- v1.0 freeze — locks the per-engine architecture.
