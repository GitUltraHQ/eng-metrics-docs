# Investment Allocation Report

Shows how work breaks down by category (feature/bug/tech-debt/etc.),
sourced from Jira ticket data via the optional
[Jira Integration](jira-integration.md)'s allocation-tracking flow
(separate from its change-failure-rate/MTTR flow -- configure either or
both). Another independent script in the same `eng-reports` image as
the [org report](generating-reports.md).

```
docker compose run --rm eng-reports allocation_report.py --output /out/allocation.pdf
```

Same `--period`/`--start`/`--end`/`--output` flags as `report.py`
(see [Useful flags](generating-reports.md#useful-flags)), but **no
`--repo`/`--org`/`--team-map` scoping** -- it always
covers every tracked Jira project as one combined report (Jira tickets
have no repo relationship the way commits/PRs do, so there's nothing
to scope by). If the allocation flow isn't configured at all, the PDF
renders a single "not configured" page rather than misleading empty
tables.
