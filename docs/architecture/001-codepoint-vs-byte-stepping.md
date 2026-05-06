# 001 — Matcher position-stepping is asymmetric across engines

niyama's five engines do not all step through input the same way.
This is intentional and load-bearing for the v1.0 frozen surface.

## What's true

- **re2 (Pike NFA)** — outer matcher loop is **codepoint-stepped**.
  At each iteration: decode UTF-8 at `pos` to get `(cp, step)`,
  dispatch all active threads, advance `pos = pos + step`. ASCII
  inputs give `step = 1` (no behavior change vs. byte-stepping);
  multi-byte UTF-8 inputs advance by full codepoint length.
- **vim (Pike NFA)** — same as re2. Codepoint-stepped at v0.8.0.
- **pcre (backtracking)** — outer matcher is **byte-stepped**.
  Each opcode controls its own advance: `CHAR`/`CLASS` advance by
  1 byte; `UPROP`/`NUPROP`/`UCHAR_CI` decode UTF-8 inline and
  advance by full codepoint length.
- **bre (Pike NFA)** — **byte-stepped**. No Unicode-aware opcodes;
  not codepoint-stepped (would be a no-op given its feature set).
- **fuzzy (Levenshtein DP)** — neither stepped. The DP iterates
  over byte indices in a 2D grid; no opcode dispatch at all.

## Why the split (load-bearing detail)

re2 and vim were converted to codepoint-stepping in v0.8.0 to
support `\p{L}` (Unicode property classes) without re-decoding
UTF-8 per opcode. The Pike NFA's "thread queue at each position"
model works naturally with codepoint-stepping — schedule the
successor thread at `pos + step` and the queue handles the rest.

pcre's backtracking matcher, by contrast, is recursive: each
opcode call is an explicit step, so it's natural for each opcode
to decide its own advance. Codepoint-stepping the pcre outer loop
would conflict with `LOOKAHEAD` / `LOOKBEHIND` / `RECURSE`
sub-match semantics (those need to start at exact byte positions
that may not be on codepoint boundaries — e.g. `\K` reset).

bre has no Unicode-aware features, so codepoint-stepping its
matcher would buy nothing but add code.

fuzzy operates on bytes by design — the Levenshtein metric is
defined on bytes (with `FUZZY_FLAG_UNICODE_NFD` normalizing the
input upstream).

## Consequences for consumers

- **ASCII inputs** behave identically across all engines. The
  asymmetry is invisible.
- **Multi-byte UTF-8 inputs**:
  - In re2/vim: `.` matches one full codepoint. `[a-z]` against
    `α` (multi-byte) doesn't match (the lead byte 0xCE isn't in
    `a-z`); position advances by 2 bytes regardless. `\p{L}`
    matches.
  - In pcre: `.` matches one byte (lead byte of α, then matcher
    is mid-codepoint at next position). `\p{L}` properly decodes
    and matches. Pattern authors mixing `.` with multi-byte
    input need to use `(?u)`-style codepoint-aware patterns
    (which niyama doesn't provide; cross-pattern with `\p{L}` or
    explicit byte ranges is the workaround).

The flavor-selection rubric in `docs/development/state.md`
documents the consumer-visible side of this split.

## What changed when (for the future reader)

- v0.6.0 — vim shipped byte-stepped, no Unicode opcodes.
- v0.7.0 — no stepping change; all engines byte-stepped still.
- v0.8.0 — re2 + vim flipped to codepoint-stepped to back `\p{L}`.
  The `UCHAR` (multi-byte literal) opcode landed alongside to
  prevent the regression where pattern `α` would split into two
  CHAR opcodes that the codepoint-stepped matcher couldn't reunite.
- v1.0 freeze — locks all five engines as above.
