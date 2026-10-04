# Executive Report (Enterprise plan)

!!! note "This report requires the **Enterprise** plan"
    Every other report is free with the self-hosted
    pipeline. This is the exception -- see
    [gitultra.com](https://gitultra.com) for plan details, or apply for
    beta access there. Once you have access we issue you a license key
    (`GITULTRA_LICENSE_KEY`, same credential [API Access](api-access.md)
    uses) covering `"executive-report"`; without it the script exits
    with an actionable error rather than a partial/broken PDF.

Rather than another table of metrics, this one is a *judgment layer* on
top of the [other reports](generating-reports.md) -- a
configurable target per KPI, an on-track/at-risk/off-track status, a
weekly trend, and a fixed suggestion for what to look at next when
something's off track. Built entirely from the same data the other
three reports already compute; nothing new to configure beyond an
optional target-override CSV.

```
docker compose run --rm eng-reports executive_report.py --output /out/executive.pdf
```

Same `--period`/`--start`/`--end`/`--output` flags as the
[other reports](generating-reports.md#useful-flags), plus `--kpi-targets` (a CSV of `kpi,operator,target` rows --
defaults to 8 built-in KPIs covering deployment frequency, lead time,
change failure rate, MTTR, bug effort share, feature delivery,
high-priority bug share, and overdue ratio). Four more KPIs (cycle
time, ticket-to-first-commit lead time, ticket scope mismatch, and
planning signature) are available but off by default -- add a row to
your own copy of the CSV to turn one on. To set your targets once for
both the PDF and the API, put the CSV under `/var/lib/eng-metrics-suite`
and set `KPI_TARGETS_PATH` in your `.env` to its in-container path
(e.g. `/var/lib/eng-metrics-suite/kpi_targets.csv`). `eng-api` picks
up edits within a minute, no restart needed; `--kpi-targets` still
overrides it for a one-off run.
Whichever KPIs aren't
configurable on your instance (e.g. change failure rate without the
Jira integration configured) show as "not available," never a
fabricated number.

**No `--repo`/`--org`/`--team-map` scoping** -- like
[Investment Allocation](investment-allocation.md) and
[Planning Quality Signals](planning-quality-signals.md), this is one org-wide
view, not a drill-down tool. Also available via [API Access](api-access.md)
(`GET /v1/executive-report`) and [AI Agent Access](mcp-access.md)
(`executive_report` tool) if you'd rather pull it programmatically or
ask an agent for it than regenerate a PDF.

## Target history, baseline and annotations

`eng-api` keeps a history of your targets and where you started, so
results can always be read against the target that was in force at
the time, not just today's:

- **Target history.** Every change to your targets CSV is recorded
  with when it was made and the previous value. History starts on the
  day you upgrade to a version with this feature.
- **Two optional CSV columns.** `expected` records what you believe a
  KPI's value is today, before you see the measured one. `note` records
  why a target was chosen. Existing 3-column files keep working.

  ```
  kpi,operator,target,expected,note
  Lead Time for Changes,<=,48,36,Matches the platform team's SLA
  Change Failure Rate,<=,0.10,,Board asked for under 10% after the March outage
  ```

- **Baseline.** On first run, each KPI's value over your last full
  fiscal quarter is recorded as your starting point. Set
  `FISCAL_YEAR_START_MONTH` in `.env` (1 to 12, default 1) if your
  fiscal year doesn't start in January. A fiscal year is named for the
  year it ends.
- **Annotations.** An optional YAML file of dated notes about what
  your organization changed. Point `ANNOTATIONS_PATH` in `.env` at it:

  ```yaml
  annotations:
    - date: 2026-02-03
      note: Moved to a weekly on-call rotation
      by: Jane Smith        # optional
  ```

If an edit to either file is invalid, `eng-api` logs the problem and
keeps using the last good version.

Two commands help you manage this:

```
docker compose exec eng-api python manage.py status
docker compose exec eng-api python manage.py rebaseline --reason "Reorganized into platform teams"
```

`status` shows each KPI's target, baseline and expected value.
`rebaseline` records a new starting point, for example after a reorg
or acquisition. It never overwrites the old one. Add `--kpi "<name>"`
to re-baseline only some KPIs (for example once Jira is connected),
or `--quarter 2026-Q1` to pick the quarter.

!!! note "After updating to suite v1.10.0"
    High-Priority Bug Share, Overdue Ratio and Feature Delivery are now
    measured as of the end of each period, so past periods no longer
    shift as tickets close later. Their existing baselines are marked as
    outdated in `manage.py status`. Re-baseline them once:

    ```
    docker compose exec eng-api python manage.py rebaseline --kpi "High-Priority Bug Share" --kpi "Overdue Ratio" --kpi "Feature Delivery" --reason "Calculation now measured as of period end"
    ```

The [QBR Report](qbr-report.md) reads this history to show each
quarter in context.

!!! warning "Back up your database"
    Target history, baselines and annotations can't be rebuilt from
    git or Jira the way everything else can. Include your Postgres
    database in your backups.

Next: [Scheduling Reports](scheduling.md) to get this running automatically.
