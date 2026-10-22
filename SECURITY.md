# Security policy

## Supported versions

This repository has no releases, and callers use the reusable workflow at
`@main`. Only the latest commit on `main` gets fixes.

## Reporting a vulnerability

Please do not open a public issue, pull request or discussion for a security
problem.

Report it privately through GitHub:
[Report a vulnerability](https://github.com/izam-mohammed/.github/security/advisories/new)
(the "Security" tab, then "Report a vulnerability").

GitHub also shows this policy on any of the owner's public repositories that
have no `SECURITY.md` of their own. For a problem in one of those, use the
"Report a vulnerability" button in that repository's own Security tab.

Helpful details:

- the workflow or file, and the commit you looked at
- what an attacker could do, and what they need first (for example a crafted
  pull request against a calling repository)
- the smallest steps that show it

## What happens next

This is a project with one maintainer, so there are no guaranteed response
times. We aim to acknowledge the report, agree on how serious it is, fix it
on `main`, and publish a GitHub security advisory that credits you (unless
you would rather not be named).

## Scope

In scope: the reusable workflow `.github/workflows/security-scan.yml` (for
example a way to run code, read a token or write to a calling repository it
should not) and the pre-commit baseline in `.pre-commit-config.yaml`.

A problem in a tool the workflow runs (gitleaks, osv-scanner, govulncheck,
semgrep, zizmor) belongs to that tool's own project; tell us too if the way
the workflow uses it makes things worse.
