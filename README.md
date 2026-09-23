# niyama

> **Additional regex engines for the AGNOS-lineage Cyrius ecosystem** —
> POSIX BRE, linear-time RE2, Perl-compatible PCRE, Levenshtein fuzzy
> and vim-flavor matchers, alongside Cyrius stdlib's foundational Pike
> NFA engine. Part of cyrius stdlib as `lib/niyama.cyr` since cyrius
> v5.9.0.

**niyama** (Sanskrit नियम — *rule, regulation, restraint*) is the
additional-rules-engine layer for the AGNOS Cyrius ecosystem. Regex
*is* a rule — and niyama is the home for the rule engines that don't
fit the universal-floor case Cyrius stdlib already covers.

## Status

**v1.x — fold-maintenance.** All five engines have shipped, and the
public surface is frozen (ADR 0010). niyama has been folded into cyrius
stdlib (ADR 0011), so this repo carries bug fixes, hardening, and
toolchain bumps. For the live version, toolchain pin, and test counts,
see [`docs/development/state.md`](docs/development/state.md). For open
work, see [`docs/development/roadmap.md`](docs/development/roadmap.md).

## Engines

| Engine | API | Matcher | Use it for |
|---|---|---|---|
| **bre** | `niyama_bre_*` | Pike NFA, linear time | `grep -G` / `sed` POSIX BRE, plus GNU `\<` `\>` and POSIX bracket classes |
| **re2** | `niyama_re2_*` | Pike NFA, linear time | ERE on untrusted input — named captures, inline flags, `\p{…}` |
| **pcre** | `niyama_pcre_*` | Backtracking, step-limited | Backrefs, lookahead and fixed-width lookbehind, atomic groups, recursion, conditionals |
| **fuzzy** | `niyama_fuzzy_*` | Levenshtein DP | Typo-tolerant matching — anchored, substring, or prefix |
| **vim** | `niyama_vim_*` | Pike NFA, linear time | vim/cyim patterns — four magicness modes, `\zs` `\ze`, `\<` `\>` |

Each engine rejects the features it leaves out at compile time, each
with its own error code; none is silently degraded:
- **re2** refuses backrefs, lookaround, atomic groups, and recursion.
  That is what keeps its linear-time guarantee.
- **bre** and **vim** refuse backrefs (ADR 0009). For backrefs, use
  **pcre**, and bound it with `niyama_pcre_set_step_limit()` when the
  pattern is untrusted.

**Known limitation:** pcre's matcher recursion gets deeper with each
input position it consumes. A quantifier that has to consume more than
about 250 positions therefore reports no match and sets
`PCRE_E_DEPTH_EXCEEDED`. The fix is planned
for v1.1.0 (see the roadmap).

Every public symbol, error code, and limit is listed in
[`docs/api/README.md`](docs/api/README.md). For which engine suits which
job, see
[state.md § Engine-selection guidance](docs/development/state.md#engine-selection-guidance-for-consumers).

## Using niyama

niyama ships with cyrius stdlib. To use it:
1. Declare the stdlib leaves it needs, which are listed in
   [`dist/niyama.deps`](dist/niyama.deps).
2. Include `lib/niyama.cyr` explicitly. Folded libraries are not
   prepended automatically.

```toml
# cyrius.cyml
[deps]
stdlib = ["string", "fmt", "alloc", "io", "vec", "str", "syscalls", "assert", "unicode"]
```

```cyrius
include "lib/niyama.cyr"

fn main(): i64 {
    var re = niyama_re2_compile("(?<year>\\d{4})-(?<month>\\d{2})");
    if (re == 0) { return niyama_re2_last_error(); }

    var s = "released 2026-09-23";
    if (niyama_re2_search(re, s) < 0) { return 1; }

    var g = niyama_re2_group_by_name(re, "year");
    var from = niyama_re2_group_start(re, g);
    var to = niyama_re2_group_end(re, g);
    syscall(1, 1, s + from, to - from);   # writes "2026"
    return 0;
}

var r = main();
syscall(60, r);
```

All five engines work the same way:
- `_compile` returns a handle, or `0` with the reason in `_last_error()`.
- `_search` returns a **byte offset**, or `-1` if there is no match.
- The four regex engines read captures with `_group_start` and
  `_group_end`; group 0 is the whole match.

Your cyrius toolchain ships a particular niyama release. To pin a
different one, vendor [`dist/niyama.cyr`](dist/niyama.cyr) from that
tag (for example as `vendor/niyama.cyr`) and include it in place of
`lib/niyama.cyr`.

## Why niyama exists

Cyrius stdlib's `lib/regex.cyr` is **one foundational engine**: a Pike
NFA with POSIX-ERE-ish syntax, behind the `regex_*` ABI. It covers the
90% case. The other flavors each have real but narrower demand, so they
were built here, out of tree, until they earned a place in stdlib.

That is the **sandhi pattern**. [sandhi](https://github.com/MacCracken/sandhi)
shipped standalone, reached 1.0.0, and was vendored byte-identical into
cyrius stdlib at v5.7.0. niyama followed the same path:
1. Its surface froze at 0.9.0 (ADR 0010), and 1.0.0 shipped fold-ready.
2. When the AGNOS kernel joined cyim as a second long-horizon consumer,
   cyrius v5.9.0 vendored niyama 1.0.1's `dist/niyama.cyr` as
   `lib/niyama.cyr` (ADR 0011, 2026-05-06).

Since then, fixes land here and reach stdlib on cyrius's next refold.
Additive extensions land in stdlib's copy instead (ADR 0010).

## Consumers

- **[cyim](https://github.com/MacCracken/cyim)** — modal text editor,
  and niyama's first consumer. `--regex=<flavor>` selects any of the
  five engines (cyim ADR 0002).
- **AGNOS kernel** — the second long-horizon consumer, whose pin for the
  bare-metal target triggered the fold (ADR 0011).
- **Planned:**
  - [owl](https://github.com/MacCracken/owl) — filtering and search in
    the pager.
  - [agnoshi](https://github.com/MacCracken/agnoshi) — `fuzzy` for shell
    completion.
  - [daimon](https://github.com/MacCracken/daimon) agents — `re2` pattern
    gates on untrusted input.

Live status is in [state.md § Consumers](docs/development/state.md#consumers).

## Roadmap

[`docs/development/roadmap.md`](docs/development/roadmap.md) looks
forward only. It holds:
- the v1.1.0 pcre backtrack-stack fix;
- open v1.0.x maintenance items;
- post-fold extension candidates, which belong to cyrius stdlib's
  `lib/niyama.cyr` rather than this repo.

The M0 → v1.0 milestone history is in the 0.1.0 → 1.0.0 entries of
[`CHANGELOG.md`](CHANGELOG.md).

## Building niyama

The toolchain version is pinned by `cyrius = "…"` in `cyrius.cyml`.

```sh
cyrius deps                                          # vendor stdlib into lib/, checked against cyrius.lock
CYRIUS_DCE=1 cyrius build src/main.cyr build/niyama  # smoke binary (the engines are a library)
cyrius test                                          # tests/*.tcyr
cyrius fuzz                                          # fuzz/*.fcyr
cyrius bench tests/re2.bcyr                          # benchmarks, one harness per engine
cyrius distlib                                       # regenerate dist/niyama.cyr (never edit it by hand)
```

`dist/niyama.cyr` is the single-file bundle that cyrius stdlib vendors,
and `dist/niyama.deps` lists the stdlib leaves it needs. Both are
checked in.

## Documentation

- [`docs/api/`](docs/api/README.md) — the frozen public surface.
- [`docs/adr/`](docs/adr/) — decisions: per-engine scope, surface freeze, fold.
- [`docs/architecture/`](docs/architecture/) — invariants that aren't obvious from the code.
- [`docs/audit/`](docs/audit/) — security audits.
- [`docs/benchmarks.md`](docs/benchmarks.md) — benchmark history.
- [`docs/development/state.md`](docs/development/state.md) — live state.
- [`docs/development/roadmap.md`](docs/development/roadmap.md) — open work.
- [`CHANGELOG.md`](CHANGELOG.md) — per-release changes, complete from 0.1.0.

## License

GPL-3.0-only.

## See also

- [cyim](https://github.com/MacCracken/cyim) — first consumer.
- [sandhi](https://github.com/MacCracken/sandhi) — the fold-pattern
  precedent (out-of-tree → stdlib at cyrius v5.7.0).
- [vyakarana](https://github.com/MacCracken/vyakarana) — sister
  Sanskrit-grammar lib (token-level grammar, used by cyim for syntax
  highlighting).
- [first-party-standards](https://github.com/MacCracken/agnosticos/blob/main/docs/development/applications/first-party-standards.md)
  — AGNOS-lineage project conventions niyama conforms to.
