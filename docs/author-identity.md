# Author Identity Consistency

The same person can show up under several different names/emails across
their commit history — this affects how accurately reports reflect
real contributor and team activity, and it's fixable.

## The problem

Git records whatever `user.name`/`user.email` was configured locally at
commit time, with no validation and no link back to a single "identity"
for a person. It's common for the same person to end up with several
distinct combinations in the same repo's history:

- A personal email on one machine, a work email on another
- A name typo, a shortened/incomplete name (`J Smith` vs. `Jane Smith`),
  or different capitalization
- A config that was never updated after a name change (e.g. after
  marriage, or switching teams/orgs)

Git treats each distinct `(name, email)` pair as a separate identity —
there's nothing under the hood that groups them back together on its
own.

## How it affects reports

`eng-reports` groups commit activity by (lowercased) `author_email`, and
rolls up `--team-map` assignments the same way. Both are direct
consequences of this:

- **Split individual stats** — if the same person committed under two
  different email addresses, they show up as two separate rows in the
  "By Author (commits)" table, each with a fraction of their real
  commit/line counts, instead of one accurate total.
- **Team rollups silently miscategorize people** — if your `--team-map`
  CSV lists `jane@work.com` but some of Jane's commits used
  `jane.smith@gmail.com`, those commits land in "Unmapped" instead of
  Jane's real team, even though the CSV itself is correct.

Neither of these produces an error — they just quietly under-report or
misattribute real activity, which is what makes this worth checking for
rather than assuming it isn't happening.

## Fixing it going forward: `.mailmap`

`.mailmap` is git's own native mechanism for this — a file at the root
of a repo mapping inconsistent identities to one canonical one:

```
Jane Smith <jane@work.com> <jane.smith@gmail.com>
Jane Smith <jane@work.com> J Smith <jane@work.com>
```

Each line is `<canonical name> <canonical email> [<messy name>]
<messy email>` — the bracketed name is only needed if the messy commit
used a different name, not just a different email. Add one line per
alias.

`git ls-stats`/`git ls-tags` (the tools `git-processor` uses to import
commit/tag data) resolve `author_name`/`author_email` and
`tagger_name`/`tagger_email` against a repo's `.mailmap` automatically,
the same way `git log`/`git shortlog` already do — there's no flag to
opt in or out.

!!! tip "You don't have to commit it to test it"
    `.mailmap` just needs to exist in the repo's working directory when
    `git-processor` runs — it doesn't need to be committed first. For a
    real, shared fix (and so plain `git log`/`git shortlog` show the
    same resolved identities for everyone), commit it like any other
    file.

## Reprocessing already-imported data

Adding a `.mailmap` doesn't retroactively fix commits `git-processor`
already imported, at least not on its own. A normal run only re-resolves
the single most-recently-imported commit (`git ls-stats` re-fetches its
own start point inclusively every time) — everything older than that
stays as originally imported.

Run `git_processor.py` with `--force-full-reimport` against the repo to
fully re-resolve every commit's identity:

```
docker compose run --rm -v /path/to/local/clone:/repo --entrypoint python3 \
  git-processor git_processor.py /repo --force-full-reimport
```

(You need a local clone of the repo to bind-mount in. The queue worker
keeps its own mirror of each repo, but there's no way yet to ask it for a
full re-import, so this runs against your clone instead.)

No extra setup needed for the bind mount itself — earlier versions of
this image required a manual `git config --add safe.directory` step to
work around a host/container UID-ownership check; that's now baked into
the image itself as long as you bind-mount to `/repo` (the path shown
above), so it works out of the box.

Tags don't need this step — `git ls-tags` output is fully reconciled
against the database on *every* normal run (not just full reimports),
so a tag's `tagger_name`/`tagger_email` picks up a new `.mailmap`
automatically the next time `git-processor` runs.

One thing `.mailmap`/reprocessing does **not** touch:
`Co-authored-by:` trailers in commit messages. Those are extracted as
plain text, not resolved as git identities (the same way `git` itself
doesn't apply mailmap to them) — a coauthor's name/email there stays
exactly as written in the commit message.

Next: [Generating Reports](generating-reports.md) to confirm the fix
actually collapsed those split rows.
