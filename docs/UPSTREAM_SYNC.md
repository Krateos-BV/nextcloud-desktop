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
4. **Conflict** → aborts the merge (no branch is pushed) and fails the run, with the conflicting
   files and manual-resolution commands written to the run's job summary. Issues are disabled on
   this repo, so a failed scheduled run (and its notification) is the alert — there's no Issue.

`.github/workflows/` is always kept as this fork's own version and excluded from the merge
result, regardless of what upstream changed there. This isn't a preference — GitHub's
`GITHUB_TOKEN` is hard-blocked from ever pushing changes to workflow files (anti-self-escalation
protection with no permission that lifts it), so a sync that included upstream's workflow changes
would fail to push on every run where upstream touched CI. If upstream's CI is ever worth pulling
in deliberately, do that merge by hand with a personal token that has the `workflow` OAuth scope.

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

## Required repo/org settings

- **Krateos-BV org Settings → Actions → General → "Workflow permissions" must be "Read and write
  permissions"**, and **"Allow GitHub Actions to create and approve pull requests" must be
  enabled.** Both are org-level; a repo-level attempt to raise either while the org is more
  restrictive silently has no effect or 409s. Confirmed enabled 6 Sep 2026.
- The PR-creation step uses `gh api repos/.../pulls` (REST), not `gh pr create`. `gh pr create`
  drives GitHub's GraphQL `createPullRequest` mutation, which 403s
  (`Resource not accessible by integration`) for `GITHUB_TOKEN` on forked repos even when the
  above settings are correctly enabled and the equivalent REST call with the same token succeeds.
  Verified end-to-end via `workflow_dispatch` against a disposable test branch on 6 Sep 2026 —
  REST succeeded immediately after the org settings were confirmed; GraphQL (`gh pr create`)
  still failed under identical permissions.

## Rollback

- Disable: repo Settings → Actions → disable, or delete
  `.github/workflows/upstream-sync.yml` and revert that commit.
- Remove stray branches: `git push origin --delete sync/upstream-<date>`.
- Close any erroneous bot-opened PRs/Issues.
