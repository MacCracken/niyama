# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

## [0.7.0] — 2026-05-03

M4.5 first-of-three catch-up release: the no-Unicode-dep slice. Per
ADR 0007, eight features land across bre/re2/pcre, all sharing one
small new module `src/posix_classes.cyr`. Lookbehind + pcre
recursion remain deferred to v0.8.0; Unicode work to v0.9.0.

### Added

- **`src/posix_classes.cyr`** — shared ASCII fillers + name
  recognizer for the 12 POSIX bracket classes (`alpha`, `digit`,
  `space`, `upper`, `lower`, `alnum`, `blank`, `cntrl`, `graph`,
  `print`, `punct`, `xdigit`). Used by bre + pcre. (vim still
  carries an in-engine copy from M4 — folding is a v0.9.0
  cleanup.)
- **niyama_bre**:
  - GNU `\<` / `\>` word boundaries with **strict** semantics —
    distinct from `\b`. Two new opcodes `BRE_OP_WORDBEGIN` (= 10)
    and `BRE_OP_WORDEND` (= 11). Matches `grep -G` traditional
    behavior.
  - POSIX bracket classes `[[:alpha:]]` etc. via the shared
    module. Unknown class names → `BRE_E_SYNTAX`.
- **niyama_re2**:
  - Named captures `(?<NAME>...)` and `(?P<NAME>...)` with
    `niyama_re2_group_by_name(nfa, name)` lookup. Mirrors the
    pcre name-table mechanism (40-byte slots, 9-name max).
  - Inline flags `(?i)`, `(?m)`, `(?s)` and combinations
    (e.g. `(?ims)`). Pattern-wide effect from declaration onward.
  - Four new opcodes — `RE2_OP_CHAR_CI` (= 12), `RE2_OP_BOS`
    (= 13), `RE2_OP_EOS` (= 14), `RE2_OP_ANY_NONL` (= 15).
  - New error code `RE2_E_DUPLICATE_NAME = 7` (next-available;
    M2-frozen codes 0..6 unchanged).
- **niyama_pcre**:
  - POSIX bracket classes via the shared module.
  - Inline flags `(?i)`, `(?m)`, `(?s)` + combinations.
  - `\K` reset-match-start — emits `SAVE 0`, overrides implicit
    match-start save (same mechanism as vim's `\zs`).
  - Branch-reset groups `(?|...)` — alternatives reuse capture
    numbers.
  - Conditional patterns `(?(N)yes|no)` and `(?(<NAME>)yes|no)`
    via new opcode `PCRE_OP_COND` (= 22). Matcher consults saves
    array at runtime to dispatch.
  - Callouts `(?C)` and `(?C<num>)` — observability only via new
    opcode `PCRE_OP_CALLOUT` (= 23) and
    `niyama_pcre_last_callout()` accessor. No callback API.
  - Six new opcodes total (`PCRE_OP_CHAR_CI` = 18,
    `PCRE_OP_BOS` = 19, `PCRE_OP_EOS` = 20,
    `PCRE_OP_ANY_NONL` = 21, `PCRE_OP_COND` = 22,
    `PCRE_OP_CALLOUT` = 23).
  - New error code `PCRE_E_BAD_CONDITION = 8` for unrecognized
    condition forms or unknown named references in
    `(?(...)...)`. The `PCRE_E_CONDITIONAL_UNSUPPORTED = 5`
    slot is **frozen but no longer emitted** — kept reserved per
    ABI freeze rules.
- ADR 0007 — records the v0.7.0 carve, what's in vs. deferred,
  the strict-default behavior change, and the cross-engine
  uniformity gains.
- Tests — 121 new assertions across `tests/{bre,re2,pcre}.tcyr`:
  bre word boundaries (9), bre POSIX classes (28), re2 named
  captures (12), re2 inline flags (12), re2 strict defaults (3),
  pcre POSIX classes (8), pcre inline flags (12), pcre strict
  defaults (3), pcre `\K` (4), pcre branch-reset (8), pcre
  conditional (10), pcre callout (8), plus a handful of supporting
  assertions. Aggregate test count: 362 → 483.
- Fuzz — 22 new assertions across
  `fuzz/{bre,re2,pcre}.fcyr`: extended pattern alphabets to
  exercise the new parser branches (POSIX bracket bodies, inline
  flags, named captures, branch-reset, conditional, callout) and
  invariant checks on duplicate-name rejection (re2 + pcre) and
  bad-condition rejection (pcre). Aggregate fuzz count: 1636 →
  1658.

### Changed

- **niyama_re2 BEHAVIOR CHANGE** — `^`, `$`, and `.` are now
  **strict-by-default** (RE2-spec compliant). `^` matches only at
  pos 0; `$` only at pos len; `.` excludes `\n`. Multi-line
  semantics require `(?m)`; dot-matches-newline requires `(?s)`.
  Pre-v0.7.0 implementations were silently loose. Existing tests
  didn't exercise distinguishing inputs (no `\n`), so no test
  regressed; downstream consumers relying on loose semantics will
  see the change. Pre-surface-freeze (M5) is the right time to
  fix this; M5 freezes semantics until v1.0.
- **niyama_pcre BEHAVIOR CHANGE** — same strict-default fix as re2,
  for the same reason. PCRE-spec compliant.
- niyama_bre's bracket parser routed through the shared
  `_posix_match_class_name` recognizer instead of falling through
  to syntax error on `[[:` prefixes.
- niyama_pcre's bracket parser routed through the shared
  `_posix_match_class_name` recognizer; previously rejected
  POSIX-class brackets with `PCRE_E_SYNTAX`.
- `RE2_NFA_HEADER_SIZE` extended 216 → 256 to make room for the
  name-table offset/count words at offsets 216/224.
- `dist/niyama.cyr` bundle now includes
  `src/posix_classes.cyr` ahead of the engine modules.

### Notes

- niyama_bre and niyama_vim **keep their pre-v0.7.0 loose
  defaults** for `^`/`$`/`.` (vim is loose by spec; POSIX BRE
  matches `grep` traditional behavior). The strict-default fix is
  re2/pcre-only.
- Backreferences `\1`-`\9` in bre and vim **remain rejected**
  per the explicit M1/M4 "potentially post-v1.0; document, don't
  skip" calls. Not revisited in v0.7.0.
- `(?-i)` negated inline flags and `(?i:...)` scoped forms are
  out of scope; the `(?ims)` pattern-wide spelling covers ~95%
  of real-world use.

## [0.6.0] — 2026-05-03

M4 — fifth and final pre-catch-up engine: vim (`niyama_vim_*`).
vim/cyim flavor with all four magicness modes, `\<`/`\>` word
boundaries, `\zs`/`\ze` match-position markers, and POSIX bracket
classes `[[:alpha:]]` etc. Pike NFA matcher (fork of re2) — `\1`-`\9`
backref rejected by default per ADR 0006, flagged for v0.9.0 review
alongside the bre backref question.

### Added

- `src/vim.cyr` — Pike NFA matcher with vim-flavor parser. ~1100
  lines. Mode-dependent character dispatch covers all four magicness
  variants without duplicating engine logic.
- Public ABI mirroring the established per-engine shape:
  - `niyama_vim_compile(pat)` — defaults to `VIM_MODE_MAGIC`.
  - `niyama_vim_compile_opts(pat, mode)` — explicit mode flag.
  - `niyama_vim_match` / `_search` / `_search_at` /
    `_group_start` / `_group_end` / `_last_error`.
- **All four magicness modes** as opts-flag-controlled values:
  - `VIM_MODE_VERY_MAGIC` (= 0): all metachars special bare.
  - `VIM_MODE_MAGIC` (= 1, default): `*` `.` `[` special bare;
    `\(` `\|` `\+` `\?` `\=` `\{` `\}` need backslash.
  - `VIM_MODE_NOMAGIC` (= 2): only `^` `$` special bare; `\.` `\*` `\[`
    for those.
  - `VIM_MODE_VERY_NOMAGIC` (= 3): nearly everything literal; `\^` `\$`
    needed even for anchors.
- vim feature set: `\<`/`\>` word boundaries, `\zs`/`\ze`
  match-position markers, `\d`/`\D`/`\w`/`\W`/`\s`/`\S` predefined
  classes, brace quantifiers `\{n,m\}` greedy AND `\{-n,m\}` lazy
  (vim's lazy syntax), `\+` `\?` `\=` quantifiers (and bare `+` `?`
  `=` in very-magic), `\(...\)` groups (and `(...)` in very-magic),
  `\|` alternation (and `|` in very-magic), bare/escape-flipped
  forms throughout.
- POSIX bracket classes inside `[...]`: `[[:alpha:]]`,
  `[[:digit:]]`, `[[:space:]]`, `[[:upper:]]`, `[[:lower:]]`,
  `[[:alnum:]]`, `[[:blank:]]`, `[[:cntrl:]]`, `[[:graph:]]`,
  `[[:print:]]`, `[[:punct:]]`, `[[:xdigit:]]`. Implementation
  shared with v0.9.0 catch-up for bre/pcre/re2.
- `\zs` resets match-start (overrides implicit start-save). `\ze`
  freezes match-end (parser tracks `_vim_saw_ze` to suppress
  implicit final SAVE 1 so `\ze` wins).
- `\1`-`\9` backreferences **rejected at compile** with
  `VIM_E_BACKREF_UNSUPPORTED`. Decision flagged for v0.9.0 review
  alongside the bre backref question.
- ADR 0006 — niyama_vim engine ABI, matcher architecture, and
  scope. Records the Pike-NFA-not-backtracking decision, the
  four-mode opts surface, the `\zs`/`\ze` SAVE-emission strategy,
  and the deferral list (mid-pattern mode switching, vim's vast
  `\X` escape menagerie, replacement-language helpers).
- `tests/vim.tcyr` — 88 unit tests across 13 groups: per-mode
  parser semantics (×4 modes), `\<`/`\>` boundaries, `\zs`/`\ze`
  match-position semantics, all 12 POSIX bracket classes,
  predefined classes, backref rejection, anchors, invalid mode
  rejection, lazy brace, DoS-resistance.
- `fuzz/vim.fcyr` — 219-assertion harness with mode-coverage sweep
  (every random pattern exercised across all 4 modes), rejection
  invariants (backref, bad mode), and a linear-time adversarial
  pattern.
- `tests/vim.bcyr` — bench harness across all four modes plus
  vim-feature benches.

### Performance floor (M4, x86_64, cyrius 5.8.42)

- `vim_compile_*` (per mode): 3-5 μs.
- `vim_search_magic` / `_very_magic` / `_nomagic` / `_very_nomagic`
  (3-way alt over 75-byte text): 7-8 μs across all modes.
- `vim_search_zs_ze` (`foo\zsbar\zebaz`): ~3 μs.
- `vim_search_posix` (`[[:alpha:]]\+`): ~3 μs.
- `vim_search_word_bound` (`\<word\>`): ~3 μs.

### Changed

- `dist/niyama.cyr` — bundle now includes `src/vim.cyr` alongside
  the four prior engines. `NIYAMA_VERSION` → `"0.6.0"`.
- `src/main.cyr` smoke banner reflects M4 status.

### Deferred (not in M4 — see ADR 0006)

- Mid-pattern mode switching (`\v` / `\m` / `\M` / `\V` mid-pattern).
  Opts-flag-only entry in M4; mid-pattern switching is a v0.9.0
  candidate if asked.
- vim's vast `\X` escape menagerie (`\a`, `\A`, `\l`, `\L`, `\u`,
  `\U`, `\x`, `\X`, `\o`, `\O`, `\h`, `\H`, etc.) — niyama_vim
  ships standard `\d`/`\w`/`\s`; vim-specific extras are post-v1.0
  unless asked.
- Replacement-language helpers (`~`, `&`, `\u`/`\l`/`\U`/`\L`).
  Replacement is a consumer concern; cyim handles its own.

### Roadmap reorg (2026-05-03)

Consolidated 11 features that prior ADRs (0003 / 0004 / 0005) had
deferred unilaterally as "post-v1.0" or "M3.5 candidate" into a new
v0.9.0 (M4.5) catch-up milestone. The rationale: deferrals that
silently shrink what ships in v1.0 are scope decisions belonging to
the user, not to the engine implementer. The v0.9.0 catch-up release
clears those deferrals before the M5 freeze, so the surface that
gets frozen is the surface the roadmap originally promised.

Items consolidated:

- **niyama_re2**: named captures, Unicode property classes `\p{L}`,
  inline flags `(?i)/(?m)/(?s)`.
- **niyama_pcre**: lookbehind, `\p{L}`, POSIX bracket classes,
  recursion, conditional patterns, inline flags, branch-reset
  groups, callouts, `\K`.
- **niyama_bre**: GNU `\<`/`\>` word boundaries, POSIX bracket
  classes (shared implementations with pcre/vim).
- **niyama_fuzzy**: Unicode NFD normalization, exact start-position
  recovery in `_search`.

`niyama_bre` `\1`-`\9` backreferences remain per the user's M1
explicit "potentially post-v1.0; document, don't skip" call —
v0.9.0 is a natural revisit point but not a commitment.

ADRs 0002 / 0003 / 0004 / 0005 updated to point their deferral
sections at v0.9.0; `docs/development/roadmap.md` adds the M4.5
milestone; `docs/development/state.md` updates the "Next" sequencing.
No code changes; no shipped engines affected.

## [0.5.0] — 2026-05-03

M3.5 — fourth engine: fuzzy (`niyama_fuzzy_*`). The one engine in
niyama that isn't regex — Levenshtein edit-distance matching for
shell completion, fuzzy-name lookup, and typo-tolerant command
matching. Per ADR 0005.

### Added

- `src/fuzzy.cyr` — Wagner–Fischer Levenshtein DP with two-row
  optimization. ~300 lines (smallest engine in niyama). Three
  match modes via three named functions.
- Public ABI mirroring niyama_bre / niyama_re2 / niyama_pcre per
  ADR 0002, plus fuzzy-specific options:
  - `niyama_fuzzy_compile(pat)` — default options (max_edits=2).
  - `niyama_fuzzy_compile_opts(pat, max_edits, flags)` — full opts.
  - `niyama_fuzzy_match(h, s)` — anchored full-string fuzzy match.
  - `niyama_fuzzy_search(h, s)` — substring-fuzzy: best contiguous
    slice of `s` within edit distance.
  - `niyama_fuzzy_search_prefix(h, s)` — prefix-fuzzy: pattern is
    a typo-tolerant prefix of `s`. The shell-completion shape.
  - `niyama_fuzzy_distance(h, s)` — full-string distance.
  - `niyama_fuzzy_last_distance()` — distance from the last match
    call. Useful for "match found AND it cost N typos".
  - `niyama_fuzzy_last_error()` — error code from last compile.
- `FUZZY_FLAG_CASE_INSENSITIVE` (= 1) — ASCII case-fold flag.
- ADR 0005 — niyama_fuzzy ABI shape and scope. Records the
  algorithm choice (DP over bitap), the three-mode API, and the
  Unicode-NFD deferral.
- `tests/fuzzy.tcyr` — 45 unit tests across 7 groups: distance
  correctness against known Levenshtein values (kitten/sitting,
  Saturday/Sunday), all three match modes, threshold edges,
  case-insensitive flag, last_distance / last_error
  observability, real-world command-completion sketches.
- `fuzz/fuzzy.fcyr` — 757-assertion harness verifying all five
  Levenshtein **mathematical invariants** on randomized inputs:
  identity (`d(s,s)=0`), symmetry (`d(a,b)=d(b,a)`),
  non-negativity, length bound (`d(a,b) ≤ max(|a|,|b|)`), triangle
  inequality (`d(a,c) ≤ d(a,b)+d(b,c)`). Plus the no-crash sweep
  and compile-error invariants.
- `tests/fuzzy.bcyr` — bench harness covering compile, distance,
  match, search (substring + prefix), case-insensitive, and a
  larger-pattern stress.

### Performance floor (M3.5, x86_64, cyrius 5.8.42)

- `fuzzy_compile_*` — ~1 μs.
- `fuzzy_distance` (6-byte pattern, 5-byte text): ~1 μs.
- `fuzzy_distance` (6-byte pattern, 30-byte text): ~3 μs.
- `fuzzy_match`: ~1 μs.
- `fuzzy_search_short` (6-byte pattern, 26-byte text): ~3 μs.
- `fuzzy_search_long` (6-byte pattern, 256-byte text): ~17 μs.
- `fuzzy_search_prefix`: ~2 μs.
- `fuzzy_case_insensitive`: ~1 μs.
- `fuzzy_medium_pattern_distance` (25-byte pattern, 46-byte text):
  ~15 μs.

### Changed

- `dist/niyama.cyr` — bundle now includes `src/fuzzy.cyr` alongside
  `src/bre.cyr`, `src/re2.cyr`, and `src/pcre.cyr`.
  `NIYAMA_VERSION` → `"0.5.0"`.
- `src/main.cyr` smoke banner reflects M3.5 status.

### Deferred (not in M3.5 — see ADR 0005)

- Unicode NFD normalization (`FUZZY_FLAG_UNICODE_NFD`) — needs
  ~25KB Unicode decomposition table; ASCII-heavy AGNOS consumers
  don't benefit yet. Post-v1.0.
- Exact start-position recovery in `_search` — currently returns
  `end - len(pat)` clamped. Heuristic covers ≥90% of consumer
  cases; M5 may revisit if asked.

## [0.4.0] — 2026-05-03

M3 — third engine: pcre (`niyama_pcre_*`). Backtracking matcher (the
first non-Pike-NFA engine in niyama) bringing the features re2
deliberately rejects: backreferences, lookahead, atomic groups, named
captures, possessive quantifiers. Catastrophic-backtracking risk is
mitigated by an explicit step-limit guard with a configurable budget
(default 1M steps) and a hard recursion-depth bound (256). Per ADR
0004.

### Added

- `src/pcre.cyr` — backtracking matcher with PCRE-flavor parser,
  ~1100 lines. New opcodes: `PCRE_OP_BACKREF`, `PCRE_OP_LOOKAHEAD`,
  `PCRE_OP_NLOOKAHEAD`, `PCRE_OP_LOOKAHEAD_END`, `PCRE_OP_ATOMIC`,
  `PCRE_OP_ATOMIC_END`. Same instruction layout as bre/re2 for
  shared opcodes.
- Public ABI mirroring niyama_bre / niyama_re2 plus pcre-specific
  extensions:
  - `niyama_pcre_compile` / `_match` / `_search` / `_search_at`
  - `niyama_pcre_group_start` / `_group_end` (groups 0..9)
  - `niyama_pcre_group_by_name(nfa, name)` — named-capture lookup
  - `niyama_pcre_last_error()` — frozen error code set
  - `niyama_pcre_set_step_limit(n)` — configurable step budget
  - `niyama_pcre_last_step_count()` — observability hook
- **PCRE feature set in M3**: ERE base (literals, `.`, anchors,
  `\d`/`\w`/`\s`/`\b`, classes, `*` `+` `?` `{n,m}`, lazy,
  alternation, capturing + non-capturing), plus:
  - **Backreferences `\1`-`\9`** — the headline PCRE feature.
  - **Lookahead `(?=...)` and `(?!...)`** — variable-width.
  - **Atomic groups `(?>...)`** — no internal backtracking.
  - **Possessive quantifiers `*+`, `++`, `?+`, `{n,m}+`** —
    desugared to atomic-wrapping at compile.
  - **Named captures `(?<name>...)` and `(?P<name>...)`** — both
    syntaxes; lookup by name. Up to 9 named (shared with positional).
  - Step-limit + depth-limit catastrophic-backtracking guard.
- ADR 0004 — niyama_pcre engine ABI shape, matcher architecture,
  and scope. Records the backtracking-vs-Pike-NFA decision, the
  step-limit semantics, and the deferral list.
- `tests/pcre.tcyr` — 83 unit tests across 13 groups, including
  backref correctness, lookahead semantics, atomic-blocks-backtracking
  demo, named-capture lookup, all 7 deferred-feature rejection codes,
  and the step-limit guard kicking in on `(a+)+b` against 25 'a's.
- `fuzz/pcre.fcyr` — 229-assertion harness with adversarial pattern
  generator, every rejection invariant, and 4 catastrophic-backtracking
  patterns under tight step limit.
- `tests/pcre.bcyr` — bench harness covering compile + search +
  PCRE-specific features + the bounded-DoS bench (`(a+)+b` on 30
  'a's with `step_limit=50k` terminates in ~2.4ms).

### Performance floor (M3, x86_64, cyrius 5.8.42)

- `pcre_compile_*` — 4-5 μs avg (literal, email-class, backref,
  lookahead).
- `pcre_search_literal` (256-byte input): ~17 μs.
- `pcre_search_email`: ~9 μs.
- `pcre_search_alt` (3-way alt): ~61 μs.
- `pcre_backref` (`(\w+) \1` on `hello hello world`): ~2 μs.
- `pcre_lookahead` (`\w+(?=@)` on 256-byte input): ~180 μs (largest
  search bench — lookahead validates against multiple match start
  positions).
- `pcre_atomic` (`(?>a*)b` on `aaaaaaaaaaaaab`): ~3 μs.
- `pcre_named_captures`: ~3 μs.
- **Catastrophic-backtracking guard**: `(a+)+b` against 30 'a's with
  no terminator, `step_limit=50000` — ~2.4 ms. Without the limit
  this pattern would explore ~2^30 paths (would never terminate in
  practice).

### Changed

- `dist/niyama.cyr` — bundle now includes `src/pcre.cyr` alongside
  `src/bre.cyr` and `src/re2.cyr`. `NIYAMA_VERSION` → `"0.4.0"`.
- `src/main.cyr` smoke banner reflects M3 status.

### Deferred (not in M3 — see ADR 0004 for rationale)

- Lookbehind `(?<=...)` `(?<!...)` — needs fixed-width analysis;
  M3.5 candidate if a consumer asks. Compile rejects with
  `PCRE_E_LOOKBEHIND_UNSUPPORTED`.
- Unicode property classes `\p{L}` — needs Unicode database. Rejects
  with `PCRE_E_UNICODE_PROP_UNSUPPORTED`.
- POSIX bracket classes `[:alpha:]` — deferred to M4 (vim flavor
  inherits the same semantics).
- Recursion `(?R)` `(?P>name)` — `PCRE_E_RECURSION_UNSUPPORTED`.
- Conditional patterns `(?(...)...)` — `PCRE_E_CONDITIONAL_UNSUPPORTED`.
- Inline flags `(?i)` `(?m)` `(?s)` — generic `PCRE_E_SYNTAX`.

## [0.3.0] — 2026-05-03

M2 — second engine: re2 (`niyama_re2_*`). Linear-time Pike NFA matcher
with ERE-flavor parser and **explicit compile-time rejection of every
feature that would break the linear-time guarantee**. Each
non-regular construct gets its own error code so consumers know
exactly which engine to fall back to (niyama_pcre at M3). Per ADR
0003.

### Added

- `src/re2.cyr` — Pike NFA matcher with ERE syntax: `\d` `\D` `\w`
  `\W` `\s` `\S` `\b` `\B`, alternation `|`, `(...)` capturing
  groups, `(?:...)` non-capturing, greedy AND lazy quantifiers
  (`*` `+` `?` `{n,m}` and lazy variants). Same Pike NFA matcher
  kernel as niyama_bre — the linear-time guarantee is structural.
- Public ABI: `niyama_re2_compile`, `niyama_re2_match`,
  `niyama_re2_search`, `niyama_re2_search_at`,
  `niyama_re2_group_start`, `niyama_re2_group_end`,
  `niyama_re2_last_error`. Mirrors `niyama_bre_*` shape per ADR
  0002.
- **Compile-time rejection** of features that would break linear
  time, each with its own error code:
  - `RE2_E_BACKREF_UNSUPPORTED` (= 2) — `\1`-`\9` backreferences.
  - `RE2_E_LOOKAROUND_UNSUPPORTED` (= 3) — `(?=...)` `(?!...)`
    `(?<=...)` `(?<!...)` lookaround.
  - `RE2_E_ATOMIC_UNSUPPORTED` (= 4) — `(?>...)` atomic groups.
  - `RE2_E_RECURSION_UNSUPPORTED` (= 5) — `(?R)` `(?P>name)`
    recursion / subroutine calls.
  - `RE2_E_TOO_LARGE` (= 6) — pattern compile exceeds instruction
    or class limits.
- ADR 0003 — niyama_re2 engine ABI shape and scope. Records the
  ERE feature set, the linear-time guarantee, the rejection
  contract, and the deferred-to-post-M3 named-capture decision.
- `tests/re2.tcyr` — 76 unit tests across 11 feature groups,
  including each rejection error code AND adversarial linear-time
  patterns (`(a|a)*b`, `(a*)*b`, Cox's `a?{30}a{30}` adversary,
  deep alternation × repetition).
- `fuzz/re2.fcyr` — 221-assertion harness, including the four
  rejection-invariant checks (every pattern matching `\1`/`(?=`/
  `(?>`/`(?R` MUST yield the corresponding error code).
- `tests/re2.bcyr` — bench harness with **dedicated DoS-resistance
  benches**: `(a|a)*b` and `(a*)*b` against 200-`a` inputs that
  would DoS a backtracking engine.

### Performance floor (M2, x86_64, cyrius 5.8.42)

- `re2_compile_*` — 3-5 μs avg (literal, alt, email-class).
- `re2_search_literal` (256-byte input): ~44 μs.
- `re2_search_alt` (3-way alt over 256-byte input): ~73 μs.
- `re2_search_email`: ~15 μs.
- **DoS-resistance** (the headline numbers — these would never
  terminate on a backtracking engine):
  - `(a|a)*b` against 200 `a`s, no match: ~84 μs.
  - `(a*)*b` against 200 `a`s, no match: ~60 μs.
  - Cox's `a?{30}a{30}` adversary, match: ~201 μs.

### Changed

- `dist/niyama.cyr` — bundle now includes `src/re2.cyr` alongside
  `src/bre.cyr`. `NIYAMA_VERSION` → `"0.3.0"`.
- `src/main.cyr` smoke banner reflects M2 status.

### Deferred (not in M2)

- Named captures `(?P<name>...)` / `(?<name>...)` — deferred to
  post-M3. Both re2 and pcre will share the named-capture API
  surface; designing it once with both engines in hand avoids
  shipping it twice.
- Unicode property classes `\p{L}` — M2 is ASCII-only.
- Inline flags `(?i)` `(?m)` `(?s)` — post-v1.0 unless asked.

## [0.2.0] — 2026-05-03

M1 — first engine shipped: POSIX BRE (`niyama_bre_*`). Per ADR 0002,
ABI mirrors stdlib `regex_*` shape; `\1`-`\9` backreferences are
rejected at compile time with `BRE_E_BACKREF_UNSUPPORTED` (deferred
to potentially post-v1.0 — explicit reject + document, never silent
skip).

### Added

- `src/bre.cyr` — POSIX BRE engine. Forks the cyrius stdlib
  `lib/regex.cyr` Pike NFA / Thompson construction (instruction
  model, class bitmap, matcher) with a BRE-flavor parser. All
  globals `_bre_*`-prefixed for collision-free coexistence with
  `lib/regex.cyr` in the same program.
- Public ABI: `niyama_bre_compile`, `niyama_bre_match`,
  `niyama_bre_search`, `niyama_bre_search_at`,
  `niyama_bre_group_start`, `niyama_bre_group_end`,
  `niyama_bre_last_error`. Error codes frozen at M1: `BRE_E_OK` (0),
  `BRE_E_SYNTAX` (1), `BRE_E_BACKREF_UNSUPPORTED` (2),
  `BRE_E_TOO_LARGE` (3), `BRE_E_BAD_ANCHOR` (4 — reserved).
- POSIX BRE features: literals, `.`, `*`, `\(...\)` capturing groups
  (1..9), `\{n,m\}` / `\{n,\}` / `\{n\}` quantifiers, `^` start
  anchor (only at byte 0), `$` end anchor (only at end), `[...]` /
  `[^...]` bracket expressions with ranges, common backslash
  escapes (`\.` `\*` `\\` `\n` `\t` etc.).
- POSIX-faithful literal-by-default for `+`, `?`, `(`, `)`, `{`,
  `}`, `|`. `^`/`$` are literal in mid-pattern positions.
- ADR 0002 — niyama_bre engine ABI shape and scope. Records ABI
  surface, error-code numbering (frozen ABI at M1), backref
  rejection policy + post-v1.0 reconsideration path.
- `tests/bre.tcyr` — 68 unit tests covering literals, dot, star,
  POSIX literal-by-default for `+`/`?`, anchor placement, bracket
  expressions, groups + brace quantifiers, escapes, backref
  rejection, syntax error paths, search semantics, and a
  catastrophic-backtracking-class adversarial pattern (asserts
  linear-time DoS-resistance).
- `fuzz/bre.fcyr` — randomized fuzz harness, 200-iter sweep over
  metachar-heavy random patterns; smoke corpus drawn from past
  stdlib regex bug-fix history (3-way alt regression at v5.7.18,
  body-pc re-entry regression).
- `tests/bre.bcyr` — bench harness across compile + search paths.

### Performance floor (M1, x86_64, cyrius 5.8.42)

- `bre_compile_*` — ~3 μs avg (literal, dot-star, quantifier, group).
- `bre_search_literal_hit` (256-byte input, 6-byte needle): ~44 μs.
- `bre_search_literal_miss` (22-byte input): ~9 μs.
- `bre_search_dot_star` (256-byte input, `.*needle.*`): ~112 μs.
- `bre_search_anchored` (`^needle`): ~2 μs (anchor short-circuits).
- `bre_search_group` (256-byte input, two captures): ~16 μs.

Floor recorded for M5 hardening regression detection.

### Changed

- Cyrius pin bumped: `5.7.24` → `5.8.42` (`cyrius.cyml [package].cyrius`).
- `dist/niyama.cyr` — placeholder replaced with the M1 bundle
  (single-include over `src/bre.cyr`). `NIYAMA_VERSION` now `"0.2.0"`.

### Deferred (not in M1)

- `\1`-`\9` backreferences — rejected at compile per ADR 0002.
  Patterns needing backrefs should use `niyama_pcre` (M3); BRE
  backref support is potentially post-v1.0 work, gated on a real
  consumer ask.
- GNU `\<` / `\>` word boundaries — deferred to M4 (vim flavor
  inherits the same semantics).
- `[:alpha:]` POSIX bracket character classes — deferred to M4.

## [0.1.0] — 2026-04-28

Initial scaffold. Repo positioning, doc-tree, ADR 0001 (sandhi-pattern
positioning + fold lifecycle), roadmap to v1.0. No engines shipped
yet — engine work begins at M1 (POSIX BRE).

### Added

- Project scaffold via `cyrius init niyama` (first-party-standards
  conformance: VERSION, cyrius.cyml, CLAUDE.md, CHANGELOG.md,
  README.md, LICENSE, CI workflows, doc-tree).
- README.md positioning: additional-regex-engines repo for the
  AGNOS-lineage Cyrius ecosystem, sandhi-pattern lifecycle, M0–v1.0
  engine roadmap, planned consumers (cyim, owl, agnoshi, daimon).
- ADR 0001 — niyama as the additional-engines repo following the
  sandhi-pattern fold lifecycle. Records:
  - Why one foundational engine in cyrius stdlib + additional engines
    in niyama (rather than per-engine repos or stdlib expansion).
  - The fold-back gate (≥2 long-horizon consumers + 1.0.0 + frozen
    surface + explicit fold ADR per sandhi ADR 0002 template).
  - Speculative cyrius 5.8.0 fold target (CONDITIONAL on consumer
    count — not a deadline).
- `docs/development/roadmap.md` — M0 (scaffold, done), M1 (bre — first
  engine, picked to shake out dispatch surface), M2 (re2, DoS-safe),
  M3 (pcre, largest fuzz target), M3.5 (fuzzy, Levenshtein), M4 (vim
  flavor), M5 (hardening + freeze), v1.0 (fold-ready).
- `docs/development/state.md` — initial scaffold snapshot.
- `dist/niyama.cyr` — placeholder for the fold-ready single-file
  artifact (sandhi precedent: `dist/sandhi.cyr` is what stdlib
  vendored byte-identical at the fold).
