# Updating

Each release of `gitultra-metrics-suite` pins an
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

## Coming from eng-metrics-suite

The free plan used to have its own repo, `eng-metrics-suite`, which is
now archived. If you installed from it, switch to
[gitultra-metrics-suite](https://github.com/GitUltraHQ/gitultra-metrics-suite) (the same bundle as the paid plans) from
the folder you cloned it into:

```
git remote set-url origin https://github.com/GitUltraHQ/gitultra-metrics-suite.git
git fetch --tags --force
git reset --hard origin/master
docker compose pull && docker compose up -d
```

Your `.env`, your `reports/` folder and your database stay as they are:
nothing is re-imported. `--tags --force` matters: both repos used the
same version numbers (v1.0.0 to v1.8.0) for different commits, and this
replaces the old ones, so a later `git checkout <tag>` gets the right
files. After switching, `eng-api` and `gitultra-mcp` show up in
`docker compose ps`; on the free plan they log one "no license key"
line and idle.

`eng-metrics-suite-pro` was renamed to `gitultra-metrics-suite`. Existing
clones keep working through GitHub's redirect, so a plain `git pull` is
all you need; to update the URL anyway:
`git remote set-url origin https://github.com/GitUltraHQ/gitultra-metrics-suite.git`.

