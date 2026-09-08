# Running Workers

`git-processor` and `pr-processor` (already running from `docker compose
up`) pick up queued repos automatically and import commit/PR stats into
Postgres. `issue-processor` does the same for Jira data if you've
configured it (see [Jira Integration](jira-integration.md)). Watch
progress with:

```
docker compose logs -f git-processor pr-processor issue-processor
```

See [Logs](logs.md) for what each JSON line means.

## Continuous operation

Each worker runs forever, polling for new work every
`--poll-interval-seconds` (default 30) whenever its queue is empty --
it doesn't exit. This matters for two things:

- **Newly discovered repos/projects** get picked up automatically, no
  restart needed.
- **Already-imported repos/projects get resynced periodically** --
  every `--resync-after-minutes` (default 120), a worker re-imports
  anything that finished importing that long ago, to pick up new
  commits, new PR activity, or new Jira tickets. This is the *only*
  mechanism that keeps already-tracked data current -- without it, a
  repo's data would be frozen at whatever it was when first imported.
  Both intervals are configurable per-worker if you want fresher (or
  less frequent) updates.

`git-processor` specifically keeps a persistent local mirror of each
repo (a Docker-managed volume, `git_mirrors` in `docker-compose.yml` --
no setup needed) and updates it with `git fetch` on every resync,
rather than a full re-clone. This is what makes frequent resyncing
affordable even for large repos -- a fetch only pulls new objects. If a
mirror ever gets corrupted, the worker logs a warning and automatically
falls back to a fresh clone.

## Scaling workers

There's no worker-count setting — `git-processor` and `pr-processor` each
claim one repo at a time from the shared queue (`FOR UPDATE SKIP LOCKED`,
so concurrent claims never collide), and by default `docker compose up`
runs exactly one of each. To process more repos in parallel, run more
instances with Compose's `--scale`:

```
docker compose up -d --scale git-processor=4 --scale pr-processor=4
```

Since each replica now runs continuously rather than exiting once its
queue is empty, `--scale N` just means N long-running workers polling
and claiming concurrently -- `--max-attempts`/`--lease-minutes` are
per-repo, not per-worker, so scaling up doesn't change retry behavior,
it just means more repos get claimed per pass. `restart: unless-stopped`
still matters for recovering from a real crash, it just isn't the thing
keeping data fresh anymore -- the resync mechanism above is.

!!! warning "Bitbucket Cloud and scaling"
    Be more cautious scaling `pr-processor` if you're importing from
    **Bitbucket Cloud**: it's known to temporarily block source IPs that
    get hit too aggressively. `pr-processor` throttles and backs off
    automatically, but that protection is per-worker-process, not
    coordinated across replicas — N scaled workers still add up to N× the
    request rate from this host's IP. GitHub/GitLab's limits are generous
    enough that this isn't a practical concern there.

Next: [Generating Reports](generating-reports.md) once some data's imported.
