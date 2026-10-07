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

## One file for your whole org (recommended)

A `.mailmap` committed to a repo only covers that repo, so someone who
works across 50 repos would need the same lines in all 50. Instead, keep
one identity file for your whole org, in the same `.mailmap` format:

```
Jane Smith <jane@work.com> <jane.smith@gmail.com>
Bob Brown <bob@work.com> <bob@personal.net>
```

Save it in `/var/lib/eng-metrics-suite` on the host (already mounted into
`git-processor`) and point `MAILMAP_PATH` in `.env` at it:

```
MAILMAP_PATH=/var/lib/eng-metrics-suite/identities.mailmap
```

Then restart `git-processor` (`docker compose up -d git-processor`).
Every repo's import applies the file from then on, alongside any
`.mailmap` a repo already has committed. Where both map the same
address, your org-wide file wins. If the path doesn't point to a
readable file, `git-processor` refuses to start, so a typo doesn't go
unnoticed.

## Reprocessing already-imported data

A new or changed mapping only applies to commits imported after the
change. Commits that were already imported keep the identity they had,
so existing reports don't change until you re-import.

To re-import, run `reingest_repo.py`. `git-processor` re-reads every
commit from its own copy of each repo, so you don't need a local clone,
and it updates names and emails on the commits it already has. Commit
counts and other stats stay the same.

After editing the org-wide file, re-import every repo:

```
docker compose run --rm --entrypoint python3 git-processor reingest_repo.py --all --yes
```

After changing one repo's committed `.mailmap`, re-import just that repo,
using its identity key (e.g. `github.com/acme/api`):

```
docker compose run --rm --entrypoint python3 git-processor reingest_repo.py github.com/acme/api --yes
```

Run it without `--yes` first to see what it will touch. Repos that are
mid-import are skipped; run the command again once they finish. Updated
identities show up in reports once `git-processor` has worked through
the queue.

If you imported a repo by hand with `git_processor.py` instead of through
discovery, `git-processor` has no copy of it, so re-import it from a
local clone instead:

```
docker compose run --rm -v /path/to/local/clone:/repo --entrypoint python3 \
  git-processor git_processor.py /repo --force-full-reimport
```

No extra setup is needed for the bind mount, as long as you mount the
clone at `/repo` as shown.

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
