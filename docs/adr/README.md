# Architecture Decision Records

Decisions about niyama — what we chose, the context, and the consequences we accept. Use these when a future reader would reasonably ask *"why did we do it this way?"*

## Conventions

- **Filename**: `NNNN-kebab-case-title.md`, zero-padded to four digits. Never renumber.
- **One decision per ADR.** If a decision supersedes a prior one, add a new ADR and set the old one's status to `Superseded by NNNN`.
- **Status lifecycle**: `Proposed` → `Accepted` → (optionally) `Superseded` or `Deprecated`.
- Use [`template.md`](template.md) as the starting point.

## ADR vs. architecture note vs. guide

| Kind | Lives in | Answers |
|---|---|---|
| ADR | `docs/adr/` | *Why did we choose X over Y?* |
| Architecture note | `docs/architecture/` | *What non-obvious constraint is true about the code?* |
| Guide | `docs/guides/` | *How do I do X?* |

## Index

- [0001 — niyama is the additional-engines repo, following the sandhi-pattern fold lifecycle](0001-additional-engines-repo-sandhi-pattern.md)
- [0002 — niyama_bre engine ABI shape and scope](0002-bre-engine-abi-and-scope.md)
- [0003 — niyama_re2 engine ABI shape and scope](0003-re2-engine-abi-and-scope.md)
- [0004 — niyama_pcre engine ABI shape, matcher architecture, and scope](0004-pcre-engine-abi-and-scope.md)
- [0005 — niyama_fuzzy engine ABI shape and scope](0005-fuzzy-engine-abi-and-scope.md)
- [0006 — niyama_vim engine ABI shape and scope](0006-vim-engine-abi-and-scope.md)
- [0007 — v0.7.0 catch-up: no-Unicode-dep slice of M4.5](0007-v070-catchup-no-unicode-dep.md)
- [0008 — Unicode-stdlib pivot + M4.5b/c reshape](0008-unicode-stdlib-pivot-and-reshape.md)
- [0009 — bre / vim backref `\1`-`\9`: review + exposure surface](0009-backref-review-and-exposure.md)
- [0010 — Surface freeze for v1.0](0010-surface-freeze.md)
