# Scheduling Reports

`docker compose run` is a normal one-shot command — wire it into a host
cron job or systemd timer to get a report on a recurring schedule (e.g.
weekly, for a Monday-morning exec digest):

```
0 8 * * 1 cd /path/to/eng-metrics-suite && docker compose run --rm eng-reports report.py --period last-week --output /out/weekly-report.pdf
```

Adjust the cron expression, `--period`, and output filename to taste —
nothing about scheduling is special-cased in `eng-reports` itself, it's
just a script that takes `--output` and exits.

## Emailing team reports to managers

If your `--team-map` CSV has `role`/`receives_report` columns (see
[Generating Reports](generating-reports.md#optional-columns-manager-reporting-and-pseudonymization)),
`email_team_reports.py` — a second script in the same `eng-reports`
image — will generate a **fresh** report for each team a manager
oversees and email it to them directly, on whatever schedule you wire
it into:

```
0 8 * * 1 cd /path/to/eng-metrics-suite && docker compose run --rm eng-reports email_team_reports.py --team-map /var/lib/eng-metrics-suite/teams.csv
```

Each `role=manager,receives_report=true` row gets its own email, one
per team they manage — a manager who oversees two teams gets two
separate emails, never a combined one. Every run regenerates the
report fresh (never re-sends a stale PDF from an earlier run).

Requires SMTP credentials (**your own relay, never GitUltra's** — a
manager's email address only ever leaves your own infrastructure):

```
SMTP_HOST=smtp.yourcompany.com
SMTP_PORT=587
SMTP_USERNAME=...
SMTP_PASSWORD=...
SMTP_FROM_EMAIL=reports@yourcompany.com
```

Set these as environment variables on the `eng-reports` service (see
`.env.example`) — same `${VAR:-}`-style optional config as every other
credential in this bundle. If a specific team's report fails to
generate (e.g. no recent activity for that team), that failure doesn't
block other teams' emails from going out; the script exits non-zero at
the end if anything failed, so a cron failure notification still fires.
