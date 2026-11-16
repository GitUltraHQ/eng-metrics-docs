---
name: feature-branch-workflow
description: The required procedure for landing any change in this repo -- feature branch, PR into master. Use whenever asked to make a code/doc/workflow change in this repo, or when about to run `git push`.
---

# Feature Branch Workflow (eng-metrics-docs)

See `gitultra-wiki/Specs/Feature-Branch-Rollout.md` for the rationale.
This repo publishes no image, so there is no `dev` branch: every change
is a feature branch and a PR into `master`.

**Never push directly to `master`.** A local git hook
(`.githooks/pre-push`) blocks this, but follow the rule regardless of
whether the hook is active -- see `CLAUDE.md`.

## Landing a change

1. Make sure `master` is current: `git fetch origin && git checkout
   master && git pull`.
2. Branch off `master`: `git checkout -b feature/<short-description>`.
3. Commit as normal.
4. Push the feature branch: `git push -u origin feature/<short-description>`.
5. Open a PR into `master`: `gh pr create --base master --title "..."
   --body "..."`.
6. Report the PR link to the user and **stop**. Do not merge without
   the user's explicit go-ahead: a merge into `master` publishes the live docs site (`docs.yml` runs `mkdocs gh-deploy`).

## What NOT to do

- Don't push directly to `master`, even for a typo or a one-line fix.
- Don't use `git push --no-verify` to route around the pre-push hook.
- Don't merge a PR you opened without the user confirming first, even
  if CI is green.
