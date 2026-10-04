# Updating

Each release of `eng-metrics-suite` (or `eng-metrics-suite-pro`) pins an
exact version of every component in its `docker-compose.yml`. Running
`docker compose pull` on its own only re-downloads the versions you
already have, so get the new compose file first.

In the folder where you cloned the suite:

```
git pull
docker compose pull
docker compose up -d
```

Your `.env` and anything under `/var/lib/eng-metrics-suite` aren't part
of the repo, so `git pull` leaves them alone.

To see which version you're on, run `git describe --tags`. To stay on a
specific release instead of the latest, check out its tag (for example
`git checkout v1.10.0`) before pulling images.

Some releases need a one-time step after updating, such as a
re-baseline. These are listed on the page for the feature they affect,
for example [Executive Report](executive-report.md#target-history-baseline-and-annotations).
