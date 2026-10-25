# the language gate: one shared check, coverage floor and security gate for seven languages, in build order

## The order

Three repositories, one feature. `.github` owns the contract (the
`quality.yml` inputs and the new `security-scan.yml` ecosystems, in
`.github/plans/02-language-gate.md` `## Contract`); `repo-template` builds
starters that call it; `project-setup` documents both.

- Each repository's own `plans/01-quality-gates.md` closes first, as its
  root `PLAN.md` says: the gate's scripts and test runner come from
  `.github`'s, and the other two need `make check` before they can call
  anything.
- The rough cuts go first and together: `.github` Phase 1 lands
  `quality.yml` for Python, then `repo-template` Phase 1 makes a Python
  repository from the template that calls it. The first try runs through
  both before either is deepened.
- `repo-template` Phases 2 and 3 wait on `.github` Phases 2 and 3, because
  the starters need the toolchain setup, the parsers and the new
  ecosystems to exist.
- `project-setup` waits until both are documented, so the skill describes
  what exists, with the input names fixed.
- The PRs come last, one per repository.

## Phases

### Phase 1: the rough cut, Python end to end

- [ ] `@.github/plans/02-language-gate.md` phase 1 - `quality.yml` with
  Python and the LCOV floor.
- [ ] `@repo-template/plans/02-language-gate.md` phase 1 - `init.sh python`
  and a repository that calls it.

### Phase 2: every language

- [ ] `@.github/plans/02-language-gate.md` phases 2-3 - toolchains,
  parsers, the new ecosystems and `fail-on`.
- [ ] `@repo-template/plans/02-language-gate.md` phases 2-3 - the six other
  starters.

### Phase 3: documented

- [ ] `@.github/plans/02-language-gate.md` phase 4 - the gate's README.
- [ ] `@repo-template/plans/02-language-gate.md` phase 4 - the template's
  README.
- [ ] `@project-setup/plans/02-language-gate.md` phases 1-2 - the skill
  teaches it.

### Phase 4: the PRs

- [ ] `@.github/plans/02-language-gate.md` phase 5 - the gate's PR.
- [ ] `@repo-template/plans/02-language-gate.md` phase 5 - the template's
  PR.
- [ ] `@project-setup/plans/02-language-gate.md` phase 3 - the skill's PR.
