# Generating Reports

```
docker compose run --rm eng-reports report.py --output /out/report.pdf
```

If your host user isn't UID 1000 (check with `id -u`), add `--user
"$(id -u):$(id -g)"` so the output file comes out owned by you instead of
UID 1000:

```
docker compose run --rm --user "$(id -u):$(id -g)" eng-reports report.py --output /out/report.pdf
```

The PDF lands in `./reports/report.pdf` on the host (that directory is
bind-mounted into the container). If you've configured the optional
[Jira Integration](jira-integration.md), this PDF also gains a change
failure rate/MTTR section -- otherwise that section reports "not
configured" rather than a misleading 0%.

## Useful flags

- `--period last-week|last-month|last-quarter` or `--start YYYY-MM-DD --end YYYY-MM-DD`
  (default: last 30 days)
- `--repo identity_key` (repeatable) and/or `--org github.com/owner` to
  scope the report instead of covering every repo you've imported
- `--team-map /var/lib/eng-metrics-suite/teams.csv` to roll up commit
  activity by team instead of by individual author

## Team rollups

Put a CSV file with an `email,team` header at
`/var/lib/eng-metrics-suite/teams.csv` on the host — CSV rather than a
config-file format like YAML/JSON, so whoever maintains the roster (often
not an engineer) can keep it in a spreadsheet and export/save as CSV:

```csv
email,team
alice@example.com,Platform
bob@example.com,Platform
carol@example.com,Product
```

Anyone not listed shows up under "Unmapped" in the report rather than
being silently dropped.

If the same person shows up as multiple rows in the author table, or
lands in "Unmapped" despite being in your team-map CSV, see
[Author Identity Consistency](author-identity.md).

### Optional columns: manager reporting and pseudonymization

The same CSV accepts three additional optional columns, on top of the
required `email,team`:

```csv
email,team,role,receives_report,reportable
alice@example.com,Platform,manager,true,true
bob@example.com,Platform,contributor,false,true
carol@example.com,Platform,contributor,false,false
```

- **`role`** (`contributor` or `manager`, default `contributor`) —
  marks who manages a team. A person can appear on more than one row
  to manage more than one team.
- **`receives_report`** (`true`/`false`, default `false`) — whether
  this person is automatically emailed their team's report; see
  [Emailing team reports to managers](scheduling.md#emailing-team-reports-to-managers)
  below. Separate from `role` since a manager might exist in your
  roster without wanting the automated email.
- **`reportable`** (`true`/`false`, default `true`) — set to `false`
  to pseudonymize a specific contributor. Instead of their real
  name/email, reports (and any future MCP/API access to per-person
  data) show a stable, auto-assigned alias like "Contributor A" —
  the same alias every time, for that person, across every report run.
  Team-level totals are unaffected; only the individual breakout is
  aliased.

Any row can mix and match — a legacy 2-column CSV with just
`email,team` keeps working exactly as before; these three columns are
purely additive.

**What pseudonymization does and doesn't protect against**: this
reduces identifiability in the report itself — a manager reading the
PDF sees "Contributor A," not a name. It does not achieve legal
anonymity. A consistent pseudonym is generally considered reversible/
re-identifiable under GDPR and works-council standards, especially
given the behavioral cues (PTO patterns, repo ownership, commit
timing) a manager already has independent of this feature. If you
need this for works-council or GDPR compliance reasons, treat this as
one control among several, not a complete answer — your own
compliance posture (works-council agreement, opt-out process, legal
review) is what actually matters, this flag just gives you the lever
to configure it.

**PR/review coverage (GitHub, Bitbucket Cloud)**: pseudonymization
above always covers your commit history. For GitHub and Bitbucket
Cloud, it also extends to the PR-author and reviewer breakdowns (the
"By Author (pull requests)" and "By Reviewer" tables) -- but only for
a contributor who has authored a commit inside at least one pull
request your install has imported. No new token scope is needed for
either provider -- your existing import credentials already have
everything this requires.

For **GitHub**, this includes historical PRs, not just ones imported
after you set `reportable=false` -- a one-time backfill resolves
existing data the next time that repo syncs.

For **Bitbucket Cloud**, the boundary is narrower: only PRs and reviews
imported *after* you upgrade to a version with this feature get
covered. Bitbucket has no equivalent of GitHub's per-person account
lookup, so there's no cheap way to backfill identity on a PR/review
your install already imported before upgrading -- those rows keep
showing the bare Bitbucket display name, unaliased, permanently. This
is a disclosed limitation of the platform, not something a future
release fixes by trying harder.

For either provider, if a contributor has never had a commit go
through a tracked pull request (e.g. a repo that's mostly
direct-to-master, or a reviewer who's never opened a PR of their own),
their vendor username still shows unaliased in these two tables
specifically -- a disclosed gap, not a bug, and independent of whether
their commit-level activity elsewhere in the same report is aliased.
GitLab, Bitbucket Server/Data Center, and Azure DevOps PR/review
coverage is not yet supported.

The same GitHub/Bitbucket Cloud coverage also improves *display* for
every non-pseudonymized contributor: the "By Author (pull requests)"
and "By Reviewer" tables show that person's real git author name (e.g.
"Josh Sooter") instead of their bare vendor username, whenever it's
known -- independent of whether you've configured a `team_map` at all.
Falls back to the bare username under the same coverage boundary as
above (no commit through a tracked pull request yet, or -- for
Bitbucket Cloud specifically -- a PR/review imported before you
upgraded).

### `org_roles.csv`: org-level roles (Admin / Executive / Director)

A separate, optional CSV, independent of the team roster above --
`email,role`, `role` one of `admin`/`executive`/`director`:

```csv
email,role
admin@example.com,admin
cfo@example.com,executive
vp-eng@example.com,director
```

One row per email (unlike the team roster, none of these three roles
are team-scoped, so there's nothing to enumerate per team). A person
can have both a team-roster row (manager of specific teams) and an
`org_roles.csv` row at the same time -- both grants apply. See [API
Access](api-access.md#org-level-roles-admin-executive-director) and
[AI Agent Access](mcp-access.md#org-role-tools-require-email-verification)
for what each role can reach and how to verify as one; set
`ORG_ROLES_PATH` (`eng-api`) to this file's path to turn the feature
on -- leaving it unset changes nothing about existing behavior.

### rewrite-ratio pseudonymization

[rewrite-ratio](https://github.com/GitUltraHQ/rewrite-ratio) (a
separate paid-tier CLI tool) supports the same pseudonymization as
above via two flags:

```
python3 rewrite_ratio.py --repo <path> --author <email> \
    --start 2026-01-01 --end 2026-04-01 \
    --pseudonymize --dsn postgres://user:pass@host/db
```

- `--dsn` (or a `DATABASE_URL` environment variable) -- a read-only
  connection to your shared metrics database, used only to look up
  existing aliases.
- `--pseudonymize` -- requires a license covering **both**
  `rewrite-ratio` and `team-reporting`.

Unlike the report/API/MCP surfaces above, **rewrite-ratio never
creates a new alias** -- it only reflects ones eng-reports or eng-api
already assigned. This keeps the alias table's contents controlled
entirely by the report/API pipeline, not by whichever tool happens to
run first. If `--pseudonymize` is set and a scoped contributor doesn't
have an alias yet (nobody's run a report covering them), that person's
real email is shown as-is, with a logged warning -- run a report or
query covering them first if you need them aliased everywhere.

Same "reduces identifiability, not legal anonymity" caveat as above
applies here too.

## Investment allocation report

A second, independent script in the same `eng-reports` image: shows
how work breaks down by category (feature/bug/tech-debt/etc.), sourced
from Jira ticket data via the optional [Jira Integration](jira-integration.md)'s
allocation-tracking flow (separate from the change-failure-rate/MTTR
one above -- configure either or both).

```
docker compose run --rm eng-reports allocation_report.py --output /out/allocation.pdf
```

Same `--period`/`--start`/`--end`/`--output` flags as `report.py`
above, but **no `--repo`/`--org`/`--team-map` scoping** -- it always
covers every tracked Jira project as one combined report (Jira tickets
have no repo relationship the way commits/PRs do, so there's nothing
to scope by). If the allocation flow isn't configured at all, the PDF
renders a single "not configured" page rather than misleading empty
tables.

## Planning quality signals report

A third, independent script: how well planning tracked actual
execution, joining the same allocation-tracking ticket data above
against a **new** link from commit to ticket -- if a commit's message
contains a ticket key (e.g. `ENG-123`), it's automatically picked up
(a best-effort match, not a Jira-verified one; nothing else needs
configuring).

```
docker compose run --rm eng-reports planning_report.py --output /out/planning.pdf
```

Same flags as `allocation_report.py` above, plus `--max-tickets N`
(default 100) capping the per-ticket table. Two sections: **ticket-to-
first-commit lead time** (with a linkage-coverage number shown
prominently -- if your commits don't reference ticket keys, this
section won't have much to say, and the report tells you that plainly
rather than showing an empty chart) and **ticket scope mismatch**
(story points vs. actual change size, ranked by how unusual each
ticket is relative to your own team's typical ratio).

## Executive Report (Enterprise plan)

!!! note "This one report requires the **Enterprise** plan"
    Every other report on this page is free with the self-hosted
    pipeline. This is the exception -- see
    [gitultra.com](https://gitultra.com) for plan details, or apply for
    beta access there. Once you have access we issue you a license key
    (`GITULTRA_LICENSE_KEY`, same credential [API Access](api-access.md)
    uses) covering `"executive-report"`; without it the script exits
    with an actionable error rather than a partial/broken PDF.

A fourth, independent script: rather than another table of metrics,
this one is a *judgment layer* on top of the reports above -- a
configurable target per KPI, an on-track/at-risk/off-track status, a
weekly trend, and a fixed suggestion for what to look at next when
something's off track. Built entirely from the same data the other
three reports already compute; nothing new to configure beyond an
optional target-override CSV.

```
docker compose run --rm eng-reports executive_report.py --output /out/executive.pdf
```

Same `--period`/`--start`/`--end`/`--output` flags as the reports
above, plus `--kpi-targets` (a CSV of `kpi,operator,target` rows --
defaults to 8 built-in KPIs covering deployment frequency, lead time,
change failure rate, MTTR, bug effort share, feature delivery,
high-priority bug share, and overdue ratio). Four more KPIs (cycle
time, ticket-to-first-commit lead time, ticket scope mismatch, and
planning signature) are available but off by default -- add a row to
your own copy of the CSV to turn one on. Whichever KPIs aren't
configurable on your instance (e.g. change failure rate without the
Jira integration configured) show as "not available," never a
fabricated number.

**No `--repo`/`--org`/`--team-map` scoping** -- like Investment
Allocation and Planning Quality Signals above, this is one org-wide
view, not a drill-down tool. Also available via [API Access](api-access.md)
(`GET /v1/executive-report`) and [AI Agent Access](mcp-access.md)
(`executive_report` tool) if you'd rather pull it programmatically or
ask an agent for it than regenerate a PDF.

Next: [Scheduling Reports](scheduling.md) to get this running automatically.
