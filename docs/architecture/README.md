# Architecture notes

Non-obvious constraints, quirks, and invariants that a reader cannot derive from the code alone. Numbered chronologically — never renumber.

Not decisions (those live in [`../adr/`](../adr/)) and not guides (those live in [`../guides/`](../guides/)). An item here describes *how the world is*, not *what we chose* or *how to do something*.

## Items

- [001 — Matcher position-stepping is asymmetric across engines](001-codepoint-vs-byte-stepping.md) — re2/vim are codepoint-stepped, pcre is byte-stepped, bre is byte-stepped (no Unicode), fuzzy is DP-grid (neither).
- [002 — Save table layout is even/odd indexed](002-save-table-layout.md) — `save[2N]`/`save[2N+1]` for group N start/end; MAX_SAVES = 20 across all four regex engines.
- [003 — Char-class bitmaps are 32 bytes; cover bytes 0–255 only](003-class-bitmap-is-byte-only.md) — Unicode-aware features layer via `UPROP`/`UCHAR_CI` opcodes, not the bitmap.
- [004 — Each engine owns its full matcher; no shared kernel](004-no-shared-matcher-kernel.md) — Per-engine ABI per ADR 0001 + ADR 0009. Utility modules (posix_classes, unicode_props) are the *only* shared layer.
