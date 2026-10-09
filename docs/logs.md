# Logs

All three application services log structured JSON lines to stdout — one
JSON object per line, with `timestamp`, `level`, `component`, `message`,
plus whatever fields are relevant to that specific event (e.g.
`identity_key`, `imported`, `failed`).

```
docker compose logs -f git-processor pr-processor
```

## Verbosity

Controlled by `LOG_LEVEL` in `.env` — `INFO` (default), `DEBUG`,
`WARNING`, or `ERROR`.

## Paused imports

If a provider rejects your credentials, the worker stops importing from
that provider and logs an `ERROR` line right away and again every 15
minutes, until you fix it and restart:

- `provider halted on rejected credentials` (git-processor,
  pr-processor). The `error` field names the provider and the `.env`
  settings to check.
- `Jira imports halted on a config problem` (issue-processor). The
  `error` field says whether Jira rejected the credentials or
  `JIRA_BASE_URL` doesn't point at a Jira site.

The worker keeps running while paused, so `docker compose ps` still
shows it up. Other providers keep importing.

## Failures

Failures include a full traceback under an `"exception"` field, not just
a one-line message — pipe through `jq` if you want to pull just that
field out. `docker compose logs` prefixes each line with the container
name by default, so add `--no-log-prefix` to get pure JSON lines `jq` can
parse:

```
docker compose logs --no-log-prefix pr-processor | jq -r 'select(.level=="ERROR") | .exception'
```
