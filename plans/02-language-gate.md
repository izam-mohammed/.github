# a repository in any of seven languages gets its check, a 90 % coverage floor and the security gate by calling the shared workflows

## The ask

Every repository should reach proper lint and format, tests, a coverage
floor of 90 % (target 95 %) and security checks (dependency audit, secret
scan, static analysis where the language has one), run by one local check
command and by CI on every pull request, by reusing the shared gate here
rather than inventing a parallel one (brief of 22/10/2026). The repositories
that will call it are written in Python, TypeScript, Rust, Go, Swift,
Kotlin and shell.

What it turned out to mean once the code was read: the gate today covers
only security, for four ecosystems (`python`, `node`, `go`, `shell`), and
cannot block on anything but a secret. Nothing shared runs a repository's
lint and tests or holds a coverage floor. Repositories in the missing
languages already work around the gap: the plans written on 22/10/2026 for
`mac-notch` (Swift), `money-logs` (Kotlin and Rust), `car-mate` (Kotlin),
`turbogit` and `agent-eye` (Rust) call the gate as `shell` or `node` and
list the new ecosystem values as a follow-up waiting on this plan. This plan is the `.github` half of
a feature across three repositories; the order is
`plans/master-language-gate.md`. `repo-template` ships the per-language
starters that call what this plan lands, and `project-setup` documents it.

Built so far: `docs/built/BUILT-2026-10-22-1828.md`.

## Constraints

- The existing gate is reused, never a parallel one beside it.
- Site and coverage are marked not applicable where they truly do not
  apply: a caller must be able to say coverage does not apply, visibly,
  never silently.
- 19 callers pin today's gate; nothing they rely on may change meaning.

## Alternatives

### Decision 1: how lint, tests and coverage are offered (one-way door)

Callers will build on whatever shape this takes, so it is a contract.

Searched 2026-10-22: "reusable workflow vs composite action GitHub Actions
differences 2026"; GitHub's docs on reusing workflow configurations (jobs,
runners, outputs, four nesting levels) and its blog post on reusable
workflows.

| Must | Baseline: nothing shared, each repo writes its own CI | `quality.yml` runs the caller's own check command | One reusable workflow per language running fixed tools | Composite action per language | Stages added to `security-scan.yml` |
| --- | --- | --- | --- | --- | --- |
| The same command runs locally and in CI | per repo, often not | yes (the caller's `make check`) | no (CI runs tools the repo may not run locally) | yes | yes |
| Picks its own runner (Swift needs macOS) | yes | yes (`runs-on`) | yes | no (runs in the caller's job) | yes |
| Leaves the 19 security callers untouched | yes | yes | yes | yes | no (every caller would start running tests) |
| **Survives** | yes | **yes** | no | no | no |

| Criterion | Weight | Baseline: each repo its own CI | `quality.yml` runs the caller's check |
| --- | ---: | ---: | ---: |
| Performance efficiency | 35 | 4 | 4 |
| Fit to the ask | 25 | 2 | 5 |
| Reliability | 20 | 2 | 4 |
| Maintainability of the result | 20 | 2 | 5 |
| **Weighted total** | **100** | **2.70** | **4.45** |

- Efficiency 4 for both: one job running the check; `quality.yml` adds one
  `awk` pass over one report (expected under a second; measured in Phase 1).
- Baseline fit 2: there is no shared floor; `mac-notch` holds its own 90 %
  in `scripts/coverage-floor`, and whether others hold one is unverified.
- Reliability: one shared floor and toolchain setup, fixed once, against
  19 copies that drift (the security pins already drifted to three commits).

Sensitivity: the winner holds even with efficiency at 60; the baseline
never wins fit. Risk: a broken `quality.yml` breaks every caller at once;
limited by callers pinning a SHA or tag and by the smoke tests of
`plans/01-quality-gates.md`.

**Chosen: `quality.yml` runs the caller's check command.** The repository
keeps one command for laptop and CI; the shared part is the runner setup,
the floor and the report. Revisit if two callers need a second job shape.

### Decision 2: how the coverage floor is enforced (one-way door)

Seven languages produce four report formats: LCOV (coverage.py, Vitest,
cargo-llvm-cov, `llvm-cov export` for Swift), Go's cover profile, JaCoCo XML
(Kover for Kotlin) and Cobertura XML (bashcov or kcov for shell).

Searched 2026-10-22: "GitHub Actions enforce code coverage threshold LCOV
Cobertura multiple languages without Codecov", "cargo-llvm-cov
fail-under-lines option", "Kover koverVerify minBound line coverage
Gradle", "go-test-coverage threshold", "swift llvm-cov export lcov", "bash
shell script code coverage kcov bashcov bats"; the Very Good Coverage and
coverage-scope READMEs, cargo-llvm-cov's README, the Kotlin docs on Kover,
the Swift forums threads on `llvm-cov export`.

| Must | Baseline: each language's own fail-under flag only | Gate parses the report (`scripts/gate/coverage.sh`) | Very Good Coverage action | Codecov |
| --- | --- | --- | --- | --- |
| Reads all four formats | n/a | yes | no (LCOV only) | yes |
| No secret and no code data sent to a third party | yes | yes | yes | no (upload token, report leaves GitHub) |
| A caller cannot lower the floor without it showing | no (the flag sits in the caller's config) | yes (the floor is printed in the comment, 90 by default) | yes | yes |
| **Survives** | no | **yes** | no | no |

Only one option survives, so no weighted table. The native flags still
stay in each repository's `make check` (repo-template's starters set them
to 90), so the laptop fails first; the gate re-reads the report so CI shows
one number in one place.

Risk: a parser misreads a format and passes a low number. Limited by a
fixture report per format with a known total in `tests/run.sh`, checked
against the native tool's own total.

**Chosen: the gate parses the report itself, in POSIX `sh` and `awk`.**
Revisit if a caller needs per-file floors (go-test-coverage has them).

### Decision 3: how Rust, Swift and Kotlin get security checks (two-way door)

Searched 2026-10-22: "semgrep registry rulesets p/rust p/swift p/kotlin",
"osv-scanner supported lockfiles Cargo.lock Package.resolved
gradle.lockfile"; Semgrep's Kotlin and Swift language docs and registry tag
page, osv-scanner's supported lockfiles page and the v2.4.0 notes on Swift.

| Criterion | Baseline: no rulepack, osv-scanner as today | Language rulepacks added, osv-scanner as today | Rulepacks plus cargo-audit or cargo-deny for Rust |
| --- | --- | --- | --- |
| Performance efficiency | fastest | one more pack per run | a second advisory scan of `Cargo.lock` |
| Fit to the ask | no static analysis for three languages | static analysis where Semgrep has rules | same, plus licence checks nobody asked for |
| Compatibility | as today | same osv-scanner step for all | a Rust-only step |

- osv-scanner 2.5.1 (pinned in the gate) reads `Cargo.lock`,
  `gradle.lockfile` and `Package.resolved` (Swift since 2.4.0).
- Semgrep lists Kotlin and Swift default rulesets and a general Rust
  ruleset; the exact slugs are confirmed against the registry in Phase 3
  before they are written into the gate.

**Chosen: language rulepacks added, osv-scanner as today.** One advisory
source for every language. Revisit if osv-scanner misses a RustSec advisory
that cargo-audit reports.

### Decision 4: whether stages 2 to 4 can block (two-way door)

Searched 2026-10-22: "dependabot updates reusable workflow SHA pin";
GitHub's changelog of 13/03/2023 on Dependabot updating reusable workflows
by SHA or tag.

| Criterion | Baseline: report only | A `fail-on` input, default `none` | Flip every stage to blocking |
| --- | --- | --- | --- |
| Fit to the ask | security reported, never enforced | enforced where a repository opts in | enforced everywhere |
| Compatibility | as today | 19 callers unchanged | 19 callers may go red at once |
| Reliability | n/a | a repository opts in when its backlog is clear | red for backlog, not for the change |

**Chosen: a `fail-on` input, default `none`.** `high` fails stages 2 and 3
on a high or critical finding. The starters in `repo-template` set it to
`high`. Revisit when every caller has opted in, then flip the default.

## The decision

- New reusable workflow `.github/workflows/quality.yml` (decision 1): sets
  up the language's toolchain, runs the caller's check command, parses the
  coverage report against the floor (decision 2) and writes a sticky
  comment and job summary.
- `security-scan.yml` gains `rust`, `swift` and `kotlin` ecosystems
  (decision 3) and a `fail-on` input (decision 4). Every existing input keeps
  its meaning.
- A release tag, `v0.2.0`, once this lands, so callers can pin a tag and
  Dependabot can bump their SHA pins with the version comment.

New dependencies: `actions/setup-node`, `dtolnay/rust-toolchain` or the
runner's rustup (decided in Phase 2 by the same method), `actions/setup-java`
for Kotlin, each pinned by SHA. No new permission: `quality.yml` asks
`contents: read` and `pull-requests: write` for its comment. No secret.

Standards: `docs/standards.md` (`make check`: pre-commit, actionlint,
zizmor and `tests/run.sh`, from `plans/01-quality-gates.md`).

## Contract

Owned by this repository; read by every caller's `.github/workflows/check.yml`
and `security.yml`, and by `@repo-template/plans/02-language-gate.md`.
Additive only: an input added later has a default that keeps today's
behaviour. Breaking changes need a new major tag.

`quality.yml` inputs:

| Input | Type | Default | Meaning |
| --- | --- | --- | --- |
| `language` | string, required | | `python`, `node`, `rust`, `go`, `swift`, `kotlin`, `shell` |
| `check-command` | string | `make check` | run with `sh -c` from the repository root, passed through `env` |
| `coverage-file` | string | `''` | report path inside the workspace; no `..`, not absolute |
| `coverage-format` | string | `lcov` | `lcov`, `gocover`, `jacoco`, `cobertura` |
| `coverage-floor` | string | `90` | a number 0 to 100, or `none`; `none` needs `coverage-reason` |
| `coverage-reason` | string | `''` | shown in the comment when the floor is `none` |
| `runs-on` | string (JSON) | `"ubuntu-latest"` | as in `security-scan.yml` |
| `python-version`, `node-version`, `go-version`, `java-version` | string | `3.12`, `22`, `1.25`, `21` | toolchain pins; Rust and Swift read `rust-toolchain.toml` and the runner's Xcode |

Output: `coverage`, the total line (or statement) percentage to one
decimal, or `n/a`. The job fails when the check command fails, when the
report is missing while a floor is set, or when the total is under the
floor.

`security-scan.yml` additions: `ecosystem` also accepts `rust`, `swift`,
`kotlin`; new input `fail-on`, `none` (default) or `high`.

| Language | Coverage tool the starter uses | Format |
| --- | --- | --- |
| Python | coverage.py (`coverage lcov`) | `lcov` |
| TypeScript | Vitest coverage, `lcov` reporter | `lcov` |
| Rust | cargo-llvm-cov `--lcov` | `lcov` |
| Go | `go test -coverprofile` | `gocover` |
| Swift | `swift test --enable-code-coverage`, then `llvm-cov export -format=lcov` | `lcov` |
| Kotlin | Kover XML report | `jacoco` |
| Shell | bashcov or kcov, Cobertura output | `cobertura` |

## Security

What is protected: callers' merge gates, and the token their check runs
with. Who can reach it: a caller's own workflow passes the inputs; a fork's
PR runs it with the caller's read-only `pull_request` token.

| Threat | Here | Handled by | Proved by |
| --- | --- | --- | --- |
| Tampering: code built from input | `check-command` and `coverage-file` | both pass through `env`, never `${{ }}` in a body; the command is the caller's own, as trusted as its `run:` lines | zizmor 0 findings in `make check` |
| Information disclosure: a path outside the workspace read | `coverage-file` | rejected when absolute or holding `..` | `tests/run.sh` case: `../etc/passwd` refused |
| Elevation of privilege | `quality.yml` permissions | `permissions: {}` top, `contents: read` and `pull-requests: write` on the comment job only | actionlint and zizmor clean |
| Repudiation: the floor lowered quietly | `coverage-floor` | the floor and any `none` reason printed in the comment and summary | `tests/run.sh` case: `none` without a reason fails |
| Denial of service: a huge report | `coverage-file` | the parser reads one pass with `awk`; the job keeps `timeout-minutes` | fixture of 100,000 LCOV lines parses within the job timeout |

Secrets: none, and callers must not pass `secrets: inherit`. Logged: the
command's own output, the coverage total, never environment values.
Checklist: CIS GitHub benchmark workflow items and SLSA build L1, searched
for again when Phase 2 lands.

## Where it lands

| Path | New or edit | Phase | What changes | Firm? |
| --- | --- | --- | --- | --- |
| `.github/workflows/quality.yml` | new | 1 | the reusable check workflow | fixed |
| `scripts/gate/coverage.sh` | new | 1 | report parser and floor | expected |
| `scripts/gate/setup.sh` | new | 2 | per-language toolchain extras | expected |
| `tests/fixtures/coverage/` | new | 1 | one small report per format with a known total | expected |
| `.github/workflows/security-scan.yml` | edit | 3 | new ecosystems and rulepacks (through `scripts/gate/packs.sh` from plan 01), `fail-on` | reuse |
| `README.md` | edit | 4 | `quality.yml` usage and the language table | reuse |

*Reused, not rewritten:*

- `.github/workflows/security-scan.yml` `report` job: the sticky comment
  pattern (marker, `env` only, update in place) is copied in shape for the
  quality comment, with its own marker so the two never overwrite each
  other.
- `scripts/gate/` and `tests/run.sh` from `plans/01-quality-gates.md`: the
  parser is one more script there, tested by the same runner.
- The `runs-on` input of PR `#2`: same JSON form, same default.

## Declined

- One workflow per language running fixed tools: CI would run tools the
  repository does not run locally (decision 1).
- Composite actions: cannot choose their own runner (decision 1).
- Adding stages to `security-scan.yml`: every current caller would start
  running tests (decision 1).
- Native fail-under flags alone: the floor lives in each caller's config
  and can drop unseen (decision 2).
- Very Good Coverage: LCOV only (decision 2).
- Codecov: needs an upload token and sends reports off GitHub (decision 2).
- cargo-audit or cargo-deny: a second advisory source for one language
  (decision 3).
- Every stage blocking by default: would turn 19 callers red at once
  (decision 4).

## What the tests hold

- `tests/run.sh` reads each fixture report and prints the total that
  fixture was written to hold (one under 90, one at exactly 90, one over),
  fails under the floor and passes at or above it,
  refuses a path outside the workspace, and refuses `none` without a
  reason.
- The smoke job of `plans/01-quality-gates.md` calls `quality.yml` on a
  fixture project per language under `tests/fixtures/projects/`, each with
  a `make check` that writes its report, so every toolchain setup is run on
  every PR here.

## Follow-ups

- Tag `v0.2.0` on `main` once the PR merges, with a release of generated
  notes (no workflow here triggers on tags), so callers can pin it.
- Callers that work around the missing ecosystems (`mac-notch`,
  `money-logs`, `car-mate`, `turbogit`, `agent-eye`) switch to `swift`,
  `kotlin` or `rust` once Phase 3 lands; their own plans list it as a
  follow-up.
- Callers re-pin to `v0.2.0` and add `check.yml`: each repository's own
  `NN-quality-gates.md` plan, not this one.
- Flipping `fail-on` to `high` by default, once every caller has opted in.

## Repository knowledge

Read root `KNOWLEDGE.md` when present before each phase. Follow the global
and repository instructions for recording useful development findings.

## Phases

### Phase 1: the rough cut, Python end to end (needs you: try it)

- [ ] `quality.yml` sets up Python with uv, runs the caller's
  `check-command`, and fails under `coverage-floor` read from an LCOV
  report (expected in `.github/workflows/quality.yml` and
  `scripts/gate/coverage.sh`), pinned by the LCOV fixture cases in
  `tests/run.sh`.
- [ ] The quality sticky comment shows the check result, the coverage total
  and the floor, under its own marker.
- [ ] Check: `make check` passes, and the smoke job runs `quality.yml` on
  `tests/fixtures/projects/python/` green, then red after its floor is set
  to 100.
- [ ] You try it: on a branch of one Python repository, add a `check.yml`
  calling `quality.yml@<this branch's sha>` with `language: python`, and say
  whether the comment tells you what you need.

### Phase 2: every language and format

- [ ] Toolchain setup for `node`, `rust`, `go`, `swift` (macOS runner via
  `runs-on`), `kotlin` and `shell` (expected in `quality.yml` and
  `scripts/gate/setup.sh`), each with a fixture project under
  `tests/fixtures/projects/<language>/` run by the smoke job.
- [ ] The parser reads `gocover`, `jacoco` and `cobertura`, pinned by one
  fixture each; `none` with a reason passes and prints the reason.
- [ ] Check: `make check` passes, and every smoke matrix row is green.

### Phase 3: security for the new languages, and opt-in blocking

- [ ] `security-scan.yml` accepts `rust`, `swift` and `kotlin`, adding each
  language's Semgrep pack, with the slugs confirmed against the Semgrep
  registry on the day (expected in `scripts/gate/packs.sh`), pinned by
  `tests/run.sh` cases.
- [ ] `fail-on: high` fails stages 2 and 3 on a high or critical finding;
  `none` keeps today's behaviour, pinned by cases over sample `osv.json`
  and `semgrep.json` files.
- [ ] Check: `make check` passes, and the smoke job's security call with
  `fail-on: high` is green on this repository.

### Phase 4: documented

- [ ] `README.md` documents `quality.yml`: a caller example, the inputs
  table and the language table from `## Contract`, and how to say coverage
  does not apply.
- [ ] Check: `make check` passes, and `README.md`'s examples pass actionlint
  when pasted into a scratch workflow.

### Phase 5: open the PR

- [ ] The branch pushed and its PR opened against the branch it was cut
  from (or, if one is already open for this branch, its body brought up to
  date) with a Conventional Commit title for the whole change, a body
  naming this plan, what changed and why, and the checks run with their
  result. The PR's URL printed as the evidence. Opened, never merged.
