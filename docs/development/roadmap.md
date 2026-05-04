# niyama — Roadmap

> Milestone plan through v1.0. State lives in [`state.md`](state.md);
> this file is the sequencing — what ships, in what order, against
> what dependency gates.

## v1.0 criteria

niyama hits v1.0 when it's **fold-ready** — Cyrius stdlib can vendor
`dist/niyama.cyr` byte-identical as `lib/niyama.cyr` (sandhi
precedent at cyrius v5.7.0). Concrete bar:

- [ ] All five engines shipped (bre, re2, pcre, fuzzy, vim) with
      working compile + match + count + substitute paths.
- [ ] Public surface frozen — exported `niyama_*` symbols documented;
      no breaking changes after the freeze ADR (sandhi ADR 0005 is
      the template).
- [ ] Comprehensive test coverage per engine — unit tests + per-engine
      fuzz harness + cross-engine diff-test suite where flavors
      overlap.
- [ ] Benchmarks captured in `docs/benchmarks.md` — per-engine
      compile/match/search latency, per-engine pattern-corpus throughput,
      cross-engine comparison on a shared corpus.
- [ ] **≥2 long-horizon AGNOS-lineage consumers** actively using
      niyama (per the `feedback_placement_pushback` rule from cyim
      memory). cyim is consumer #1 once it links niyama; second
      consumer must materialize before fold.
- [ ] Security audit pass — `docs/audit/YYYY-MM-DD-audit.md`. Per
      first-party-standards § Security Hardening: input validation,
      buffer safety, syscall review, pointer validation, no command
      injection, no path traversal, known-CVE review (PCRE has a
      rich CVE corpus to reference).
- [ ] CHANGELOG complete from v0.1.0 onward.
- [ ] Fold ADR drafted (template: sandhi ADR 0002).

## Milestones

### M0 — Scaffold (v0.1.0) — ✅ shipped 2026-04-28

- `cyrius init` scaffold landed.
- Doc-tree per [first-party-documentation.md](https://github.com/MacCracken/agnosticos/blob/main/docs/development/applications/first-party-documentation.md).
- README positioning, CHANGELOG entry, ADR 0001 (positioning +
  sandhi-pattern fold lifecycle), `dist/niyama.cyr` placeholder.
- CI workflows (`.github/workflows/{ci,release}.yml`).

### M1 — bre engine (v0.2.0) — POSIX Basic Regular Expressions — ✅ shipped 2026-05-03

**Why first.** Smallest engine. Shakes out the niyama dispatch
surface — module organization, per-engine ABI naming, option-bag
struct extension points, error reporting — without diving into
PCRE complexity. Direct value: POSIX-strict tooling compatibility
for any consumer that needs `grep` / `sed` BRE semantics.

**Acceptance — done:**

- ✅ `niyama_bre_compile(pat)` → opaque NFA pointer (`0` on error;
  error code via `niyama_bre_last_error()`).
- ✅ `niyama_bre_match`, `niyama_bre_search`, `niyama_bre_search_at`,
  `niyama_bre_group_start`, `niyama_bre_group_end` (mirrors stdlib
  `regex_*` shape per ADR 0002).
- ✅ POSIX BRE syntax: literal-by-default metachars, `\(...\)`
  capturing groups (1..9), `\{n,m\}` quantifiers, `^`/`$` anchors
  at pattern boundaries only.
- ✅ `\1`-`\9` backreferences rejected at compile with
  `BRE_E_BACKREF_UNSUPPORTED` per ADR 0002 (deferred to potentially
  post-v1.0).
- ✅ `tests/bre.tcyr` — 68 assertions across 13 feature groups.
- ✅ `fuzz/bre.fcyr` — 200-iter randomized sweep + smoke corpus
  drawn from past stdlib regex bug-fix history.
- ✅ `tests/bre.bcyr` — bench floor recorded in CHANGELOG.
- 🟦 cyim-side: `--regex=bre` flavor — landing in cyim repo as a
  separate PR (one elif arm per cyim ADR 0002). Not blocking M2.

**Deferred from M1 to later milestones:**

- GNU `\<` / `\>` word boundaries → M4 (vim flavor inherits the
  same semantics; implement once).
- `[:alpha:]` POSIX bracket character classes → M4.
- Backref support → potentially post-v1.0, per ADR 0002.

### M2 — re2 engine (v0.3.0) — Thompson NFA, linear-time guarantee — ✅ shipped 2026-05-03

**Why second.** DoS-safe regex for untrusted patterns is a real
need (agnoshi log analysis, daimon agent pattern gates). RE2's
guarantee — O(N×M) match time regardless of pattern complexity, no
backreferences — is exactly what untrusted-input scenarios need.
Architecturally similar to stdlib's Pike NFA but with the safety
guarantees made explicit at the API level (any pattern that requires
backref or backtracking gets rejected at compile time).

**Acceptance — done:**

- ✅ Compile-time rejection of features that break linear-time
  guarantee. Each gets its own error code so consumers know which
  engine to fall back to: `RE2_E_BACKREF_UNSUPPORTED`,
  `RE2_E_LOOKAROUND_UNSUPPORTED`, `RE2_E_ATOMIC_UNSUPPORTED`,
  `RE2_E_RECURSION_UNSUPPORTED` (per ADR 0003).
- ✅ Linear-time complexity verified by benchmark on adversarial
  patterns. `(a|a)*b` against 200 `a`s, `(a*)*b` against 200 `a`s,
  Cox's `a?{30}a{30}` adversary all complete in <250μs.
- ✅ ERE conformance covered: literals, `.`, anchors, char classes,
  `\d`/`\w`/`\s`/`\b` predefined, alternation `|`, capturing +
  non-capturing groups, greedy + lazy `*` `+` `?` `{n,m}`. Bench
  floor recorded.
- ✅ `fuzz/re2.fcyr` adversarial pattern generator + 4
  rejection-invariant checks.
- 🟦 cyim-side: `--regex=re2` flavor — landing in cyim repo as a
  separate PR per cyim ADR 0002.

**Deferred from M2 to later milestones:**

- Named captures `(?P<name>...)` / `(?<name>...)` — deferred to
  post-M3 cross-engine surface consolidation (re2 + pcre share the
  named-capture API).
- Unicode property classes `\p{L}` — post-v1.0 unless asked.
- Inline flags `(?i)` `(?m)` `(?s)` — post-v1.0 unless asked.

### M3 — pcre engine (v0.4.0) — Perl-compatible

**Why third.** Largest surface, largest fuzz target. Comes after re2
so we have a battle-tested simpler engine to fall back to in
diff-testing.

**Acceptance:**

- PCRE feature set TBD per per-engine ADR. Initial scope: non-capturing
  groups `(?:...)`, named captures `(?<name>...)`, lookaround `(?=...)`
  / `(?<=...)`, atomic groups `(?>...)`, backreferences `\1`/`\2`,
  Unicode property classes `\p{L}`. Conditional patterns and recursion
  deferred unless demand surfaces.
- Strong fuzz coverage — PCRE is historically a CVE-rich engine;
  reference vim/PCRE CVE history for known patterns.
- Diff-test against re2 + bre on overlapping syntax.
- cyim-side: `--regex=pcre` flavor.

### M3.5 — fuzzy engine (v0.5.0) — Levenshtein / typo-tolerant

**Why folded in.** Not strict regex but lives in the same
pattern-matching neighborhood. agnoshi shell completion (typo-tolerant
command match) and daimon agent fuzzy-name-match are immediate
consumers; cyim's `:e <prefix>` completion is a plausible third.

**Acceptance:**

- Edit-distance threshold parameter (default 2 typos).
- Optional case-folding, optional Unicode-aware (NFD-normalize before
  match).
- Substring-fuzzy AND prefix-fuzzy modes (different consumer needs).
- cyim-side: `--regex=fuzzy` flavor.

### M4 — vim engine (v0.6.0) — vim/cyim flavor

**Why fourth.** Consumer-driven — cyim's `:s/old/new/` and ex-mode
pattern history will eventually want vim-flavor compatibility for
muscle-memory continuity.

**Acceptance:**

- Magic / nomagic / very-magic / very-nomagic modes (mode flag in the
  RegexOpts struct).
- vim metachars: `\<` / `\>` word boundaries, `\zs` / `\ze`
  match-start/end markers, `\(` / `\)` groups (nomagic-style),
  `&` in replacement = full match.
- vim-style char classes (`[[:alpha:]]` etc. — POSIX bracket
  expressions).
- cyim-side: `--regex=vim` flavor.

### M5 — P(-1) hardening + closeout (v0.7.x → freeze)

**Why before fold.** Per first-party-standards § Security Hardening
(required before every release) plus the cyim-style closeout pass
(refactor / cleanup / dead-code audit / surface freeze):

- Full audit pass per first-party-standards (input validation, buffer
  safety, syscall review, pointer validation, no command injection,
  no path traversal, known-CVE review).
- Cross-engine refactor — consolidate any parallel codepaths that
  accreted across M1–M4.
- Dead-code audit; record floor in CHANGELOG.
- Surface freeze ADR (template: sandhi ADR 0005).
- Comprehensive bench history captured.

### v1.0 — Fold-ready release

`dist/niyama.cyr` is byte-identical fold candidate. Cyrius stdlib
vendors as `lib/niyama.cyr` per the sandhi v5.7.0 lifecycle. niyama
repo enters maintenance mode; fold ADR (template: sandhi ADR 0002)
records the decision.

**Speculative cyrius fold target: 5.8.0** — *conditional on
multi-consumer adoption*, NOT a deadline. If adoption isn't there,
niyama stays out-of-tree until it earns the fold.

## Out of scope (for v1.0)

- **Cloud / hosted regex services.** niyama is in-process matchers
  only.
- **Regex-DSL extensions (e.g. structural regex à la Sam editor).**
  Possible post-v1.0 if a consumer asks; not a v1.0 commitment.
- **Regex generation / inverse-regex (synthesizing strings from
  patterns).** Plausible but distinct enough to warrant its own repo
  (`viparyaya`?) if/when needed.
- **GUI / TUI tooling** for regex testing. Out of scope; that's a
  consumer concern (cyim could build a `:RegexTest` mode).
- **Engine-on-engine recursion** (e.g. PCRE's recursive patterns
  calling into RE2 via niyama). Not planned; engines stay
  independent.
- **Backwards-compat shims for non-AGNOS regex APIs** (e.g. wrapping
  PCRE2's C API). niyama is sovereign Cyrius — if a consumer needs
  PCRE2's wire API, they wrap it themselves.
