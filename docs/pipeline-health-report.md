# Pipeline Health Report

!!! note "Free on every plan"
    The Pipeline Health Report needs no license key. A failed import
    you can't see is a problem on any plan.

When an import stops working, nothing else tells you: a repository that
fails three times in a row is marked failed and never retried on its
own, and every other report quietly leaves it out. The Pipeline Health
Report shows what's broken in the import pipeline, why, and the command
that fixes it.

```
docker compose run --rm eng-reports pipeline_health_report.py --output /out/pipeline-health.pdf
```

Run it whenever you like. Nothing is stored between runs, so a fix shows
up as soon as the data changes: fix something, run it again, and check
that it's gone. To get it on a schedule, run the same command from cron
(see [Scheduling Reports](scheduling.md)).

On Team and Enterprise plans, an Admin without access to the server can
also [ask an AI agent for it](#getting-it-by-email-team-enterprise).

## What's in it

**Last successful sync.** For git history, pull requests and both Jira
flows: how many items are tracked and when any of them last synced.
Every successful sync updates this, even when nothing new arrived, so if
one stops moving while the others keep going, that processor has
probably stopped.

Then one section per check. A check that finds nothing shows as a single
line with a check mark; one that finds something says what it sees, why
it matters and what to do, with the exact command.

| Section | What it looks for |
| --- | --- |
| Providers paused on rejected credentials | pr-processor stopped every pull request import for a provider because its token was rejected. Fix the token in `.env` and run `docker compose up -d pr-processor`; the waiting repos pick up again on their own. |
| Failed imports | Repositories and Jira projects that failed and won't be retried on their own, with the error and the retry command. Also lists **pull request access problems** (see below). |
| Possible lost access | Repositories returning 404 where nothing else under the same owner has synced recently (see below). |
| Probably moved or deleted | Repositories returning 404 while other repositories under the same owner still sync, with a list ready to paste into `mark_inactive.py`. |
| Pull requests with failed polls | Pull requests whose last fetch failed, so they're stored with gaps. "To retry" (rate limits, server errors, timeouts) are fetched again on every sync; "unavailable" (404 on that pull request) only when the pull request changes. |
| No recent activity | Repositories with no commits or pull request updates for 7 days (warning) or 14 days (stale). |
| Developers who may need merging | One GitHub or Bitbucket Cloud account committing under more than one email, with the lines to add to your [identity file](author-identity.md#one-file-for-your-whole-org-recommended). Shows real email addresses, since the fix needs them. |
| Marked inactive | Repositories you've marked inactive: who, when and why. |

Each repository shows up in one place only: one that's probably moved
isn't also listed under failed imports or no recent activity.

## Getting it by email (Team / Enterprise)

The report is most useful to the person who looks after the install,
and that person may not have shell access to the server. On Team and
Enterprise plans, an Admin can ask their AI agent for it, and the same
PDF arrives in their inbox:

> Send me the pipeline health report.

The agent uses the `email_admin_pipeline_health_report` tool (see
[AI Agent Access](mcp-access.md#org-role-tools-require-email-verification)).
What you need:

- **The Admin role.** Your email has a row with role `admin` in
  `org_roles.csv` (see [Generating Reports](generating-reports.md#org_rolescsv-org-level-roles-admin-executive-director)).
  Executives and Directors are refused. You verify your email once per
  conversation with a 6-digit code, the same as the other Admin tools.
- **SMTP settings** (`SMTP_HOST` and the rest) in `.env`, so `eng-api`
  can send email.

The report only ever goes to your own verified address, never to an
address in the request. It's the same report the command above writes,
from the same checks and with the same fix commands.

The agent gets back only each section's name and how many items it
found, never the items themselves. So it can tell you "all clear" or
"3 failed imports, details in your email", but not which repositories
they are. That keeps the developer email addresses in the "Developers
who may need merging" section in your inbox and out of the chat. If a
section couldn't be checked, the agent says so instead of calling it
clear.

The same thing is available directly from the API as
`POST /v1/admin/email-pipeline-health-report` (see
[API Access](api-access.md#org-level-roles-admin-executive-director)),
with an optional `max_rows` (default 50) for how many rows each section
lists. Every section still says how many it left off.

## When a repository returns 404

A 404 from the provider can mean the repository is gone, or that your
token can no longer see it: GitHub, for example, answers 404 for both.
The report sorts each one by what else it can see:

- **Pull request access problem.** git-processor can still read the
  repository, so it isn't gone; only the pull request API stopped seeing
  it. Check the scope of pr-processor's token for that repository.
- **Probably moved or deleted.** Another repository under the same
  owner had a successful sync in the last 6 hours, so the token still
  works there and the 404 is about this repository.
- **Possible lost access.** Nothing else under the same owner synced in
  the last 6 hours, or it's the owner's only repository. That's what it
  looks like when a token loses access to a whole organization
  (someone left, a fine-grained token's repository list changed, an app
  was uninstalled), so check access before marking anything inactive.

The owner is the repository's path minus its name: the organization on
GitHub, the workspace on Bitbucket Cloud, the project on Bitbucket
Server and Azure DevOps, and the group the repository is directly in on
GitLab. The 6 hours is three of pr-processor's default 2-hour resync
cycles; if you've set `--resync-after-minutes` above 6 hours, expect
more repositories under "Possible lost access".

One case the report can't tell apart: a single repository removed from a
fine-grained token while the rest of its organization still syncs looks
the same as a moved repository. `mark_inactive.py` previews before it
changes anything, so check the list before you confirm.

## Marking repositories inactive

Some repositories are quiet or gone on purpose: archived, or moved to
cheaper storage. Mark them inactive and the report stops raising alarms
about them. Unlike [removing a repository](discovering-repos.md#untracking-a-repo),
nothing is deleted: the repository keeps syncing and its history still
counts in every report.

```
docker compose run --rm --entrypoint python3 git-processor mark_inactive.py github.com/acme/old-service \
    --actor "you@company.com" --reason "archived" --yes
```

To mark the report's "Probably moved or deleted" list in one go, save it
as `repos.txt` and pipe it in (one repository per line; blank lines and
lines starting with `#` are ignored):

```
docker compose run --rm -T --entrypoint python3 git-processor mark_inactive.py --file - \
    --actor "you@company.com" --reason "moved or archived" --yes < repos.txt
```

Leave off `--yes` to see what would change first. If any repository in
the list isn't tracked, nothing is changed. `--actor` is required.
Repositories that are already inactive are skipped.

To bring one back:

```
docker compose run --rm --entrypoint python3 git-processor mark_inactive.py github.com/acme/old-service \
    --undo --actor "you@company.com" --yes
```

The record of who marked it, when and why is kept.
