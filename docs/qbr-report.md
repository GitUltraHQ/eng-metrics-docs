# QBR Report (Team plan)

!!! note "This report requires a paid plan"
    The QBR report is included in every paid plan. Your license key
    (`GITULTRA_LICENSE_KEY`) needs to cover `"qbr-report"`. Without it,
    the script exits with a message instead of producing a partial PDF.
    If your key was issued before this report existed, contact
    support@gitultra.com for an updated one.

The QBR report is a PDF for running a quarterly business review. It
shows the quarter in context: where your org started, what it aimed
for and when that changed, what it changed along the way, and how all
of that turned out.

```
docker compose run --rm eng-reports qbr_report.py --quarter 2026-Q3 --output /out/qbr.pdf
```

- `--quarter` takes a completed fiscal quarter, such as `2026-Q3`. A
  quarter that hasn't ended yet is rejected. Quarters follow
  `FISCAL_YEAR_START_MONTH` in your `.env` (see
  [Executive Report](executive-report.md#target-history-baseline-and-annotations)).
- The KPIs come from the same targets file as the Executive Report
  (`KPI_TARGETS_PATH`, or `--kpi-targets` for a one-off run).
- It's org-wide only. No individual is named or ranked, and your org is
  only ever compared with its own history, never with industry
  benchmarks.

## What's in it

1. **Summary.** How many KPIs landed in each band, and a short "How to
   read this report" box.
2. **KPI progress.** For each KPI: baseline, this quarter and target,
   the previous quarter, year to date, the same quarter last year, and
   a weekly chart of this quarter and the last one. Numbered lines on
   the chart mark your annotations and re-baselines.
3. **Target history.** Targets you changed this quarter, with your
   note and what happened next (for example, "Raised the bar and met
   it").
4. **What the org changed.** Your annotations and re-baselines for the
   quarter, in date order.
5. **Expectations vs results.** Only if you've recorded expected
   values. Each estimate next to what was measured over the 13 weeks
   before you entered it.
6. **Where to look next.** For KPIs in Early progress or Moved away:
   which related KPI or report to check, and how the related KPI moved
   this quarter.
7. **What this report is based on.** When your history started, which
   KPIs weren't available and why, and how many weeks of data each one
   had.

## Bands

Progress is how much of the distance from your baseline to your target
you covered by the end of the quarter.

| Band | Meaning |
| --- | --- |
| Met | The target was reached. A good time to consider a new one. |
| Sustained | Reached again, after also being met last quarter. |
| Substantial progress | Half or more of the way from baseline to target. |
| Early progress | Some of the way, less than half. |
| Moved away | Further from the target than the baseline. |
| No clear change | The difference is within normal week-to-week variation. |
| Not enough data to tell | Fewer than 8 weeks of data in the quarter or the baseline. |
| Target adjusted to current levels | Met, but only after the target was loosened to where the KPI already was. |

A KPI shows no band, and says why, when it has no target recorded for
the quarter, no baseline from before the quarter, or a baseline that
needs re-baselining.

The report never says that one of your changes caused a movement. It
shows when things happened and leaves the interpretation to the people
in the room.

## Before your first QBR

The QBR reads the target history and baseline that `eng-api` records
(see [Executive Report](executive-report.md#target-history-baseline-and-annotations)).
History starts on the day you upgrade, so the first quarter with a
full picture is the first one that starts after that. Earlier quarters
still work, and the report says plainly where history is missing.

Next: [Scheduling Reports](scheduling.md) to run it automatically after
each quarter ends.
