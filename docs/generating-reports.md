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

Next: [Scheduling Reports](scheduling.md) to get this running automatically.
