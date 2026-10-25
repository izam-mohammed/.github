# .github: what every change passes

Run everything: there is no single check command yet.
`plans/01-quality-gates.md` lands `make check` and a CI workflow that runs
it on every pull request. Until then a change passes when the checks below,
run by hand, are clean.

Recorded on 22/10/2026 from what the repository already runs; nothing here
is new.

## Toolchain

| What | Version | Pinned in |
| --- | --- | --- |
| GitHub Actions workflow syntax | n/a | `.github/workflows/security-scan.yml` |
| pre-commit hooks | gitleaks v8.30.1, pre-commit-hooks v6.0.0 | `.pre-commit-config.yaml` |
| Tools the gate installs at run time | gitleaks 8.30.1, osv-scanner 2.5.1 | `env:` in `.github/workflows/security-scan.yml` |
| Tools the gate installs unpinned | semgrep, zizmor (`pip install`), govulncheck (`@latest`) | `.github/workflows/security-scan.yml` |

## Checks

| Check | Tool | Config | Command | Passing means |
| --- | --- | --- | --- | --- |
| Secret scan (local) | gitleaks via pre-commit | `.pre-commit-config.yaml` | `pre-commit run --all-files` | no findings |
| Hygiene (local) | pre-commit-hooks: private key, large files, merge and case conflicts, end of file, trailing whitespace | `.pre-commit-config.yaml` | `pre-commit run --all-files` | no findings |
| Workflow security | zizmor | none | `zizmor .github/workflows` (by hand) | no findings; 0 findings and 1 suppressed with zizmor 1.31.1 on 22/10/2026 |
| Action updates | Dependabot | `.github/dependabot.yml` | weekly | PRs reviewed and merged |

Not run today, and planned in `plans/01-quality-gates.md`: workflow
correctness lint (actionlint), a test that calls the reusable gate, and CI
on this repository's own pull requests.

## Tests

None. The gate has only been exercised by its callers' pull requests and
scheduled runs (for example `repo-template` and `project-setup`, both
green on their scheduled run of 19/10/2026).

Coverage: not applicable. The repository holds workflow YAML and one inline
`github-script` body; there is no library code a line-coverage tool can
measure. Its tests are fixture repositories run through the gate, planned in
`plans/01-quality-gates.md`.

## Constraints

- Shared by every repository under `izam-mohammed`: 19 checkouts under
  `~/fun` call `security-scan.yml` on 22/10/2026, pinned at three different
  commits. A change to an input is a contract change: additive only, with a
  default that keeps today's behaviour.
- The gate needs no secrets, and callers must not pass `secrets: inherit`.
- It must run on GitHub-hosted `ubuntu-latest` and on a self-hosted runner
  with no sudo (`runs-on` input, PR `#2`).

## Conventions

From `README.md` § *Conventions*, held by zizmor:

- `permissions: {}` at the top level, granted per job.
- `persist-credentials: false` on every checkout.
- Every action pinned to a full commit SHA with the version in a trailing
  comment.
- Dynamic values reach `run:` and `github-script` through `env:`, never
  through `${{ }}` interpolation in the body.
- Commits: Conventional Commits with a scope (`fix(security-scan): ...`,
  `ci(deps): ...`), lowercase subject; changes land through pull requests.
- No attribution lines in commits or PRs.

## CI

No workflow runs on this repository's own pull requests. `security-scan.yml`
is `workflow_call` only, and Dependabot is the only automation that runs
here.

## Decided

- gitleaks blocks; osv-scanner, govulncheck, semgrep and zizmor report only
  (`ce1b7cc`, README § *Tightening a stage*).
- Tools installed into `$RUNNER_TEMP/bin`, no sudo (PR `#2`).
