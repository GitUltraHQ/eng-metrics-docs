# Discovering Repos

`git-processor` and `pr-processor`'s default command is their worker
(`worker.py`), so one-shot scripts like `discover_repos.py` need
`--entrypoint python3` to override that:

```
docker compose run --rm --entrypoint python3 git-processor discover_repos.py <org-or-group> --provider github
```

`--provider` is `github` (default), `gitlab`, `bitbucket_cloud`,
`bitbucket_server`, or `azure_devops` (Azure DevOps Services/cloud only —
Azure DevOps Server/TFS isn't supported). `<org-or-group>` is an
org/group/workspace, not a personal account — this discovers everything
the org owns, not one user's repos. Queues each repo found — safe to
re-run later to pick up new ones.

!!! note "Bitbucket Cloud credentials"
    Set `BITBUCKET_EMAIL` to your Atlassian account email (not your
    Bitbucket username) and `BITBUCKET_API_TOKEN` to a Bitbucket
    [API token](https://support.atlassian.com/bitbucket-cloud/docs/using-api-tokens/)
    with read access to repositories and pull requests. That's all:
    git-processor works out the git username on its own. Atlassian
    removed app passwords in July 2026. If your `.env` still uses
    `BITBUCKET_USERNAME` / `BITBUCKET_APP_PASSWORD`, it keeps working,
    and the logs remind you to rename them.

If the provider rejects your credentials, `discover_repos.py` stops
with a message naming the settings to check.

## Filtering what gets queued

To skip archived/inactive repos or ones matching a name pattern (e.g. bot
repos), or to track only part of an org, write a `discover_config.yaml`
to `/var/lib/eng-metrics-suite/` on the host:

```yaml
exclude_archived: true            # skip repos the provider reports as archived
exclude_inactive_days: 730        # skip repos with no push in this many days
exclude_patterns:                 # skip repos whose name (not "org/repo") matches
  - "-bot$"
  - "^terraform-"
include_patterns:                 # bring back a repo any rule above dropped
  - "^terraform-core$"
exclude_namespaces:               # skip whole orgs, groups, workspaces or projects
  - "^acme/sandbox(/|$)"
include_namespaces:               # bring back a namespace excluded above
```

There are two kinds of rules. In each, excludes are applied first and
includes bring repos back, and a repo is queued only if it passes both:

- **Name rules** match the repo name alone (`my-bot`, not `acme/my-bot`).
  `include_patterns` can bring back a repo dropped as archived, inactive
  or by an exclude pattern.
- **Namespace rules** match the repo's path minus its name: the GitHub
  org, every GitLab subgroup level (`acme/payments/cards`), the Bitbucket
  Cloud workspace, the Bitbucket Server project key, or the Azure DevOps
  project. `include_namespaces` only undoes `exclude_namespaces`; an
  archived or inactive repo in an included namespace stays out.

All patterns are regular expressions, not wildcards: `^payments` means
"starts with payments", while `payments*` means "payment" followed by any
number of "s". Every key is optional, a key left blank is the same as
leaving it out, and with no rules every repo is queued.

For example, to track only GitLab's `acme/payments` group and the
subgroups under it (the `(/|$)` keeps `acme/payments-legacy` out):

```yaml
exclude_namespaces: [".*"]
include_namespaces: ["^acme/payments(/|$)"]
```

Or only the repos whose names start with `payments`:

```yaml
exclude_patterns: [".*"]
include_patterns: ["^payments"]
```

Pass the config via `--config`:

```
docker compose run --rm --entrypoint python3 git-processor discover_repos.py <org-or-group> --provider github --config /var/lib/eng-metrics-suite/discover_config.yaml
```

Changing the rules later never untracks a repo that's already tracked;
use `remove_repo.py` (below) for that. Loosening them picks up the newly
included repos on the next run.

### Preview with a dry run

Before queuing anything, add `--dry-run` to see what your rules include
and exclude. It only talks to your provider, so it works before the
database is set up. Run it from the directory where you want the
results:

```
docker compose run --rm -T --user "$(id -u):$(id -g)" -v "$PWD:/out" -w /out \
    --entrypoint python3 git-processor /app/discover_repos.py <org-or-group> --provider github \
    --config /var/lib/eng-metrics-suite/discover_config.yaml --dry-run
```

The terminal shows a summary, with a count for each rule:

```
Dry run for gitlab 'acme' (nothing was queued)
  found:    3776
  include:  96
         96  rescued by include namespace '^acme/payments(/|$)'
  exclude:  3680
       3654  matched exclude namespace '.*'
         26  archived
Per-repo list: ./discover-dry-run-acme-20261014T054523Z.csv
```

The CSV lists every repo with its decision (`include` or `exclude`) and
the reason, including which include rule brought a repo back and what
had excluded it. On a large org it can run to tens of thousands of rows.

The `-v "$PWD:/out" -w /out` part puts the CSV in your current directory.
Without it the file is written inside the container, which is deleted
when the run ends. `--user` makes you the file's owner instead of root;
if your config file is readable only by root (for example, created with
`sudo` and mode 600), `chmod 644` it or drop `--user`.

!!! note "Coverage varies by provider"
    GitHub and GitLab support every key. Bitbucket Cloud has no "archived"
    concept (`exclude_archived` never matches there) and checks
    `updated_on` instead of push date for inactivity. Bitbucket Server's
    repo-list endpoint exposes neither signal, so only the pattern and
    namespace rules do anything there. Azure DevOps is similar to Bitbucket Server: its
    repo-list endpoint has no push date at all (`exclude_inactive_days` is
    a no-op), and its closest "archived" signal (`isDisabled`) means
    something slightly different — hidden org-wide, not just inactive —
    so treat `exclude_archived` there as an approximation.

Renamed a repo upstream? Re-running this is all you need to do — see
[Repo Renames & Duplicate Repos](repo-renames.md) for how that's handled
and how to check for any repos duplicated by a rename that happened
before you upgraded to a version with this fix.

## Untracking a repo

Decommissioned a repo, added one by mistake, or a customer asked you to
stop tracking it? `remove_repo.py` permanently deletes it and everything
that depends on it (commits, tags, PR data, any linked Jira project and
its incidents):

```
docker compose run --rm --entrypoint python3 git-processor remove_repo.py <identity_key> \
    --actor "you@company.com" --reason "decommissioned" --yes
```

`<identity_key>` is the normalized form `discover_repos.py`/`worker.py`
already use internally (e.g. `github.com/owner/repo`) — the same value
shown in logs and in a `SELECT identity_key FROM repos` if you need to
look it up. Omit `--yes` first to preview exactly what would be deleted
without changing anything. `--actor` is required and never inferred —
this self-hosted deployment has no user-identity system to infer it
from.

Once removed, this repo is permanently guarded against being silently
re-added — neither a future `discover_repos.py` run nor a manual
`git_processor.py <path>` import will resurrect it; both fail loudly
(the latter with an error pointing at the fix) instead of quietly
re-tracking it. If it should be trackable again later, lift the guard
deliberately:

```
docker compose run --rm --entrypoint python3 git-processor remove_repo.py <identity_key> \
    --allow-readd --actor "you@company.com" --yes
```

This only lifts the guard — it doesn't restore any deleted data, and
doesn't itself re-add the repo. Run `discover_repos.py` (or
`git_processor.py`) again afterward to actually re-track it.

Next: [Jira Integration](jira-integration.md) if you want change failure
rate/MTTR or an investment allocation report too, or straight to
[Running Workers](running-workers.md) to actually import what you just queued.
