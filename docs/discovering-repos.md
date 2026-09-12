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

## Filtering what gets queued

To skip archived/inactive repos or ones matching a name pattern (e.g. bot
repos), write a `discover_config.yaml` to `/var/lib/eng-metrics-suite/`
on the host:

```yaml
exclude_archived: true            # skip repos the provider reports as archived
exclude_inactive_days: 730        # skip repos with no push in this many days
exclude_patterns:                 # skip repos whose "org/repo" name matches (regex, re.search)
  - "-bot$"
  - "^terraform-"
```

Then pass it via `--config`:

```
docker compose run --rm --entrypoint python3 git-processor discover_repos.py <org-or-group> --provider github --config /var/lib/eng-metrics-suite/discover_config.yaml
```

!!! note "Coverage varies by provider"
    GitHub supports all three keys. Bitbucket Cloud has no "archived"
    concept (`exclude_archived` never matches there) and checks
    `updated_on` instead of push date for inactivity. Bitbucket Server's
    repo-list endpoint exposes neither signal, so only `exclude_patterns`
    does anything there. Azure DevOps is similar to Bitbucket Server: its
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
