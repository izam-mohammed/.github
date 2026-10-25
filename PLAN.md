# .github: what is left to build, in order

What is built is in `docs/built/BUILT-2026-10-22-1828.md`. The standards
every change passes are in `docs/standards.md`.

Not planned, because it does not apply: a `site/` one-pager (the
repository is shared CI whose only readers are the repositories calling
it, and they read the README on GitHub) and a line-coverage floor (there is
no program code to measure; fixture cases stand in, see
`plans/01-quality-gates.md`).

The language gate spans three repositories; their order is
`plans/master-language-gate.md`.

### Phase 1: the gate checks itself

- [ ] `plans/01-quality-gates.md` - `make check` with actionlint, zizmor and
  script tests, CI on every PR, and pinned tool versions.

### Phase 2: every language reaches the bar through the gate

- [ ] `plans/02-language-gate.md` - `quality.yml` with a 90 % coverage
  floor, Rust, Swift and Kotlin in the security gate, opt-in blocking.

### Phase 3: the README

- [ ] `plans/03-readme-banner.md` - a banner pair at the top of the README.
