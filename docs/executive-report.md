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
your own copy of the CSV to turn one on. Whichever KPIs aren't
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

Next: [Scheduling Reports](scheduling.md) to get this running automatically.
