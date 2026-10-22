# the README opens with a banner that says what the shared gate does, in the viewer's theme

## The ask

Every repository should have a proper minimal README with a banner (a light
and a dark SVG, per the `github-banner` skill) or a UI screenshot near the
top (brief of 22/10/2026). This README is already short and to the point
(`README.md`: what the gate does, a stage table, how to call it,
conventions); it has no image.

Built so far: `docs/built/BUILT-2026-10-22-1828.md`.

Not applicable here: a `site/` one-pager. This repository is shared CI with
one audience, the repositories that call it; its README on GitHub is where
that audience already reads. A site would repeat it.

## Alternatives

### Decision 1: banner or screenshot (two-way door)

The only thing a person sees of this repository's work is the sticky
comment on a pull request, so a screenshot of it is a real option.

Searched 2026-10-22: "GitHub README picture prefers-color-scheme dark light
image"; GitHub's docs on writing on GitHub and the GitHub Blog post on
images that follow dark and light mode.

| Criterion | Weight | Baseline: no image | Banner, two SVGs | Screenshot of the PR comment |
| --- | ---: | ---: | ---: | ---: |
| Performance efficiency | 40 | 5 | 4 | 3 |
| Fit to the ask | 35 | 1 | 5 | 4 |
| Maintainability of the result | 25 | 5 | 4 | 2 |
| **Weighted total** | **100** | **3.60** | **4.35** | **3.10** |

- Banner efficiency 4: two small text SVGs; screenshot 3: a PNG per theme,
  larger than text SVG.
- Screenshot maintainability 2: goes stale whenever the comment's rows
  change, which `plans/02-language-gate.md` is about to do; the banner
  shows the function, not the layout.
- `<picture>` with `prefers-color-scheme` follows the viewer's GitHub theme
  (GitHub Blog); one SVG with a `@media` query follows the OS instead.

**Chosen: banner, two SVGs.** Revisit if the README grows a "what it looks
like" section, where the comment screenshot belongs.

## The decision

A banner pair, `assets/banner-light.svg` and `assets/banner-dark.svg`, made
with the `github-banner` skill (its concept log read first, its build
script's lint clean), shown at the top of `README.md` with `<picture>`. The
rest of the README keeps its voice and length.

No new dependency in the repository; the skill's build and preview scripts
run outside it.

Standards: `docs/standards.md` (`make check` once
`plans/01-quality-gates.md` lands; until then the pre-commit hooks).

Security: no new input, data, secret, dependency or permission; the SVGs
hold no script, link or external resource (the skill's build refuses them).

## Where it lands

| Path | New or edit | Phase | What changes | Firm? |
| --- | --- | --- | --- | --- |
| `assets/banner-light.svg` | new | 1 | light banner | fixed (skill rule) |
| `assets/banner-dark.svg` | new | 1 | dark banner | fixed (skill rule) |
| `README.md` | edit | 1 | `<picture>` block at the top | reuse |

*Reused, not rewritten:*

- The `github-banner` skill's `scripts/build.py` and `scripts/preview.sh`
  for the build, lint and look; no banner tooling lands here.

## Declined

- No image: misses the ask (decision 1).
- A screenshot of the PR comment: goes stale with every change to the
  comment (decision 1).

## What the tests hold

The skill's build exits 0 (no overlap, no animation, no script or external
resource, title and description present); contrast warnings read and fixed;
both themes looked at in the preview and on GitHub.

## Repository knowledge

Read root `KNOWLEDGE.md` when present before each phase. Follow the global
and repository instructions for recording useful development findings.

## Phases

### Phase 1: the rough cut, the banner on the README (needs you: try it)

- [ ] A fresh concept of what the gate does, checked against the skill's
  `concepts-used.md`, built into `assets/banner-light.svg` and
  `assets/banner-dark.svg` with the skill's build script exiting 0.
- [ ] `README.md` opens with the `<picture>` block and alt text naming what
  the gate does.
- [ ] Check: `make check` (or `pre-commit run --all-files` before
  `plans/01-quality-gates.md` lands) passes, and the skill's preview shows
  both themes with nothing overlapping.
- [ ] You try it: open the branch's README on GitHub in light and dark
  theme and say whether the joke lands.

### Phase 2: logged

- [ ] The concept is added to the skill's `concepts-used.md` so no other
  banner repeats it, and anything you asked for in Phase 1 is applied to
  `assets/banner-light.svg` and `assets/banner-dark.svg`.
- [ ] Check: the skill's build still exits 0.

### Phase 3: open the PR

- [ ] The branch pushed and its PR opened against the branch it was cut
  from (or, if one is already open for this branch, its body brought up to
  date) with a Conventional Commit title for the whole change, a body
  naming this plan, what changed and why, and the checks run with their
  result. The PR's URL printed as the evidence. Opened, never merged.
