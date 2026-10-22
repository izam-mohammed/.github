# a change to the shared gate is linted and proved by its own check before any caller runs it

## The ask

Every repository should have proper lint, tests, a coverage floor and
security checks, run by one local check command and by CI on every pull
request (brief of 22/10/2026). This repository is the one the others reuse,
so its own bar comes first: today nothing runs on its pull requests, zizmor
is run by hand, and a change to `security-scan.yml` is first exercised by
the 19 callers' runs after it merges.

Built so far: `docs/built/BUILT-2026-10-22-1828.md`.

Coverage: not applicable here. The repository is workflow YAML plus small
POSIX `sh` scripts this plan extracts; no line-coverage tool measures a
GitHub Actions workflow, and POSIX `sh` coverage tools are unreliable
(kcov reported unable to cover bash 5.2 scripts on Ubuntu 24.04; bashcov
needs Ruby and bash). The stand-in is a case table: every script has a
fixture case for each branch it takes, listed in `## What the tests hold`.

## Alternatives

### Decision 1: what lints the workflows (two-way door)

zizmor already holds the conventions, by hand. The question is what runs in
the check, and whether workflow correctness (expression types, `needs`,
input names, shell in `run:`) is linted too.

Searched 2026-10-22: "actionlint vs zizmor GitHub Actions workflow linting
security"; the WordPress GitHub Actions workflow standards (runs both),
zizmor's docs page on CodeRabbit, actionlint's README.

| Criterion | Baseline: zizmor by hand | zizmor in the check | actionlint and zizmor in the check |
| --- | --- | --- | --- |
| Performance efficiency | nothing runs | under 1 s on one file | under 2 s together on one file (both static, no network with `--offline`) |
| Fit to the ask | security only, and only if remembered | security only | correctness and security |
| Reliability | misses a typo in `needs` or an input name | same miss | actionlint type-checks expressions and reusable-workflow inputs |

- zizmor 1.31.1 ran on `security-scan.yml` in about a second here on
  22/10/2026 (0 findings, 1 suppressed).
- actionlint validates inputs of reusable workflows and runs ShellCheck on
  `run:` bodies (WordPress standards page); zizmor covers template
  injection, permissions, pinning and credential persistence. They overlap
  little.

**Chosen: actionlint and zizmor in the check.** The only option that
catches a broken `workflow_call` input before callers do. Revisit if
zizmor grows correctness rules that make actionlint redundant.

### Decision 2: where the gate's logic lives, so it can be tested (two-way door)

Every stage today is inline shell in YAML, so a test can only run the whole
workflow on GitHub.

Searched 2026-10-22: "reusable workflow vs composite action GitHub Actions
differences 2026", "testing reusable GitHub Actions workflows fixtures act
nektos local run"; GitHub's docs on reusing workflow configurations, act's
documentation via the Ethereum execution-specs guide.

| Must | Baseline: inline YAML | Scripts in `scripts/gate/`, workflow calls them | Inline YAML tested with `act` |
| --- | --- | --- | --- |
| A case runs locally with no Docker | no | yes | no (act needs Docker) |
| Runs without sudo on a self-hosted runner (PR `#2`) | yes | yes | yes |
| **Survives** | no | **yes** | no |

**Chosen: scripts in `scripts/gate/`, called by the workflow.** The only
option whose cases run in `make check` on a laptop. The workflow keeps its
jobs, permissions and env passing; only the shell bodies move.

### Decision 3: what runs the script tests (two-way door)

Searched 2026-10-22: "bats-core vs shunit2 shell script testing
framework"; the 2026 comparison of Bats, ShellSpec and shUnit2, the
OpenStack thread on choosing Bats.

| Criterion | Weight | Plain `sh` runner | bats-core | shUnit2 vendored |
| --- | ---: | ---: | ---: | ---: |
| Performance efficiency | 40 | 5 | 4 | 5 |
| Portability | 30 | 5 | 3 | 5 |
| Maintainability of the result | 30 | 4 | 5 | 4 |
| **Weighted total** | **100** | **4.70** | **4.00** | **4.70** |

- Plain `sh` and shUnit2 efficiency 5: no interpreter beyond `sh`; bats 4
  forks a bash per test.
- bats portability 3: needs bash 3.2 or later and an install step; the house
  rule is POSIX `sh` scripts that need no setup step.
- bats maintainability 5: `run` and `assert` helpers; the others hand-roll
  two helpers.

Sensitivity: plain `sh` and shUnit2 tie; broken on running cost, a
vendored third-party file against none.

**Chosen: plain `sh` runner (`tests/run.sh`).** Ties shUnit2 and adds no
file to keep updated. Revisit past about 40 cases.

### Decision 4: the one check command (two-way door)

Searched 2026-10-22: "just command runner vs make for project task
runner"; the Atomic Object and Linux Magazine write-ups on just, and
project-setup's `SKILL.md` step 8 (same target names in every repo).

| Criterion | Make (`make check`) | `scripts/check.sh` | just |
| --- | --- | --- | --- |
| Performance efficiency | instant | instant | instant |
| Compatibility | the house target names (`project-setup` step 8) | no target names | a new binary on every machine |
| Portability | on every runner and Mac | same | install needed |

**Chosen: `make check`.** It matches the target names every other repo
uses. Revisit never, unless the house style changes.

## The decision

- actionlint and zizmor lint the workflows in `make check`
  (decision 1).
- The gate's shell moves into `scripts/gate/*.sh`, POSIX `sh`, and the
  workflow calls them (decision 2). Behaviour does not change: same tools,
  same exit codes, same artifact names.
- `tests/run.sh` runs fixture cases for each script (decision 3); fixtures
  that need a secret-looking string generate it at run time into a temp
  directory, so the repository never holds one.
- `make check` runs pre-commit, actionlint, zizmor and `tests/run.sh`
  (decision 4); `.github/workflows/check.yml` runs it on every pull request
  and on push to `main`, and a second job calls the gate itself with
  `uses: ./.github/workflows/security-scan.yml` as a smoke test.
- semgrep, zizmor and govulncheck are pinned to versions in the gate's `env`
  block, like gitleaks and osv-scanner already are.

New dependencies: actionlint (Go binary from its GitHub release, pinned by
version and SHA-256) and zizmor (from PyPI through `uvx`, pinned) in the
check; no new permission beyond `contents: read` for `check.yml`, and the
existing `pull-requests: write` for the smoke job's sticky comment.

Standards: `docs/standards.md` (this plan replaces its "no check command"
with `make check` and updates its CI section in the same phase).

## Where it lands

| Path | New or edit | Phase | What changes | Firm? |
| --- | --- | --- | --- | --- |
| `Makefile` | new | 1 | `check`, `lint`, `test` targets | fixed |
| `tests/run.sh` | new | 1 | the case runner, prints PASS or FAIL per case | expected |
| `scripts/gate/` | new | 1 | the stages' shell, moved out of YAML | expected |
| `.github/workflows/security-scan.yml` | edit | 1 | steps call `scripts/gate/*.sh`; tool versions pinned in `env` | reuse |
| `.github/workflows/check.yml` | new | 2 | `make check` and the self-call smoke job | expected |
| `docs/standards.md` | edit | 1 | the check command, CI | reuse |
| `README.md` | edit | 2 | "Develop it" section: `make check` | reuse |

*Reused, not rewritten:*

- `.github/workflows/security-scan.yml`: kept as the one gate; its job
  layout, permissions, artifacts and sticky comment stay; only the bodies
  of `run:` steps move.
- `.pre-commit-config.yaml`: run as-is inside `make check`.
- `project-setup/templates/Makefile.python` `security` target: the same
  gitleaks, osv-scanner, semgrep and zizmor commands; the new Makefile
  reuses their flags rather than inventing new ones.

## Security

What is protected: every caller's pull request gate, and through it their
code. Who can reach it: anyone who can open a PR here (public repository);
`check.yml` runs on `pull_request`, never `pull_request_target`, so a fork's
PR gets a read-only token and no secrets.

| Threat | Here | Handled by | Proved by |
| --- | --- | --- | --- |
| Tampering: a tool release swapped upstream | the gate downloads binaries at run time | versions pinned in `env`, SHA-256 checked before install | a case in `tests/run.sh` feeds a wrong checksum and expects a refusal |
| Tampering: an unpinned action | new `check.yml` | every action pinned to a full SHA | zizmor in `make check` reports 0 findings |
| Elevation of privilege: a workflow writes with a broad token | new `check.yml` | `permissions: {}` at top, `contents: read` per job | zizmor and actionlint clean; review of the job permissions |
| Information disclosure: a real secret in fixtures | secret-detection cases | fixture strings generated at run time into `$TMPDIR`, never committed | gitleaks over the tree in `make check` stays clean |

Secrets: none; the gate and the check need none. Logged: tool output only.
Checklist: the CIS GitHub benchmark items for workflows and SLSA build L1
(scripted, versioned build steps), searched for again when Phase 2 lands.

## Declined

- zizmor alone, and the by-hand baseline: miss workflow correctness
  errors such as a wrong reusable-workflow input name (decision 1).
- Inline YAML and inline YAML tested with `act`: no Docker-free local case
  (decision 2).
- bats-core: lost on portability and efficiency (decision 3); shUnit2 tied
  and lost the tie-break on carrying a vendored file.
- `scripts/check.sh` and just: break the house target names (decision 4).

## What the tests hold

- `tests/run.sh` covers each `scripts/gate/` script: the secret stage exits
  1 on a generated fake key and 0 on a clean tree; the dependency count
  reads a sample `osv.json` and an absent one as 0; the rulepack selection
  prints the right packs for each ecosystem and adds Dockerfile rules only
  when a `Dockerfile` exists; a wrong checksum is refused.
- The smoke job proves the reusable workflow still runs end to end and
  posts its comment, on this repository, before callers see the change.

## Repository knowledge

Read root `KNOWLEDGE.md` when present before each phase. Follow the global
and repository instructions for recording useful development findings.

## Phases

### Phase 1: the rough cut, `make check` on a laptop (needs you: try it)

- [ ] `make check` runs pre-commit, actionlint and zizmor over
  `.github/workflows/`, then `tests/run.sh` (expected in `Makefile` and
  `tests/run.sh`), each tool fetched to a cache directory at a pinned
  version when missing.
- [ ] The secret, dependency and rulepack steps of `security-scan.yml` move
  into `scripts/gate/secrets.sh`, `scripts/gate/deps.sh` and
  `scripts/gate/packs.sh`, POSIX `sh`, called from the workflow with the
  same inputs through `env`; pinned by the cases in `tests/run.sh`.
- [ ] `docs/standards.md` names `make check` as the command and lists the
  new rows.
- [ ] Check: `make check` passes, and `sh tests/run.sh` prints PASS for
  every case.
- [ ] You try it: run `make check` in a clone, then break a `needs:` name in
  `security-scan.yml` and run it again; say whether the failure is clear.

### Phase 2: CI runs it, and the gate calls itself

- [ ] `.github/workflows/check.yml` runs `make check` on every pull request
  and push to `main`, read-only, actions pinned by SHA (expected in
  `.github/workflows/check.yml`).
- [ ] A `smoke` job in the same file calls
  `./.github/workflows/security-scan.yml` with `ecosystem: shell` and
  passes when every stage completes.
- [ ] `README.md` gains a short "Develop it" section naming `make check`.
- [ ] Check: `make check` passes, and the PR's `check` and `smoke` jobs are
  green with the sticky comment posted.

### Phase 3: the gate runs the same tools every time

- [ ] semgrep, zizmor and govulncheck are pinned by version in the gate's
  `env` block and installed through `scripts/gate/`, with a checksum for
  every downloaded binary (expected in `.github/workflows/security-scan.yml`
  and `scripts/gate/install.sh`), pinned by the wrong-checksum case.
- [ ] Check: `make check` passes, and the smoke job prints the pinned
  versions in its log.

### Phase 4: open the PR

- [ ] The branch pushed and its PR opened against the branch it was cut
  from (or, if one is already open for this branch, its body brought up to
  date) with a Conventional Commit title for the whole change, a body
  naming this plan, what changed and why, and the checks run with their
  result. The PR's URL printed as the evidence. Opened, never merged.
