---
name: create-release
description: 'Cut a production release by merging the Release PR that release-please keeps open on the base branch. Finds the PR by its configured head branch, shows its version and changelog, reads the SQL of the release''s new database migrations for destructive statements, merges only on the user''s explicit go-ahead, then watches the release workflow create the `vX.Y.Z` tag and the tag''s deploy run, and verifies production through the configured version URL. Use when the user runs /iwol:create-release or asks to "cut a release", "release", "deploy to production", "ship to production", "merge the release PR", or otherwise wants what has merged into the base branch released to production. Reads the repository''s release settings from .claude/iwol-config.md.'
---

# /iwol:create-release

Release what has merged into the base branch to production. release-please keeps one Release PR open on the base branch: its title carries the next version and its body is that version's changelog entry. Merging it is the only way to release:

1. The merge's push runs the release workflow, which creates the `vX.Y.Z` tag and the GitHub Release.
2. The tag push runs the deploy workflow, which passes its gate and then deploys to production. Other workflows may run on the tag too.

Run this only when the user asks to release or deploy to production. Never merge the Release PR just because the base branch looks ready. Nothing pushes to the base branch directly: every change reaches it through a reviewed PR, and a release is cut only by merging the Release PR.

## Config

Read `.claude/iwol-config.md` first. It holds everything specific to this repository. If it's missing, stop and ask the user to run `/iwol:setup`, which creates it.

Then look at its Releases section:

- **`Releases: none`:** say this repository has no release flow, and stop. Change nothing.
- **Otherwise,** take these values from it. If any is missing, as in a config written before the field existed, stop and ask the user to run `/iwol:setup` to add it. Don't guess a value.
  - **Release PR branch:** the Release PR's head branch.
  - **Release workflow:** the workflow file that creates the tag and the GitHub Release.
  - **Deploy workflow:** the workflow file the tag push runs, and what gates its deploy.
  - **Production version:** the URL that reports the running build's version, and the shape of its response.
  - **Migrations to check before a release:** the directories whose new migrations the release must be inspected for, or `none`.
  - **Rollback:** where rolling back is documented.

From the Git and GitHub section take the base branch (**Base branch**) and the **Merge method**. From the CI section take **Runs on** and **Recovering a stuck run**. Below, `main` means the base branch the config names, and `origin/main` means that branch on `origin`. `<release-branch>`, `<release-workflow>` and `<deploy-workflow>` mean the config's values.

## Steps

### 1. Sync `main`

```
git fetch origin
git checkout main
git pull --ff-only origin main
```

If it can't fast-forward, stop and tell the user. Don't merge or rebase over other people's commits.

In a linked worktree, `main` is usually checked out in the main checkout, so `git checkout main` fails. You're in a linked worktree when the two lines this prints are different absolute paths. It needs git 2.31 or later; `--path-format=absolute` is what keeps the main checkout from looking like a linked worktree when you run it from a subdirectory:

```
git rev-parse --path-format=absolute --git-dir --git-common-dir
```

Run only the fetch there. Every later step reads `origin/main`, not the local branch.

### 2. Find the Release PR

**First, make sure release-please has caught up with `main`,** whatever the PR list will return, because it may not have opened or updated the Release PR for the latest push yet. Check the latest release run on `main` against `origin/main`:

```
git rev-parse origin/main
gh run list --workflow <release-workflow> --branch main --limit 1 --json databaseId,status,conclusion,headSha
```

Trust the Release PR only when that run's `headSha` equals `origin/main`'s commit, its `status` is `completed` and its `conclusion` is `success`. If the `headSha` is another commit, the run for the latest push hasn't appeared yet: give it a few seconds and list again, for up to about two minutes, and if it still doesn't appear, stop and tell the user. If the run completed with any other `conclusion`, report the run and stop. If the run isn't `completed`, watch it, then check that it succeeded (report it and stop if it didn't):

```
gh run watch <run-id> --exit-status
```

Then list the PR:

```
gh pr list --state open --head <release-branch> --json number,title,url,body,headRefName,headRefOid,isCrossRepository,labels
```

`--head` matches the branch's name only, so a PR from a fork can be listed too. Proceed only when exactly one PR is listed, its `isCrossRepository` is `false` and it carries release-please's `autorelease: pending` label. Any PR other than that one, such as a second PR, a fork PR or a PR without that label, stop and show it to the user.

Note the PR's `headRefOid`: step 4 pins the merge to it.

**If no Release PR is open,** report that nothing releasable has merged since the last tag and stop. Only `feat`, `fix`, `perf`, `revert` and dependency commits open one. `chore`, `ci`, `docs`, `refactor`, `test` and `style` commits don't, however many have merged.

### 3. Show the release

- **Version:** the PR title carries it (e.g. `chore(main): release 1.11.0`). Note `X.Y.Z` for the later steps; the tag will be `vX.Y.Z`.
- **Changelog:** summarise the PR body, which is the changelog entry for this version. Group it the way the body does (features, bug fixes, and so on), and include the PR URL.
- **Commit read:** the commit `origin/main` is at (`git rev-parse origin/main`) when you run the checks below. Step 4 checks it hasn't moved before merging.

**Check the tag format.** This skill supports only v-prefixed tags (`vX.Y.Z`). Stop and tell the user, before merging, when the release's tag would be different, for example `<component>-vX.Y.Z`, which is what release-please creates for a release with a component unless `include-component-in-tag` is `false`. Look at `release-please-config.json` for a component on the release without that setting, at whether the compare link in the Release PR body names v-prefixed tags, and at the repository's existing release tags (`git ls-remote --tags --refs origin`).

**Check that the tag doesn't exist yet.** A hand-cut hotfix tag never updates `main`'s release manifest, so release-please can propose a version that's already released. The config's **Rollback** pointer, or the repository's release docs, may describe hand-cut hotfix tags:

```
git ls-remote --exit-code --tags origin "refs/tags/vX.Y.Z"
```

Exit code 2 means the tag is free. If it prints the tag (exit code 0), stop and don't merge. Merging would push no tag, so `main`'s changes would never deploy, and step 5 would find the old tag's green deploy run and report success. Tell the user to land a PR on `main` through the repository's normal PR flow (`/iwol:create-pr`) with a `Release-As: <next version>` footer in the body of the commit that reaches `main`, then run this skill again once the Release PR shows that version. With `Merge method: squash`, the squash commit gets a blank body by default, so the footer in a branch commit would never reach `main`: pass it with `gh pr merge <PR> --squash --body "Release-As: <next version>"`.

**Check the release for migrations.** If the config's migration directories are `none`, skip this check and say the repository has none to check. Otherwise, the release's deploy may apply pending migrations before the new build takes traffic, so the build still serving runs against the new schema until cutover. Find the last released tag reachable from `origin/main`:

```
git describe --tags --abbrev=0 --match 'v[0-9]*' origin/main
```

If it finds no tag (no `v` release yet), say there's no earlier release to diff against, and ask the user whether production already runs a build of this repository. If it does, use the commit it runs in place of `<last-tag>` below. If nothing is deployed yet, list every migration under the configured directories instead (`git ls-tree -r --name-only origin/main -- <migration-dirs>`), read each one's SQL, and skip the two diffs below.

Then, with that tag (or the commit production runs) in place of `<last-tag>` and the config's migration directories in place of `<migration-dirs>`, list the migrations the release adds and read their SQL:

```
git diff --name-status <last-tag> origin/main -- <migration-dirs>
```

```
git diff <last-tag> origin/main -- <migration-dirs>
```

**Read the SQL of every migration the release adds,** under every configured directory. Migration names are often generic, so judge by the statements themselves. Report:

- drops of a table, column or index;
- renames of a table or column;
- column type changes;
- new non-null columns without a default.

For each of these, tell the user what breaks while the old build serves against the new schema (e.g. queries that still read a dropped or renamed column), and what breaks on a rollback, since redeploying an older build doesn't undo the migration (see [Rollback](#rollback)). If the release adds no migrations, say so.

### 4. Merge on an explicit go-ahead

Ask the user to confirm, and merge only on an explicit go-ahead such as "merge it" or "go ahead and deploy". A clean changelog or an earlier "release when ready" isn't one. Production always needs the user's word at merge time.

The go-ahead can take a while, so check that nothing moved since the user saw the release. Right before the merge, run these as commands of their own:

```
git fetch origin
git rev-parse origin/main
gh pr view <PR> --json headRefName,headRefOid,isCrossRepository,labels
```

If the PR's `headRefName` isn't exactly `<release-branch>`, its `isCrossRepository` is `true` or it no longer carries the `autorelease: pending` label, don't merge: stop and show it to the user.

If `origin/main` isn't the commit step 3 read, or the PR's head isn't the `headRefOid` from step 2, don't merge: go back to step 2, including its check that the latest release run on `main` is for `origin/main`'s commit and has completed, re-run the checks, and ask again.

Then run the merge as a command of its own, never chained (`;`, `&&`, `|`) with anything else, using the config's merge method and pinned to that `headRefOid`:

```
gh pr merge <PR> --squash --delete-branch --match-head-commit <headRefOid>
```

With `Merge method: squash`, that is the command: drop `--delete-branch` when the config says GitHub deletes the branch on merge. With `Merge method: merge`, use `--merge` in place of `--squash`. release-please finds the merged Release PR by its branch and label, not by the commit message, so any merge commit is all it needs.

A permission classifier may block chained merge commands. If it still denies the merge, don't retry it in another form: ask the user to confirm again or merge it themselves. If the merge is refused because the PR's head no longer matches `<headRefOid>`, go back to step 2 as above. If the merge fails for another reason (a conflict, branch protection, a required check), report it.

Afterwards, in the main checkout, update `main` as a separate command:

```
git pull --ff-only origin main
```

### 5. Watch the tag and the deploy

The merge's push runs the release workflow, which creates the `vX.Y.Z` tag and the GitHub Release. Poll for that version's GitHub Release for up to about three minutes:

```
for _ in $(seq 36); do gh release view vX.Y.Z --json tagName,url,isPrerelease --jq '"\(.tagName) \(.url) prerelease=\(.isPrerelease)"' 2>/dev/null && break; sleep 5; done
```

If nothing prints, check the release run for the merge's commit on `main` (`gh run list --workflow <release-workflow> --branch main --limit 3`) and report what it shows. Don't create the tag or the GitHub Release yourself.

Then find the tag's deploy run. For a tag push the run's branch field is the tag name, so filter on it (the run can take a few seconds to appear):

```
gh run list --workflow <deploy-workflow> --branch vX.Y.Z --limit 1 --json databaseId,status,conclusion,url
```

Watch it to completion as a background Bash command (`run_in_background`) with a long `timeout` (`3600000`, an hour), since the gate plus the deploy's build can outlast the foreground limit:

```
gh run watch <run-id> --exit-status
```

- **The run failed:** the gate (the config names what gates the deploy) or the deploy failed. Report which job and step failed (`gh run view <run-id> --log-failed`) and stop. Don't retry blindly; a red gate on a tag means `main` itself is broken.
- **The run is stuck in `queued`:** this depends on the config's CI section.
  - **`Runs on: picker`:** the run is waiting for the Mac's self-hosted runners, usually because the Mac is asleep or offline. Check with `gh run view <run-id> --json status,jobs --jq '.status, (.jobs[] | "\(.name): \(.status)")'`, then recover it as the config's **Recovering a stuck run** says. If that says `none` or `none documented`, report the stuck run and ask the user.
  - **`Runs on: hosted`:** the run is only waiting for GitHub. Keep waiting, and report it if it stays queued.

Other workflows the tag push runs don't gate the production deploy; mention them in the report only if one failed.

### 6. Verify production

The config's **Production version** URL returns the running build's version, in the response shape the config describes. Allow a couple of minutes for the deploy's build to finish and take traffic after the deploy run goes green:

```
curl -s <production-version-url>
```

Expect the version to match step 3's `X.Y.Z`, in the config's response shape, with none of the extra suffix the config says other builds add. If it still shows the previous version after a few minutes, check the build logs on the deploy host before calling it a failure; a slow build isn't one.

### 7. Report

Give the user:

- the version;
- the Release PR URL;
- the tag (`vX.Y.Z`) and its GitHub Release URL;
- the deploy run URL and its result;
- the version URL's result, or that it's still pending.

## Rollback

Do this only when the user asks. Read the config's **Rollback** pointer first: the rollback it documents is the procedure. It says which tags can be redeployed and any faster rollback the deploy host offers. If it says `none`, tell the user the repository documents no rollback, and don't improvise one.

Dispatching the deploy workflow with an older tag as the ref rebuilds and redeploys that release:

```
gh workflow run <deploy-workflow> --ref vA.B.C
```

Use it only when the documented rollback says a dispatch redeploys to production. The config doesn't record whether the deploy workflow can be dispatched (without a `workflow_dispatch` trigger the command fails) or where a dispatch deploys, and a dispatch that goes to preview or staging runs green while production keeps the bad build. If the documentation doesn't say, don't dispatch: follow what it does say, or ask the user.

A dispatch runs the workflow file stored at that tag, so an older tag whose copy of the workflow differs from the current one may not deploy to production the same way. Check the documented rollback for tags this doesn't work for, and use the host's own rollback for those. A host-level rollback that swaps traffic back to the previous production build, where the host has one, is the faster option, since it doesn't rebuild.

**After any rollback,** check the config's **Production version** URL the way step 6 does, and confirm it reports `A.B.C`, the version rolled back to. Don't call the rollback done until it does; if it still shows another version after a few minutes, report that.

**Neither a redeploy nor a host-level rollback reverses a database migration.** The older build runs against the newer schema, so a release that dropped, renamed or retyped something stays broken on rollback. Fix that forward with a new migration and a new release instead.

## Guardrails

- **Never create a tag or a GitHub Release by hand** in this skill: no `git tag`, no `git push` of a tag, no `gh release create`. Only the release workflow creates them.
- **Never merge without the user's explicit go-ahead** at merge time.
- **Never merge any PR but the Release PR,** and never one whose head isn't the config's Release PR branch, comes from a fork or lacks release-please's `autorelease: pending` label. Any other PR into `main` goes through the repository's normal PR flow (`/iwol:create-pr`).
- **Never merge a Release PR whose `vX.Y.Z` tag already exists** (step 3's tag check), or whose tag wouldn't be v-prefixed (step 3's tag format check).
- **Never merge a Release PR or a base branch that moved since step 3 read it:** the merge is pinned to step 2's `headRefOid` and step 4 re-checks `origin/main`.
- **Never use admin or force options:** no `--admin` merges, no force-pushes, no skipped hooks (`--no-verify`).
- **Never push to `main` directly,** and never edit the Release PR's branch. release-please rewrites it on every push to `main`.
