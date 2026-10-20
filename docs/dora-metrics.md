# DORA Metrics

GitUltra measures the four DORA metrics. All four are built on one
signal: **a git tag is a deployment.**

| Metric | What it measures | Needs |
| --- | --- | --- |
| Deployment frequency | How many deployments (tags) per week | git history |
| Lead time for changes | Hours from a commit to the deployment that shipped it | git history |
| Change failure rate | Share of deployments linked to an incident | git history + [Jira](jira-integration.md#change-failure-rate-mttr) |
| Mean time to restore (MTTR) | Hours from an incident being opened to being resolved | git history + [Jira](jira-integration.md#change-failure-rate-mttr) |

They appear in the [org report](generating-reports.md) (Deployment
Metrics, plus change failure rate and MTTR once Jira is connected), the
[Executive Report](executive-report.md) and [QBR Report](qbr-report.md)
KPIs, the [Onboarding Report](onboarding-report.md)'s A2 and A4 checks,
the [API](api-access.md) and [AI agent access](mcp-access.md).

## Deployments are git tags

On every sync, git-processor reads every tag in each repo. A tag that's
deleted or moved is removed or updated on the next sync, so it doesn't
leave a deployment behind.

Each tag is dated by:

- **An annotated tag** (`git tag -a`): the date it was tagged.
- **A lightweight tag** (`git tag` without `-a`): the date of the commit
  it points at.

So tag at deploy time, and prefer annotated tags. A lightweight tag
added today to a commit from last month counts as a deployment last
month.

## Choosing which tags count

By default **every tag counts as a deployment**. If your repos also have
tags that aren't deployments (library versions, release candidates,
markers), set a tag pattern so only real deployments count.

The pattern is a regular expression matched against the tag name. It's
case-sensitive, and one pattern applies to every repo. Some examples:

| Pattern | Counts |
| --- | --- |
| `^v[0-9]` | `v1.4.0`, `v2.0.0-hotfix`, not `release-candidate` |
| `^v[0-9]+\.[0-9]+\.[0-9]+$` | `v1.4.0`, not `v1.4.0-rc1` |
| `^release-` | `release-2026-10-19` |
| `^prod-` | `prod-42` |

Set it once in your `.env`, then run `docker compose up -d`:

```
ENG_API_TAG_PATTERN=^v[0-9]
```

The name is the same in the free suite and on paid plans. Every report
uses it, and so does eng-api on paid plans, so all of them count the
same deployments. Leave it blank to count every tag.

To try a different pattern for one run without changing `.env`, pass
it to the report:

```
docker compose run --rm eng-reports report.py --tag-pattern '^release-' --output /out/report.pdf
```

The Onboarding Report's A2 check lists repositories with commits but no
deployments in your baseline quarter, and says whether a repository has
no tags at all or only tags that don't match your pattern.

## How each metric uses deployments

**Deployment frequency** counts matching tags per week, by tag date.

**Lead time for changes** gives each commit the time until the next
matching tag in the same repository, by date. It doesn't follow
branches: a commit counts toward the first deployment after it, wherever
it was merged from. That fits short-lived branches merged to the main
branch; on long-lived branches, lead time comes out shorter than it
really was. A period's lead time covers the deployments made in that
period.

**Change failure rate and MTTR** link a Jira incident to a deployment
when the incident's **Fix Version** exactly matches the tag name, in a
repository mapped to that Jira project. There's no fuzzy or time-window
matching, so your Fix Versions need to use your tag names. Setup is on
the [Jira Integration](jira-integration.md#change-failure-rate-mttr)
page. Without it, these two metrics say "not configured" rather than
showing a misleading 0%.

## Changing the pattern later

A new pattern applies to every period a report looks at, past ones
included. Baselines recorded before the change still reflect the old
pattern, and GitUltra doesn't notice the change on its own, so
re-baseline the four tag-based KPIs yourself. If you're still
onboarding, finalizing covers it (see the
[Onboarding Report](onboarding-report.md#finalizing)). Otherwise, run
this once after the change:

```
docker compose exec eng-api python manage.py rebaseline --kpi "Deployment Frequency" --kpi "Lead Time for Changes" --kpi "Change Failure Rate" --kpi MTTR --reason "Changed the deployment tag pattern"
```
