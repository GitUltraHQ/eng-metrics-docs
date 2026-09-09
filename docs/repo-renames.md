# Repo Renames & Duplicate Repos

Renaming a repo on your provider (GitHub, GitLab, etc.) is safe to do at
any time — `git-processor` recognizes the same repo across a rename and
keeps a single, continuous history for it. This page explains how that
works, and how to check for repos that were duplicated by a rename that
happened before your `git-processor` was upgraded to a version with this
fix.

## The problem this solves

Before this fix, `git-processor` only recognized an already-tracked repo
by its clone URL. A rename changes that URL, so the next time
`discover_repos.py` ran, it saw the repo at its new URL and had no way to
tell that was the *same* repo — it queued a second row for it. From
that point on, every metric for that repo (commit counts, contributor
concentration, everything) silently double-counted across two separate
tracked repos, with no error or warning to notice it by.

## How it's fixed going forward

`git-processor` now also records each provider's own permanent repo ID
(GitHub/GitLab's numeric ID, Bitbucket's UUID, Azure DevOps' GUID)
alongside the URL. That ID doesn't change when a repo is renamed, so a
renamed repo's existing row gets its URL updated in place the next time
`discover_repos.py` runs, instead of being queued as a new one.

This applies to any repo added via `discover_repos.py` (GitHub, GitLab,
Bitbucket Cloud, Bitbucket Server, Azure DevOps). It doesn't apply to a
repo imported by pointing `git_processor.py` directly at a local clone —
that path has no provider API to ask for a stable ID, so it still
recognizes a repo by URL alone, same as before.

!!! note "You don't need to do anything for future renames"
    Just re-run `discover_repos.py` as usual (see
    [Discovering Repos](discovering-repos.md)) after a rename — the fix
    is automatic from there.

## Checking for repos already duplicated by a past rename

If a repo was renamed *before* your `git-processor` picked up this fix,
the duplicate row from that rename already exists and won't resolve on
its own — a normal discovery run only ever lists a provider's *current*
repos, so the old, orphaned row is invisible to it.

Run `find_duplicate_repos.py` to check:

```
docker compose run --rm --entrypoint python3 git-processor find_duplicate_repos.py
```

It uses the same provider credentials as `discover_repos.py`/`worker.py`
(whatever's already in your `.env`) — no separate setup needed. For each
repo still missing a stable ID, it asks that repo's own provider for its
*current* state using whatever URL is on file (even if it's stale most
providers redirect an old-path lookup the same way they redirect a `git
clone` on an old URL). If two rows turn out to be the same physical
repo, it prints both, for example:

```
Found 1 likely duplicate repo(s) -- same physical repo tracked under two rows:

  ACTIVE  id=42     github.com/acme/new-name  (https://github.com/acme/new-name)
  ORPHAN  id=17     github.com/acme/old-name  (https://github.com/acme/old-name)
  -> matched by the same current (domain, vendor_id); the ORPHAN row is almost certainly
     this repo's pre-rename name. Merging is NOT automatic -- review manually before
     deciding how to reconcile commits/tags/coauthors/tickets across the two repo_ids.
```

!!! warning "This is a report, not a fix"
    `find_duplicate_repos.py` never merges or deletes anything on its
    own. Deciding how to reconcile two repos' commit/tag/coauthor/ticket
    history — which one to keep, whether any data is actually missing
    from either side — depends on specifics only you can judge for your
    repo, so it's left as a manual step. If you hit this, reach out and
    we can help work through it.

A repo that's reported as missing from a provider entirely (deleted
upstream, not renamed) is left alone and not reported as a duplicate.

!!! note "Verification status by provider"
    This relies on each provider redirecting a lookup on an old
    name/path to the repo's current data. Confirmed for GitHub; expected
    to work the same way for GitLab and Bitbucket Cloud based on their
    documented behavior, but not independently verified here. Bitbucket
    Server and Azure DevOps are unverified end-to-end, same standing
    caveat as their discovery support generally (see
    [Discovering Repos](discovering-repos.md)).

Next: [Running Workers](running-workers.md) if you haven't already, or
back to [Discovering Repos](discovering-repos.md).
