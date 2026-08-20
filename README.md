# .github

Shared CI for repositories under [`izam-mohammed`](https://github.com/izam-mohammed).

## `security-scan.yml`

A reusable security gate. Stage 1 (secret detection) blocks a merge; stages 2–4 report
to the pull request without failing it.

| Stage | Tool | Gate |
|---|---|---|
| 1 · Secrets | gitleaks | **blocks** |
| 2 · Dependencies | osv-scanner, plus govulncheck on Go | reports |
| 3 · SAST | semgrep, rulepacks chosen by ecosystem | reports |
| 4 · Workflows | zizmor | reports |
| 5 · Report | sticky PR comment | — |

### Use it

Add `.github/workflows/security.yml` to the calling repository:

```yaml
name: Security
on:
  pull_request:
  schedule:
    - cron: '17 3 * * 1'

permissions: {}

jobs:
  scan:
    permissions:
      contents: read
      pull-requests: write
    uses: izam-mohammed/.github/.github/workflows/security-scan.yml@main
    with:
      ecosystem: python   # python | node | go | shell
```

No secrets are passed. The gate needs none, so callers must not add `secrets: inherit`.

`ecosystem` selects the semgrep rulepacks and turns on govulncheck for Go. Dockerfile rules
are added automatically when a `Dockerfile` is present.

### Tightening a stage

Stages 2–4 end their scan step with `|| true`. Remove it for the stage you want to enforce
once that repository's backlog is clear.

### Conventions

Workflows here keep `permissions: {}` at the top level and grant per job, set
`persist-credentials: false` on checkout, and pin every action to a full commit SHA with the
version in a trailing comment. Dynamic values reach `run:` and `github-script` bodies through
`env`, never through `${{ }}` interpolation — this file is checked with `zizmor` and reports
no findings.
