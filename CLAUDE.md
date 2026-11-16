# eng-metrics-docs: change process

This repo uses feature branches and pull requests (see
`gitultra-wiki/Specs/Feature-Branch-Rollout.md`). These instructions
override default direct-to-master behavior and apply to every session
working in this repo.

## Rules

- **Never push directly to `master`.** A local pre-push hook
  (`.githooks/pre-push`, activate with `git config core.hooksPath
  .githooks`) blocks it, but follow the rule regardless.
- **Never use `git push --no-verify`** (or any other hook bypass) here.
- **All work happens on a feature branch, PR'd into `master`.** See
  `.claude/skills/feature-branch-workflow/SKILL.md`. There is no `dev`
  branch: this repo publishes no image, so there is no `:dev` build to
  test (spec: two tiers).
- **Never merge a PR without the user's explicit go-ahead.** Opening a
  PR is safe; merging is not: a merge into `master` publishes the live docs site (`docs.yml` runs `mkdocs gh-deploy`).

## Why

GitHub branch protection isn't available for private repos on this
org's Free plan, so this CLAUDE.md, the hook and the skill are the
no-cost way to make every change arrive as a reviewed PR.
