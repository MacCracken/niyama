# 002 — Save table layout is even/odd indexed

Capture-position bookkeeping in all four regex engines (bre, re2,
pcre, vim) uses a single flat array indexed by `2*N + open_close_bit`.

## What's true

- **`save[2*N]`** = group N's start position (`open` save).
- **`save[2*N + 1]`** = group N's end position (`close` save).
- **Group 0 is the whole match**: `save[0]` = match start,
  `save[1]` = match end. Always populated when a match succeeds.
- **Groups 1..9** are user captures. `save[2..19]` covers them.
- **`MAX_SAVES = 20`** uniformly across bre/re2/pcre/vim. (10
  groups × 2 ends.)

## Why this layout

Pike NFA threads carry their save state by-value (each thread is a
pc + saves snapshot). Backtracking matchers (pcre) snapshot/restore
saves at SPLIT/LOOKAHEAD/LOOKBEHIND/RECURSE boundaries. In both
models, saves are a flat per-thread slot array.

`SAVE 2N` and `SAVE 2N+1` opcodes are emitted at group-open and
group-close points by the parser. The matcher writes
`saves[2N] = pos` (or 2N+1) and continues. No special distinction
between "open" and "close" at runtime — both are byte-position
writes to a slot.

Public API `niyama_<engine>_group_start(nfa, n)` reads
`saves[2*n]`; `_group_end(nfa, n)` reads `saves[2*n + 1]`. Returns
`-1` if the slot was never written (group didn't participate).

## Frozen at v1.0

This layout is part of the public ABI. Adding more capture groups
(beyond 9) would require expanding `MAX_SAVES`; that's a binary
ABI break. Any post-fold extension that grows captures has to
either:

- Accept the v1.0 cap as permanent (recommended — matches PCRE2's
  default), or
- Add a *new* engine name (`niyama_pcre2_*`?) with the larger cap;
  v1.0's `niyama_pcre_*` stays at 9.

## Special slots

- `save[0]` and `save[1]` (match start/end) can be modified by
  pcre's `\K` (reset match start — emits `SAVE 0` mid-pattern) and
  vim's `\zs`/`\ze` (similar — `\zs` emits `SAVE 0`, `\ze` emits
  `SAVE 1`). This is per-engine flavor; the kernel slot semantics
  don't care.

## What changed when

- M1 (bre) — established the layout. MAX_SAVES = 20.
- M2-M4 — re2/pcre/vim adopted unchanged.
- v0.7.0 — `\K` (pcre) and named-capture (re2 + pcre) reused the
  same slots; named lookups go through a parallel name table that
  resolves to the same numeric slot.
- v1.0 freeze — locks at 20.
