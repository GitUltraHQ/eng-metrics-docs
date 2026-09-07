# Getting Started

!!! tip "Using Claude Code?"
    `eng-metrics-suite` (and `eng-metrics-suite-pro`) ship a
    `gitultra-setup` skill that automates everything on this page
    through [Generating Reports](generating-reports.md) -- clone the
    repo, open it in Claude Code, and ask it to set up GitUltra. It
    detects your tier, asks only for what it can't infer (org name,
    provider tokens, which optional integrations to enable), and
    verifies the install with a real generated report before calling
    it done. The steps below are what it's automating, and remain the
    reference if you'd rather do it by hand or the skill doesn't
    behave as expected.

Clone [eng-metrics-suite](https://github.com/GitUltraHQ/eng-metrics-suite),
then:

```
cp .env.example .env
# edit .env: set a POSTGRES_PASSWORD, and the token(s) for whichever
# provider(s) you're importing from (GitHub/GitLab/Bitbucket/Azure DevOps)

sudo mkdir -p /var/lib/eng-metrics-suite
sudo chown "$(id -u):$(id -g)" /var/lib/eng-metrics-suite
mkdir -p reports

docker compose up -d
```

## Why the two `mkdir`s

`/var/lib/eng-metrics-suite` is where config files (not secrets — those
go in `.env`) live: `discover_repos.py`'s `--config` YAML (see
[Discovering Repos](discovering-repos.md)) and `report.py`'s `--team-map`
CSV (see [Generating Reports](generating-reports.md)) both get read from
there, mounted read-only into the relevant containers at the same path.
It needs to exist *before* `docker compose up` — Docker will auto-create
it as root-owned otherwise, which then blocks you from writing to it
without `sudo`.

`./reports` needs the same up-front `mkdir`, for a related but different
reason: `eng-reports` runs as a non-root user (UID 1000) so its output
isn't root-owned on the host, but that only helps once the directory
already exists with the right owner. If Docker has to auto-create
`./reports` itself (e.g. because you skipped this step), it does so as
root, and the non-root container then can't write into it at all
(`PermissionError: [Errno 13] Permission denied`). If your host user
isn't UID 1000 (check with `id -u`), add `--user "$(id -u):$(id -g)"` to
the `docker compose run` commands in [Generating
Reports](generating-reports.md) instead.

## What `docker compose up` actually starts

Postgres, plus the `git-processor` and `pr-processor` workers (and
`issue-processor`, if your compose bundle includes it — see
[Jira Integration](jira-integration.md)). The workers exit as soon as
they find nothing queued to do — that's expected, not a crash;
`restart: unless-stopped` just means they check again periodically
rather than sitting in a busy loop. Nothing happens until you queue
some repos — continue to [Discovering Repos](discovering-repos.md).
