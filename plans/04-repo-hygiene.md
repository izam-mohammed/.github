# the shared workflow's actions and pre-commit hooks stay current, and security reports have a private route

## The ask

The owner asked, on 22/10/2026, to "have dependabot in all of my repos" and
to "have proper security.md file for all of my repos". A licence was asked
only for repositories that are a product; this one holds shared CI for the
owner's other repositories, so it gets none.

The repository already has `.github/dependabot.yml`, covering
`github-actions` (the SHA-pinned actions in
`.github/workflows/security-scan.yml`). It also has `.pre-commit-config.yaml`
with two hook repositories pinned by tag (gitleaks, pre-commit-hooks), which
nothing updates. There is no `SECURITY.md`. Because this is the owner's
`.github` repository, a `SECURITY.md` here is also the fallback GitHub shows
for any of the owner's public repositories without their own.

## Alternatives

### Decision 1: which bot keeps dependencies current? (two-way door)

Dependabot already runs here for actions; the pre-commit hooks need the
same.

Searched 22/10/2026: "dependabot pre-commit ecosystem", "renovate
pre-commit manager"; GitHub's Dependabot options reference (docs.github.com,
supported ecosystems list: `pre-commit` is a `package-ecosystem` value that
reads hook revisions from git tags), Renovate's `pre-commit` manager docs.

| Must | Baseline: Dependabot | Renovate |
| --- | --- | --- |
| Updates `rev:` pins in `.pre-commit-config.yaml` | yes | yes |
| Runs with no app install or token to manage | yes | no (needs the Renovate app) |
| **Survives** | **yes** | no |

| Criterion | Weight | Baseline: Dependabot |
| --- | ---: | ---: |
| Performance efficiency | 40 | 4 |
| Reliability | 30 | 4 |
| Compatibility | 30 | 5 |
| **Weighted total** | **100** | **4.30** |

- Dependabot compatibility 5: it already runs here, so the pre-commit
  entry is one more block in the same file.

**Chosen: Dependabot.** It already runs here and supports `pre-commit`.
Revisit if its pre-commit support misses a hook repository.

## The decision

Extend `.github/dependabot.yml` with a `pre-commit` entry (weekly,
`cooldown: default-days: 7`, one group, prefix `chore(deps)`), modelled on
`decisionsmith/.github/dependabot.yml` (Decision 1). The existing
`github-actions` entry is left as it is. Add `SECURITY.md`: the repository
is public, so reports go through GitHub private vulnerability reporting,
and the policy says it is also the fallback for the owner's other public
repositories. No release tags exist and callers use `@main`, so only the
latest commit on `main` gets fixes. No licence. Dependencies and
permissions: none.

Security: no code, input, secret or permission changes; the change adds a
config block and one policy document.

## Where it lands

| Path | New or edit | Phase | What changes | Firm? |
| --- | --- | --- | --- | --- |
| `.github/dependabot.yml` | edit | 2 | adds the `pre-commit` ecosystem | fixed |
| `SECURITY.md` | new | 3 | the security policy | fixed |

*Reused, not rewritten:*

- `.github/dependabot.yml`: extended, not replaced.
- `decisionsmith/SECURITY.md`: the shape the new policy follows.

## Declined

- Renovate: needs an app install, and Dependabot already runs here.
- A licence: shared CI configuration, not a product.

## What the tests hold

The YAML parses; GitHub accepts the Dependabot config on the PR.

## Phases

### Phase 1: topics

- [x] The repository topics set to `ci`, `devsecops`, `github-actions`,
  `gitleaks`, `osv-scanner`, `reusable-workflows`, `security`, `semgrep`,
  `zizmor` with `gh repo edit`; check: `gh repo view --json
  repositoryTopics` lists them.

### Phase 2: Dependabot

- [x] `.github/dependabot.yml` gains a `pre-commit` entry beside the
  existing `github-actions` one; check: the file parses with
  `yaml.safe_load`.

### Phase 3: security policy

- [x] `SECURITY.md` lands with supported versions, the private reporting
  link, what happens next and the scope (`security-scan.yml` and the
  pre-commit baseline); check: every link in it points at this
  repository.

### Phase 4: open the PR

- [x] The branch pushed and its PR opened against `main` (or, if one is
  already open for this branch, its body brought up to date) with the
  title `chore: dependabot and security policy`, a body naming this plan,
  what changed and why, and the checks run with their result. The PR's
  URL printed as the evidence. Opened, never merged.
