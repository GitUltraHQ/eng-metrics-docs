# Jira Integration (Optional)

`issue-processor` is an optional fourth service that unlocks two
independent capabilities, sourced from Jira:

- **Change failure rate / MTTR** -- the two DORA metrics `git-processor`/
  `pr-processor` alone can't compute, since they need an incident
  tracker, not just git/PR history. Shows up in `report.py`'s PDF and
  the `/v1/change-failure-rate`/`/v1/mttr` API endpoints.
- **Investment allocation** -- how work breaks down by category
  (feature/bug/tech-debt/etc.), story points, and cycle time. Shows up
  in the separate `allocation_report.py` PDF (see
  [Generating Reports](generating-reports.md)) and the
  `/v1/investment-allocation` API endpoint.

Enable one, the other, or both -- they're independently configured.
`issue-processor` idles cleanly if neither is set up (same "empty
queue -> clean exit -> restart" pattern as `git-processor`/
`pr-processor`), so it's safe to leave running in your compose bundle
even before you configure this.

!!! note "Jira Cloud only"
    Jira Server/Data Center isn't supported -- its REST API differs
    enough (different auth, no v3 endpoint parity) to be a separate
    follow-up.

## Shared credentials

Both flows authenticate the same way. Add to `.env`:

```
JIRA_BASE_URL=https://yourcompany.atlassian.net
JIRA_EMAIL=you@yourcompany.com
JIRA_API_TOKEN=...
```

`JIRA_API_TOKEN` is a Jira Cloud [API token](https://id.atlassian.com/manage-profile/security/api-tokens),
used as the password half of Basic auth (`JIRA_EMAIL:JIRA_API_TOKEN`).

## Change failure rate / MTTR

Linkage works via Jira's **Fix Version** field: an incident counts
against a deployment if its Fix Version exactly matches that
deployment's git tag name, scoped per repo. This requires your Jira
project's Fix Versions to already agree with your git tag naming --
there's no fuzzy matching or time-window fallback (same "no fabricated
proxy" philosophy as deployment frequency/lead time above).

1. Add to `.env`:

    ```
    JIRA_INCIDENT_ISSUE_TYPES=Incident,Bug
    ```

    Comma-separated Jira issue-type names that count as an "incident."
    **Required to enable this flow, no default** -- an unfiltered list
    (every issue counted as an incident) would silently corrupt both
    metrics.

2. Map each Jira project to the repo whose tags its Fix Versions apply
   to. Copy
   [`jira_project_map.example.yaml`](https://github.com/GitUltraHQ/issue-processor/blob/master/jira_project_map.example.yaml)
   to `/var/lib/eng-metrics-suite/jira_project_map.yaml` and edit it,
   then seed the queue:

    ```
    docker compose run --rm issue-processor python3 discover_jira_projects.py \
        --config /var/lib/eng-metrics-suite/jira_project_map.yaml
    ```

    This is a hand-maintained mapping, not auto-discovered -- there's
    no Jira API that can derive which project's Fix Versions apply to
    which repo. Safe to re-run later to add more mappings.

Once `issue-processor` has imported some incidents, `report.py`'s PDF
and the `/v1/change-failure-rate`/`/v1/mttr` API endpoints return real
numbers instead of "not configured."

**Untracking a project**: `remove_jira_project.py` permanently deletes
the mapping and its incidents:

```
docker compose run --rm --entrypoint python3 issue-processor remove_jira_project.py <project_key> \
    --repo-identity-key "github.com/owner/repo" \
    --actor "you@company.com" --reason "decommissioned" --yes
```

Same preview-first/`--yes`/`--allow-readd` contract as
[`remove_repo.py`](discovering-repos.md#untracking-a-repo) -- once
removed, `discover_jira_projects.py` won't silently re-add this
project/repo pairing until you deliberately lift the guard.

## Investment allocation

A separate flow with no repo relationship at all -- Jira tickets aren't
tied to a specific git repo the way incidents are tied (via Fix
Version) to a deployment, so this doesn't even require
`git-processor`'s schema to exist first.

1. Add to `.env`:

    ```
    JIRA_ALLOCATION_ISSUE_TYPES=Story,Task,Bug
    ```

    Comma-separated Jira issue-type names that count as "work" for
    this report. **Required to enable this flow, no default** -- an
    unfiltered list would mix in issue types (e.g. Epics, Sub-tasks)
    that don't belong in a ticket-level breakdown.

    Two more variables are optional:

    ```
    JIRA_CATEGORY_FIELD=customfield_10032
    JIRA_STORY_POINTS_FIELD=customfield_10016
    ```

    - `JIRA_CATEGORY_FIELD` -- the custom field ID holding each issue's
      investment category. If unset, every issue's category is its
      Jira issue type instead (a global fallback, not evaluated
      per-issue).
    - `JIRA_STORY_POINTS_FIELD` -- the custom field ID holding story
      points (varies per Jira instance, no universal default). If
      unset, story-point figures show "n/a"; ticket counts are
      unaffected.

2. List which Jira projects to track -- no repo mapping needed. Copy
   [`allocation_projects.example.yaml`](https://github.com/GitUltraHQ/issue-processor/blob/master/allocation_projects.example.yaml)
   to `/var/lib/eng-metrics-suite/allocation_projects.yaml` and edit
   it, then seed the queue:

    ```
    docker compose run --rm issue-processor python3 discover_allocation_projects.py \
        --config /var/lib/eng-metrics-suite/allocation_projects.yaml
    ```

Once imported, generate the report:

```
docker compose run --rm eng-reports allocation_report.py --output /out/allocation.pdf
```

or query `/v1/investment-allocation` if you're on the Team/Enterprise
API plan (see [API Access](api-access.md)).

**Untracking a project**: `remove_allocation_project.py` permanently
deletes it and its work items:

```
docker compose run --rm --entrypoint python3 issue-processor remove_allocation_project.py <project_key> \
    --actor "you@company.com" --reason "decommissioned" --yes
```

Same preview-first/`--yes`/`--allow-readd` contract as
[`remove_repo.py`](discovering-repos.md#untracking-a-repo) -- no
`--repo-identity-key` needed here, since this flow has no repo
relationship at all.

## Further reading

See [issue-processor](https://github.com/GitUltraHQ/issue-processor)'s
own README for the full schema, the claim-queue/retry mechanics shared
with `git-processor`/`pr-processor`, and known limitations (Fix Version
exact-match only, category fallback semantics, cycle time being full
ticket lifecycle rather than active-work time, logged time being a
report-period snapshot rather than a weekly trend).

Next: [Running Workers](running-workers.md) once you've configured
either flow above -- `issue-processor` picks up whatever you've queued
the same way `git-processor`/`pr-processor` do.
