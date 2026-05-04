# niyama

> **Additional regex engines for the AGNOS-lineage Cyrius ecosystem.**
> Out-of-tree home for `bre`, `re2`, `pcre`, `fuzzy`, and `vim`-style
> matchers — composable on top of Cyrius stdlib's foundational
> Pike NFA engine. Foldable into stdlib once consumer count earns it.

**niyama** (Sanskrit नियम — *rule, regulation, restraint*) is the
additional-rules-engine layer for the AGNOS Cyrius ecosystem. Regex
*is* a rule — and niyama is the home for the rule engines that don't
fit the universal-floor case Cyrius stdlib already covers.

## Status

**0.2.0** (M1, 2026-05-03) — POSIX BRE engine (`niyama_bre_*`)
shipped. ADR 0002 records the per-engine ABI shape that M2+ engines
mirror; backref support is rejected at compile time and deferred to
potentially post-v1.0. Next: M2 (`re2`, linear-time-safe). See
[`docs/development/state.md`](docs/development/state.md) for the
live snapshot and [`docs/development/roadmap.md`](docs/development/roadmap.md)
for the full M2+ plan.

## Why niyama exists

Cyrius stdlib `lib/regex.cyr` ships **one foundational regex engine**
(Pike NFA, POSIX-ERE-ish), exposed via the `regex_*` ABI as of
v5.7.23. That covers the 90% case every AGNOS-lineage consumer needs.

Additional flavors — Perl-compat features, vim-style modes, RE2's
linear-time guarantee, BRE-strict POSIX compatibility, fuzzy
(typo-tolerant) matching — each have real but narrower demand. None
have proven ≥2-long-horizon-consumer adoption yet, so they don't
belong in stdlib *today*. They DO belong in the AGNOS-lineage
ecosystem, with discoverability that an external GitHub repo provides.

niyama is that repo. Following the **sandhi pattern**:

> [sandhi](https://github.com/MacCracken/sandhi) (Sanskrit सन्धि — *junction, joining*) is
> the AGNOS service-boundary layer. Shipped standalone, hit 1.0.0,
> got vendored byte-identical into Cyrius stdlib at v5.7.0 as
> `lib/sandhi.cyr` per its own ADR 0002. Upstream repo entered
> maintenance mode after the fold; subsequent patches land via the
> Cyrius release cycle.

niyama follows the same lifecycle — out-of-tree → 1.0.0 → fold-ready
`dist/niyama.cyr` → cyrius stdlib vendors it as `lib/niyama.cyr` →
maintenance mode. The fold trigger requires ≥2 long-horizon
AGNOS-lineage consumers and an explicit fold ADR (sandhi ADR 0002 is
the template).

## Consumers (planned)

- **[cyim](https://github.com/MacCracken/cyim)** — modal text editor.
  v1.2.0 already ships `--regex=<flavor>` on six agent-drive verbs;
  the parser-side picks up niyama-provided flavors as additional
  `elif` arms in `_regex_flavor_id` once they land. cyim consumer
  code change: zero. (See cyim ADR 0002.)
- **[owl](https://github.com/MacCracken/owl)** — pager / cat-class
  utility. Filtering and search are the obvious fit.
- **[agnoshi](https://github.com/MacCracken/agnoshi)** — AI shell.
  History search, command matching, completion filters; `fuzzy`
  flavor is the immediate win.
- **[daimon](https://github.com/MacCracken/daimon)**-orchestrated
  agents — pattern gates on agent input/output; `re2` flavor for
  DoS-safe matching against untrusted patterns.

## Roadmap

See [`docs/development/roadmap.md`](docs/development/roadmap.md) for
the full M0 → v1.0 plan. Engine ordering (architecture-shakeout,
then demand):

| Milestone | Engine | Why this order |
|-----------|--------|----------------|
| **M0** | scaffold | Done — first-party-standards conformance, ABI sketch, dist artifact stub. |
| **M1** | **bre** (POSIX BRE) | ✅ Done (v0.2.0). 68 unit tests, fuzz harness, bench floor recorded. ABI shape (per ADR 0002) is the template M2+ engines mirror. |
| **M2** | **re2** (Thompson NFA, linear-time) | DoS-safe regex for untrusted patterns. Architecturally similar to stdlib's Pike NFA but with safety guarantees made explicit. |
| **M3** | **pcre** (Perl-compatible) | Power-user features: lookaround, atomic groups, named captures, backrefs, Unicode properties. Largest fuzz target — comes after re2 for diff-test fallback. |
| **M3.5** | **fuzzy** (Levenshtein) | Typo-tolerant matching. Not strict regex but lives in the neighborhood. agnoshi shell completion + daimon agent fuzzy-name-match are immediate consumers. |
| **M4** | **vim** (vim/cyim flavor) | Magic / nomagic modes, `\<`/`\>`, `\zs`/`\ze`. cyim's `:s/old/new/` and ex-mode pattern history are the consumer. |
| **M5** | P(-1) hardening + closeout | Audit, fuzz floor, perf claims, surface freeze. Fold ADR. |
| **v1.0** | fold-ready release | `dist/niyama.cyr` is byte-identical fold candidate; cyrius stdlib vendors as `lib/niyama.cyr`. |

## Build

```sh
cyrius deps                              # resolve stdlib deps
cyrius build src/main.cyr build/niyama   # compile (CLI tooling — engines are library)
cyrius test                              # run tests/*.tcyr
```

The fold-ready artifact is `dist/niyama.cyr` — single-file include
(per the sandhi precedent) that downstream consumers vendor or that
cyrius stdlib eventually folds in.

## License

GPL-3.0-only.

## See also

- [cyim](https://github.com/MacCracken/cyim) — first consumer, already
  threaded for niyama via `--regex=<flavor>` (ADR 0002).
- [sandhi](https://github.com/MacCracken/sandhi) — fold-pattern
  precedent (out-of-tree → stdlib at cyrius v5.7.0).
- [vyakarana](https://github.com/MacCracken/vyakarana) — sister
  Sanskrit-grammar lib (token-level grammar, used by cyim for syntax
  highlighting).
- [first-party-standards](https://github.com/MacCracken/agnosticos/blob/main/docs/development/applications/first-party-standards.md)
  — AGNOS-lineage project conventions niyama conforms to.
