# 0006 — niyama_vim engine ABI shape and scope

**Status**: Accepted
**Date**: 2026-05-03

> **Update 2026-05-05** — the M4 "potentially post-v1.0" deferral
> on `\1`-`\9` backref is **resolved by ADR 0009**: vim stays
> backref-rejecting for v1.0; post-fold revisit is explicitly open
> via cyrius stdlib `lib/niyama.cyr` (sandhi precedent), gated on
> the fold gate being met. The asymmetry vs. bre (which is
> permanently closed) is driven by cyim being a load-bearing
> long-horizon vim-flavor consumer. ADR 0009 carries the
> containment design that a post-fold implementation must satisfy.
> `VIM_E_BACKREF_UNSUPPORTED = 2` keeps its current name through
> v1.0; post-fold extension would shift it to reserved-but-unused.

## Context

M4 ships niyama's fifth engine: vim — vim/cyim flavor. cyim's
`:s/old/new/` and ex-mode pattern history are the consumer; users
coming from vim expect `\<word\>`, `\zs` / `\ze`, the four
magicness modes, and POSIX bracket classes inside character
classes (`[[:alpha:]]`). The roadmap calls for all four magicness
modes and the listed metachars.

Three load-bearing decisions had to land before code:

### 1. Matcher architecture — Pike NFA, not backtracking

vim's regex grammar is regular if you exclude backreferences. The
features unique to vim — magicness modes, `\<` / `\>` word
boundaries, `\zs` / `\ze` match-position markers, POSIX bracket
classes — all compose cleanly into the Pike NFA instruction model
shared by bre and re2. So niyama_vim forks the Pike NFA kernel
(same as re2's relationship to bre) with a vim-flavor parser.

**Backreferences `\1`-`\9`** would force a backtracking matcher
(same reason niyama_pcre is backtracking). Real vim has TWO
internal engines — an NFA matcher for backref-free patterns and a
backtracking fallback for patterns with backrefs. niyama_vim
picks ONE: the Pike NFA, with explicit compile-time rejection of
`\1`-`\9` (consistent with the bre/re2 posture per ADRs 0002 and
0003). Consumers needing vim-flavor regex *with* backrefs use
niyama_pcre — pcre features cover the backref case, and most
real-world vim patterns don't use backref anyway.

The backref-reject decision is **flagged for v0.9.0 (M4.5
catch-up) review**. If a real cyim consumer materializes who
needs backref in vim flavor, v0.9.0 is the natural revisit point.
Until then, niyama_vim is reject-and-document, same posture as
bre/re2.

### 2. Magicness — all four modes, opts-flag-controlled

vim's four magicness modes (`\v` very-magic, `\m` magic, `\M`
nomagic, `\V` very-nomagic) change which characters are special
in the pattern. A pattern compiled in one mode is genuinely
different from the same pattern in another — `(foo)` is a group
in very-magic but three literals in magic.

niyama_vim ships **all four modes** as an `opts` flag —
`niyama_vim_compile_opts(pat, mode)` — with `VIM_MODE_MAGIC` as
the default to match vim's out-of-box behavior. Mode-switching
mid-pattern via `\v` `\m` `\M` `\V` is **not shipped in M4** —
the opts flag is the entry point. Mid-pattern switching is a
separate-ADR feature for v0.9.0 if asked.

### 3. `\zs` and `\ze` — group 0 manipulation, not a new opcode

`\zs` resets the match start to the current position. `\ze`
freezes the match end at the current position. Both are
implementable as **SAVE 0 / SAVE 1 emissions** — the existing
opcode set already covers it.

The wrinkle: vim's compiler emits an implicit `SAVE 0` at the
very start of the pattern and an implicit `SAVE 1` at the very
end. With `\zs`, the explicit SAVE 0 just overwrites the implicit
one (later SAVE wins); semantics correct. With `\ze`, the explicit
SAVE 1 happens *before* the implicit one — and the implicit
overwrites it.

Fix: track a `_vim_saw_ze` flag in the parser. If set, **skip the
implicit final SAVE 1**. The user's `\ze` position becomes the
end. This is the only tricky bit of `\zs` / `\ze` implementation;
otherwise they're trivial SAVE emissions.

## Decision

**niyama_vim ships a Pike NFA matcher with vim-flavor parser, all
four magicness modes, `\<`/`\>` word boundaries, `\zs`/`\ze`
match-position markers, POSIX bracket classes, and explicit
compile-time rejection of `\1`-`\9` backreferences (M4 default;
flagged for v0.9.0 revisit).**

### ABI surface (mirrors niyama_re2 per ADR 0002)

- `niyama_vim_compile(pat)` — compiles in `VIM_MODE_MAGIC` (vim
  default).
- `niyama_vim_compile_opts(pat, mode)` — full opts form.
  `mode` is one of `VIM_MODE_VERY_MAGIC`, `VIM_MODE_MAGIC`,
  `VIM_MODE_NOMAGIC`, `VIM_MODE_VERY_NOMAGIC`.
- `niyama_vim_match(nfa, s)`, `niyama_vim_search(nfa, s)`,
  `niyama_vim_search_at(nfa, s, len, from)`,
  `niyama_vim_group_start(nfa, n)`, `niyama_vim_group_end(nfa, n)`,
  `niyama_vim_last_error()` — all mirror the established
  per-engine ABI shape.

### Mode constants

| Constant | Value | vim equivalent | Notes |
|---|---|---|---|
| `VIM_MODE_VERY_MAGIC` | 0 | `\v` | All metachars special; closest to PCRE/ERE feel. |
| `VIM_MODE_MAGIC` | 1 | `\m` (default) | Most special, but `( ) { + ? = |` need backslash. |
| `VIM_MODE_NOMAGIC` | 2 | `\M` | Only `^ $ . *` special; `[` literal. |
| `VIM_MODE_VERY_NOMAGIC` | 3 | `\V` | Only `\` and `^`/`$` (at edges) special. |

### vim feature set shipped in M4

| Feature | Notes |
|---|---|
| Literals + escapes | `\n` `\r` `\t` `\\` `\.` `\*` etc. |
| `.` any byte (modes v, m) | Literal in M, V. |
| `*` zero-or-more (v, m) | Literal in M, V; `\*` for quantifier. |
| `\+` one-or-more, `\?` optional, `\=` optional | All modes via `\<x>` form; `+`/`?`/`=` bare in very-magic only. |
| `\{n,m\}` brace quantifier (greedy + `\{-n,m}` lazy) | Lazy form is the vim convention. |
| `\(...\)` capturing groups (1..9) (m, M, V); `(...)` (v) | Mode-dependent paren syntax. |
| `\|` alternation (m, M, V); `|` (v) | Same mode-dependence. |
| `^` BOL, `$` EOL | Special at pattern boundaries (m, M, V); literal anywhere else without `\^` `\$`; very-nomagic needs `\^` `\$` even at edges. |
| `[...]` / `[^...]` bracket expressions (v, m) | Need `\[` in M, V. |
| `[[:alpha:]]` / `[[:digit:]]` / `[[:space:]]` / etc. | POSIX bracket classes inside char classes. **Implementation shared with v0.9.0 catch-up for bre/pcre per the M4.5 plan.** |
| `\<` start-of-word, `\>` end-of-word | Same semantics as GNU `\<`/`\>` (per ADR 0002). |
| `\zs` start-of-match marker | SAVE 0 at current position. |
| `\ze` end-of-match marker | SAVE 1 at current position; implicit final SAVE 1 suppressed. |
| `\d` `\D` `\w` `\W` `\s` `\S` predefined classes | Same ASCII-only semantics as bre/re2/pcre. |
| `\1`-`\9` backreferences | **Rejected** at compile (`VIM_E_BACKREF_UNSUPPORTED`). Default per ADR 0002/0003 posture; v0.9.0 revisit. |

### Out of M4 scope (deferred)

| Feature | Why deferred | Where it lands |
|---|---|---|
| Mid-pattern mode switching `\v` `\m` `\M` `\V` | M4 ships opts-flag mode entry only; mid-pattern is rarely-used | v0.9.0 if asked |
| `~` (last substitute string) | Replacement-language feature; consumer concern | n/a — replacement is consumer concern |
| `&` in replacement = full match | Replacement-language feature | n/a — replacement is consumer concern |
| `\u`, `\l`, `\U`, `\L` (case conversion in replacement) | Replacement-language feature | n/a |
| Vim's vast list of additional `\X` escapes (`\a`, `\A`, `\l`, `\L`, `\u`, `\U`, `\x`, `\X`, `\o`, `\O`, `\h`, `\H`, etc. for character classes) | M4 ships standard `\d`/`\w`/`\s`; vim-specific extras are rarely-used in cyim patterns | post-v1.0 if asked |

### Error codes (frozen ABI from M4 onward)

| Code | Constant | Meaning |
|---|---|---|
| 0 | `VIM_E_OK` | last compile succeeded |
| 1 | `VIM_E_SYNTAX` | generic syntax error |
| 2 | `VIM_E_BACKREF_UNSUPPORTED` | `\1`-`\9` — see § Future scope |
| 3 | `VIM_E_INVALID_MODE` | mode flag out of range |
| 4 | `VIM_E_TOO_LARGE` | pattern compiled too large |

Numeric values frozen. `VIM_E_BACKREF_UNSUPPORTED` shares numeric
value 2 with `BRE_E_BACKREF_UNSUPPORTED` and `RE2_E_BACKREF_UNSUPPORTED` —
deliberate cross-engine consistency for the one error code that
crosses engine boundaries.

## Consequences

### Positive

- **cyim's `:s/old/new/` integration unblocked** for the regex
  side (replacement is cyim's own concern). Muscle-memory
  continuity for users coming from vim.
- **Same Pike NFA kernel as bre/re2** — M5 hardening / refactor
  has three engines with the same matcher to consolidate.
- **POSIX bracket classes finally land** in niyama. ADR 0002 and
  ADR 0004 had punted to "M4 reuse"; vim is that reuse point.
  v0.9.0 catch-up backports the implementation to bre/pcre.

### Negative

- **No backref in vim flavor** — real vim users who use `\1` in
  `:s/...` patterns get a compile error. Mitigated by the v0.9.0
  revisit hook + the existing pcre alternative.
- **No mid-pattern mode switching.** Most consumers probably
  don't use it (the opts-flag form covers the use cases), but
  some pre-existing vim patterns embed `\v` mid-pattern. v0.9.0
  candidate.

### Neutral

- **Replacement language out of scope.** niyama is a matching
  library. cyim handles `:s/old/new/` replacement itself; niyama
  provides match positions + captures, and cyim does the rest.
  No replacement helpers in this engine.

## Future scope

Same revisit hook as ADR 0002's BRE backref note: **v0.9.0 (M4.5
catch-up) is the natural point to reconsider** the backref-reject
decision. If a real cyim consumer needs backref in vim patterns,
the path is:

1. Decide at v0.9.0 planning whether vim-backref ships in v0.9.0
   or stays post-v1.0.
2. If shipping: switch niyama_vim from Pike NFA to backtracking
   (same architecture as niyama_pcre) — substantial rewrite,
   roughly doubles the engine size.
3. Step-limit guard becomes load-bearing (same as pcre).

Until that decision: niyama_vim is Pike NFA, reject + document.

## Alternatives considered

- **Backtracking matcher from M4.** Rejected. Doubles the engine
  size, brings catastrophic-backtracking risk into yet another
  engine, and the Pike NFA covers ≥95% of real-world vim
  patterns. v0.9.0 is the right place to revisit.
- **Ship only magic mode in M4.** Rejected — the roadmap
  acceptance explicitly lists all four, and the per-mode dispatch
  table is finite. Mode switching mid-pattern is the harder
  feature and that one IS deferred.
- **Translate vim-flavor patterns to ERE and dispatch to
  niyama_re2.** Rejected — vim's `\zs`/`\ze` and magicness modes
  don't translate cleanly; per-engine flavor is the cleaner
  architecture per ADR 0001.
- **Implement vim's full `\X` escape menagerie in M4.** Rejected
  — most are rarely-used in real cyim patterns. Standard
  `\d`/`\w`/`\s` covers the load-bearing cases. Add others if
  asked.

## References

- [niyama ADR 0001](0001-additional-engines-repo-sandhi-pattern.md) — positioning, fold lifecycle.
- [niyama ADR 0002 / 0003 / 0004 / 0005](0002-bre-engine-abi-and-scope.md) — prior engine ABIs that niyama_vim mirrors.
- [vim docs: `:help magic`](https://vimhelp.org/pattern.txt.html#%2Fmagic) — magicness modes spec.
- [vim docs: `:help \zs`](https://vimhelp.org/pattern.txt.html#%2F%5Czs) — match-position markers.
- [`docs/development/roadmap.md`](../development/roadmap.md) — M4 acceptance criteria.
