# Planning Quality Signals Report

How well planning tracked actual execution, joining the same
allocation-tracking ticket data the
[Investment Allocation](investment-allocation.md) report uses against a
**new** link from commit to ticket -- if a commit's message
contains a ticket key (e.g. `ENG-123`), it's automatically picked up
(a best-effort match, not a Jira-verified one; nothing else needs
configuring).

```
docker compose run --rm eng-reports planning_report.py --output /out/planning.pdf
```

Same flags as [`allocation_report.py`](investment-allocation.md), plus `--max-tickets N`
(default 100) capping the per-ticket table. Two sections: **ticket-to-
first-commit lead time** (with a linkage-coverage number shown
prominently -- if your commits don't reference ticket keys, this
section won't have much to say, and the report tells you that plainly
rather than showing an empty chart) and **ticket scope mismatch**
(story points vs. actual change size, ranked by how unusual each
ticket is relative to your own team's typical ratio).
