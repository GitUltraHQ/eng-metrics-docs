# Onboarding Report

!!! note "This report comes with your first 90 days"
    The Onboarding Report is included with every paid plan for the
    first 90 days after you sign up. Your license key
    (`GITULTRA_LICENSE_KEY`) needs to cover `"onboarding-report"`; the
    report header shows how many days are left. Contact
    support@gitultra.com if you need more time.

The Onboarding Report is the first report to look at, and it's for you,
not your whole company yet. It shows where GitUltra may be seeing your
organization incorrectly, how your KPI targets compare with what your
data says, and what to fix before your starting point is locked in.

```
docker compose run --rm eng-reports onboarding_report.py --output /out/onboarding.pdf
```

Run it as often as you like. Nothing is stored between runs, so a fix
shows up as soon as the data changes: fix something, run it again, and
check that the finding is gone.

You can run it right after installing. While the first import is still
running it shows import progress and your configuration; the rest fills
in as data arrives.

## What's in it

**Header.** Days left in onboarding and how many repositories you're
onboarding with.

**Import progress.** For git history, pull requests, and both Jira
flows: how many repositories or projects have been imported at least
once, and how many failed.

**Configuration.** What the report can see: Jira projects tracked, your
team map, KPI targets, deploy tag pattern, fiscal year start, AI
co-author patterns and bot accounts. Settings that live in the
importers (access tokens, the identity file, Jira issue types) don't
appear here.

**A. Is GitUltra seeing your org correctly?** Each check either passes
with a check mark or explains what it found, why it matters and what to
do:

| Check | What it looks for |
| --- | --- |
| A1 | One developer committing under several emails (GitHub and Bitbucket Cloud accounts). Gives you the lines to add to your [identity file](author-identity.md). |
| A6 | Repositories or Jira projects that failed to import, with the error and the command to retry. |
| A3 | Repositories with commits but no pull requests at all. |
| A2 | Repositories with commits but no deployment tags in your baseline quarter. |
| A4 | Deploying repositories with no Jira project mapped, or with fewer than half their commits naming a ticket. |
| A5 | People with 3 or more commits who aren't in your team map. |
| A7 | Pull requests approved with nothing written: no comment, inline comment or review text from anyone but the author. Reviews written only by AI tools are counted separately. |

A1, A6 and A3 appear from the start. The others need your baseline
quarter and appear once the first import finishes. A1 and A5 show real
email addresses, since fixing them means adding those addresses to your
identity file or team map.

**B. How do your expectations compare with reality?** Each KPI over your
baseline quarter: your target, the estimate you recorded (the `expected`
column of your targets file), the actual value, and the normal weekly
range (the middle 80% of weeks, the same range the QBR uses). Each
target is described as already hit in a normal week, easier than most
of your weeks, outside your normal range, or beyond your best weeks.

**C. Before you finalize.** How many findings remain, the quarter that
will be locked in, and the command.

## Your baseline quarter

Your baseline is the last full fiscal quarter before your first import
finished. It's history, so it's complete on day one, and it never moves
however long onboarding takes. Every run of this report recalculates
that same quarter from your current data.

## Finalizing

When your data looks right, finalize:

```
docker compose exec eng-api python manage.py rebaseline --finalize
```

This re-records your baseline over the same quarter from your corrected
data, and every [QBR](qbr-report.md) measures progress from it. Without
it, QBRs measure from the baseline recorded automatically from your
first import, so cleaning up data would look like improvement that
didn't happen.

- You can finalize with findings still open. Some never clear, and
  that's fine (a repository that never deploys, for example).
- You can finalize again after more fixes; the latest one counts.
- The command keeps working after the 90 days, when the report itself
  stops.
- Until you finalize, the QBR and the Executive Report each open with a
  one-line reminder.

## Settings

The report reads the same settings as the rest of the suite. In
eng-metrics-suite-pro's `.env`:

| Setting | Used for |
| --- | --- |
| `ENG_API_TEAM_MAP_PATH` | A5 and the Configuration section |
| `ENG_API_TAG_PATTERN` | A2, A4 and the deployment-based KPIs |
| `ENG_API_AI_COAUTHORS_PATH` | AI co-author patterns, including AI reviewer accounts for A7 |
| `BOT_PATTERNS_PATH` | Extra bot accounts for A7, one regex per line, added to the bundled list |
| `KPI_TARGETS_PATH`, `FISCAL_YEAR_START_MONTH` | Section B and the baseline quarter |

### AI reviewers and bots in A7

A7 decides who counts as a reviewer when the report runs, so changing
these files takes effect on the next run:

- **AI review tools** (GitHub Copilot, CodeRabbit, Gemini Code Assist
  and others) are listed as `reviewer` rows in the AI co-author file.
  Their comments make a pull request "reviewed by AI only".
- **Bots and CI accounts** (GitHub Actions, Dependabot, Renovate,
  Jenkins and others) don't count as reviewers, and pull requests they
  open are left out. Add your own with `BOT_PATTERNS_PATH`, for example
  a file containing `^acme-deployer$`.

### Pull requests from before this release

GitUltra started recording pull request comments in pr-processor
v1.5.0. Pull requests fetched before that aren't judged by A7; the card
says how many there are. To include them, re-fetch every pull request
once (it takes a while and uses API rate limit on large organizations):

```
docker compose run --rm --entrypoint python3 pr-processor reingest_repo_prs.py --all --full --yes
```
