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

Next: [Jira Integration](jira-integration.md) if you want change failure
rate/MTTR or an investment allocation report too, or straight to
[Running Workers](running-workers.md) to actually import what you just queued.
