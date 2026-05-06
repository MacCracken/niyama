# 003 — Char-class bitmaps are 32 bytes; cover bytes 0–255 only

niyama's character-class storage is a fixed 32-byte bitmap per
class. This is intentional and Unicode-aware features are layered
on top via a separate mechanism (`UPROP`/`NUPROP` opcodes).

## What's true

- Each char class compiles to a **32-byte bitmap** (256 bits, one
  per byte value 0–255).
- Bit `c` of byte `c >> 3` set means the class includes byte `c`.
- Per-engine table: `_<engine>_class_base + class_index * 32`.
- **`MAX_CLASSES = 64`** uniformly across bre/re2/pcre/vim. A
  pattern with 65+ distinct char classes errors with `*_E_TOO_LARGE`.
- Class membership testing at match time reads one byte from input
  and one bit from the bitmap. O(1).

## Why bytes (not codepoints)

Codepoint-indexed bitmaps would be 0x110000 bits (~140 KB) per
class — far too large for a stack-bounded engine. Range-based
storage (sorted intervals) was considered and rejected: the O(1)
lookup cost is load-bearing for `re2_search_class`-style hot paths
(it's already the highest per-iteration cost in the v0.8.0 bench
floor).

Patterns that need codepoint-level matching (`\p{L}` etc.) bypass
this bitmap entirely and use the `UPROP` / `NUPROP` opcodes
introduced in v0.8.0, which call `unicode_category()` from the
stdlib at match time. No per-class bitmap, no compile-time
infrastructure cost.

## Consequences

- **`[α-ω]`** as a pattern cannot be expressed via the bitmap —
  the codepoints don't fit. Engines parse it but only the lead
  bytes (0xCE, 0xCF) get into the bitmap, which is *wrong* — the
  pattern won't match what the user expects.
  - **In re2/vim** (codepoint-stepped): the lead byte appears at
    the start of α; CLASS opcode tests against bitmap; advances
    by full UTF-8 length. Sometimes "works" by accident, often
    doesn't.
  - **In pcre** (byte-stepped): same behavior; pattern authors
    must use `\p{...}` for codepoint ranges.
- **POSIX bracket classes `[[:alpha:]]`** — these *are* ASCII-only
  by POSIX definition (matched by the implementation in
  `src/posix_classes.cyr`), so the byte bitmap is the correct
  storage. No issue.
- **`\d`, `\w`, `\s`** — all ASCII-only in niyama through v1.0.
  Stored in the bitmap. Consumers wanting Unicode digits/words
  use `\p{Nd}` / `\p{L}`.

## Frozen at v1.0

The 32-byte bitmap and `MAX_CLASSES = 64` are part of the public
ABI. Patterns hitting either limit error cleanly via
`*_E_TOO_LARGE`. v0.9.0 boundary tests (per-engine) lock this
contract.

## What changed when

- M1 (bre) — established 32-byte bitmap, MAX_CLASSES = 64.
- M2-M4 — re2/pcre/vim adopted unchanged.
- v0.7.0 — `src/posix_classes.cyr` extracted the 12 POSIX class
  fillers as a shared module (still byte-bitmap).
- v0.8.0 — `UPROP`/`NUPROP` (and `UCHAR`/`UCHAR_CI`) opcodes
  bypass the bitmap entirely for codepoint-aware matching.
- v0.9.0 — re2 wired into `posix_classes.cyr` (was missed in
  v0.7.0 carve, fixed in v0.9.0 P(-1) review).
- v1.0 freeze — locks layout + limits.
