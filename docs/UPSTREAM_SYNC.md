# Upstream sync (INF-225)

This fork tracks [nextcloud/desktop](https://github.com/nextcloud/desktop). `.github/workflows/upstream-sync.yml`
automates pulling in upstream changes so drift doesn't silently compound.

## Cadence

Weekly, Monday 06:00 UTC (`workflow_dispatch` is also available for an on-demand run from the
Actions tab). Weekly was chosen over monthly because the gap was already ~4 days of drift when
this was set up — staying weekly keeps each catch-up small instead of letting conflicts compound
over a longer window.

## What the workflow does

1. Fetches `upstream/master` (`nextcloud/desktop`) and compares it against `origin/master`.
2. If there's drift, creates a dated branch `sync/upstream-<date>` off current `master` and
   attempts `git merge upstream/master`.
3. **Clean merge** → pushes the branch and opens a PR against `master` for review.
4. **Conflict** → aborts the merge (no branch is pushed), and opens an Issue listing the
   conflicting files, assigned to `v-giannakopoulos`, with the commands to resolve manually.

## Branch convention

Non-sync (rebrand/normal) work continues to land **directly on `master`**, same as today —
this was not changed by INF-225.

The sync process itself is the one exception: it **never pushes to `master` directly**. Every
sync always goes through a PR (or, on conflict, a manual merge you drive) for human review before
landing — `theme/colored/*` in particular is a rebrand collision point that can conflict (or
merge cleanly but silently overwrite rebrand work) even when git reports no conflict markers, so
every sync gets eyes on it before merging.

## Conflict handling

When the workflow opens a conflict Issue:

```
git fetch upstream master
git checkout master
git checkout -b sync/upstream-manual
git merge upstream/master
# resolve conflicts, paying particular attention to theme/colored/* and any other
# rebrand-touched files
git push origin sync/upstream-manual
# open a PR against master once resolved
```

Close the Issue once the resulting PR is merged (or once you've determined the conflict doesn't
need resolving this cycle).

## Required repo/org settings

- Repo Settings → Actions → General → Workflow permissions: the workflow declares its own
  `contents/pull-requests/issues: write` permissions block, so the repo-wide default can stay at
  "Read repository contents permission" (no need to broaden it for every other workflow in this
  repo).
- Repo Settings → Actions → General → "Allow GitHub Actions to create and approve pull requests"
  must be **enabled** — without it the clean-merge PR step fails
  (`GitHub Actions is not permitted to create or approve pull requests`). If the Krateos-BV org
  has a blanket policy disabling this, the org-level setting must be enabled first (org
  Settings → Actions → General) — a repo-level toggle alone will 409 if the org blocks it.

## Rollback

- Disable: repo Settings → Actions → disable, or delete
  `.github/workflows/upstream-sync.yml` and revert that commit.
- Remove stray branches: `git push origin --delete sync/upstream-<date>`.
- Close any erroneous bot-opened PRs/Issues.
