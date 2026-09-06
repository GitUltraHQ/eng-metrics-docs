# API Access (Team / Enterprise)

Everything on this site so far covers the free, self-hosted pipeline:
import commit/PR data, then generate a PDF report on demand. **Team and
Enterprise plans** add a REST/JSON API on top of the same database, so
your own internal dashboards (Grafana, an internal tool, whatever you
already use) can pull these metrics directly and on your own schedule,
instead of regenerating and re-reading a PDF.

Like the rest of this pipeline, it's **self-hosted**: it runs in your
own infrastructure, against your own database. Two credentials gate it
-- a license key we issue you, and an API key you generate and control
yourself (see "Auth" below for the difference). No data leaves your
instance to make this work, and there's nothing to sign up for beyond
the plan itself.

## Endpoints

All endpoints return JSON and accept a required `start`/`end` date
range, plus optional scoping (see below).

| Endpoint | What it returns |
|---|---|
| Author activity | Per-author commits, lines added/removed, files changed, churn ratio, and active-days/week (a consistency-of-contribution signal) -- plus PR counts and average commits per PR. |
| Review health | PR/merge counts, cycle time broken into pickup (open → first review) and review (first review → merge) stages with p50/p90 percentiles, per-reviewer load, and a review-load-balance (Gini coefficient) score. |
| Repo trends | Weekly commit/PR volume and churn ratio, per repo. |
| Contributor concentration | Per-repo commit-volume concentration (Gini coefficient + top-contributor share) -- same anonymized shape as review load balance above, applied to commits. A cheap "which repo might warrant a closer look" signal, not a per-person ranking. |
| Recency skew | Per-repo decaying-ownership signal: total historical commits/contributors vs. how many of those contributors are still active in the last 90 days (`window_days` in the response). 0 means everyone historical is still active; approaching 1 means most have gone quiet. Same cheap-trigger family as contributor concentration. |
| Team-to-repo activity mismatch | Per-team, per-repo coverage-gap signal: for each configured team, discovers its presumed-core repo(s) the same way `team` scoping does, then reports the gap between roster size and how many roster members are actually active there (`lookback_days` in the response). **Scoped differently from every other endpoint** -- see "Scoping" below. |
| Contributor departure/reassignment impact | For each real team-membership change you've recorded (a separate, optional team-events log -- see "Team events" below), checks whether that person stopped contributing to a repo they were actually active on before leaving. **The one endpoint that returns a named individual** (their email) rather than an anonymized aggregate -- deliberately, since it's about a real personnel event. **Scoped differently from every other endpoint** -- see "Scoping" below. |
| Cycle time | The pickup/review/total percentile breakdown as its own endpoint, for dashboards that only need that slice. |
| Deployment frequency | Weekly deployment counts per repo (every git tag counts by default; scope down to a real deploy-tagging convention with a tag-pattern filter). |
| Lead time for changes | p50/p90/avg hours from a commit to its nearest later tag. |
| Change failure rate | Percentage of deployments (tags) linked to a Jira incident via that incident's Fix Version field, exact-matched to the tag name. Requires the optional [issue-processor](https://github.com/GitUltraHQ/issue-processor) Jira integration -- returns 0/`null` with `jira_configured_repo_count: 0` if it isn't set up, not a misleading 0%. |
| Mean time to restore (MTTR) | p50/p90/avg hours from a linked incident's creation to its resolution. Same issue-processor dependency as change failure rate above; still-open incidents are excluded from the average but reported separately as `open_excluded`. |
| AI usage | Per-author and org-wide weekly trend of commits with an AI-tool `Co-authored-by:` trailer (the same convention GitHub itself reads to show a tool's avatar on a commit) -- a lower bound on AI-assisted work, not a precise measurement. |
| Investment allocation | Ticket/story-point distribution, allocation trend, cycle time, and logged time, per Jira-ticket category. Requires issue-processor's separate allocation-tracking flow (independent of the CFR/MTTR one above). **Not scoped by `repo`/`org`/`team` at all** -- see "Scoping" below. |
| Author distribution trend | Weekly p50/p90 (never a per-author number) of commit count, churn ratio, and active days -- the anonymized, ranking-safe alternative to a per-author leaderboard. |
| Reviewer distribution trend | Weekly p50/p90 of reviews-per-reviewer, same anonymization reasoning as above. |
| Commits-after-open distribution trend | Weekly p50/p90 of avg commits pushed after a PR was opened, per PR. |

Deployment frequency, lead time, change failure rate, and MTTR are the
four canonical DORA metrics -- all four are available here.

## Scoping

Every endpoint accepts:

- `repo` (repeatable) and/or `org` -- limit to specific repos, same as
  the free CLI's `--repo`/`--org`.
- `team` -- limit to a team's repos and (where the data has a real git
  author identity, like commits and AI usage) filter to that team's own
  members, not just repos they happen to touch. Requires a team roster
  to be configured. PR/review data and deployment frequency can't be
  attributed to a team the same way (a PR author is a vendor-API
  username, not a git email; a tag isn't authored by anyone), so those
  stay repo-scoped.

`repo`/`org` and `team` are mutually exclusive.

**Investment allocation is one exception** -- it accepts none of
`repo`/`org`/`team`. Jira tickets have no repo relationship (allocation
tracking is purely Jira-side data), so there's nothing to scope by; it
always returns one combined view across every tracked Jira project.

**Team-to-repo activity mismatch is another exception**, in the
opposite direction -- it accepts no `repo`/`org` at all, and always
scans your whole org to discover each configured team's own presumed-
core repos. `team` is still accepted, but only to narrow the *returned
rows* to one team afterward, not to change what gets scanned.

**Contributor departure/reassignment impact accepts no scoping params
at all** -- it evaluates each recorded team-membership change against
that specific person's own commit history directly.

## Team events

A separate, optional log from the team roster above: `email,team,
action,date` rows (`action` is `added` or `removed`), recording real
team-membership changes over time -- your roster (`team` scoping) is a
current-snapshot, this is history. Only used by the contributor
departure/reassignment endpoint. Configuring one doesn't require the
other.

## Example

```
curl "https://your-instance/v1/author-activity?start=2026-01-01&end=2026-04-01&team=Platform" \
    -H "Authorization: Bearer <your key>"
```

```json
{
  "commit_activity": [
    {
      "author_email": "alice@example.com",
      "author_name": "Alice Anderson",
      "commits": 47,
      "added": 1704,
      "removed": 340,
      "churn_ratio": 0.17,
      "active_days_per_week": 1.6
    }
  ],
  "meta": {
    "start": "2026-01-01T00:00:00Z",
    "end": "2026-04-01T00:00:00Z",
    "team_scope": {"team": "Platform", "repo_count": 20}
  }
}
```

## Auth

Two separate credentials are involved, easy to conflate since both are
just an env var and a header -- don't mix them up:

- **License key** (`GITULTRA_LICENSE_KEY`) -- gates the deployment
  itself. **We issue this to you** after you get plan access (see
  "Getting access" below); it's not something you generate. The service
  checks it once at startup and refuses to boot without a valid one --
  see [eng-api](https://github.com/GitUltraHQ/eng-api)'s README for
  details.
- **API key** (`API_KEY`/`ENG_API_KEY`, sent as `Authorization: Bearer
  <key>`) -- authenticates *your own dashboards'* requests to your
  already-running instance. **You generate this yourself** (see below);
  we never see or store it.

Every request needs `Authorization: Bearer <key>` using the API key
(not the license key). Once you have plan access, there's no separate
account/key request step for the API key -- only the license key comes
from us.

### Generating an API key

Any sufficiently random string works -- a UUID, a password manager's
generator, whatever you already reach for. If you don't have a
preferred method:

```
openssl rand -hex 32
```

or, without `openssl`:

```
python3 -c "import secrets; print(secrets.token_hex(32))"
```

Set the result as `ENG_API_KEY` in your `.env` (if you're running the
`eng-metrics-suite-pro` compose bundle) or as the `API_KEY` environment
variable directly (if running the container standalone) -- then send it
back as `Authorization: Bearer <that value>` on every request. Treat it
like any other credential: don't commit it, and rotate it (just
generate a new one and restart the container) if you suspect it's
leaked.

## Getting access

Available on **Team** and **Enterprise** plans -- see
[gitultra.com](https://gitultra.com) for plan details, or apply for
beta access there if you're interested. Once you have access, we issue
you a **license key** (`GITULTRA_LICENSE_KEY`, valid 90 days, renewed on
request) -- set that alongside the API key you generate yourself (see
"Auth" above) and you're running.

Want to query these same metrics conversationally, from Claude Desktop
or Claude Code, instead of calling the API yourself? See
[AI Agent Access](mcp-access.md).
