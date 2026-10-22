# Telemetry

Each GitUltra service that connects to your database sends an
anonymous usage heartbeat to gitultra.com at most once every 24 hours:
`git-processor`, `pr-processor`, `issue-processor` (if you've set up
Jira), `eng-reports`, and `eng-api` (paid plans). It contains:

- An installation identifier — a UUID generated once and shared by
  every service using your database. Not tied to your identity,
  your organization's name, or anything else identifying.
- Which service sent it (`git-processor`, `pr-processor`,
  `issue-processor`, `eng-reports`, or `eng-api`).
- The software version (the git commit the running image was built
  from).
- A broad, *bucketed* count of tracked repositories (e.g. `"6-25"`) —
  never an exact number.

**It never includes repository names, code, commit content, author
names/emails, or anything else about what you're actually tracking.**

This is on by default. It's designed to never affect your instance: it
always fails silently rather than raising an error, and any network
attempt is capped at a few seconds even when your instance has no
internet access at all (e.g. an air-gapped deployment) — this happens
at most once every 24 hours per service, not on every run.

## Disabling it

Set `TELEMETRY_DISABLED=1` in your `.env` (or directly in the
environment for each service):

```
TELEMETRY_DISABLED=1
```

## Overriding the destination

`TELEMETRY_ENDPOINT` overrides where the heartbeat is sent — mainly
useful if you're testing or auditing this behavior yourself and want to
point it at your own HTTP listener instead.

## Shared secret

Each heartbeat request carries an `X-Telemetry-Secret` header, checked
by the receiving endpoint to filter out naive/automated traffic hitting
a known public URL. This is **not** real authentication — the default
value is a fixed constant baked into the (publicly pullable) images, so
it's trivially recoverable by anyone who looks. `TELEMETRY_SHARED_SECRET`
overrides it, but only matters if you're deliberately rotating away from
the default (and you'd need to set the same value on `gitultra-site`
too, which isn't something a self-hoster running this pipeline would
normally do).
