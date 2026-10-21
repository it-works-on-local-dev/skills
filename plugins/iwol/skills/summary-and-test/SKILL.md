---
name: summary-and-test
description: Summarize the changes on the current branch or a pull request that isn't a Beads wave, check them against the code, walk the user through manual testing one step at a time, then fix what failed in the same session and re-test it. Use when the user runs /iwol:summary-and-test, or asks to "summarize and test this", "walk me through testing this branch", "manually test this PR", or wants a brief recap plus guided manual verification of work done without beads. For a Beads epic or wave, use /iwol:beads-summary-and-test instead.
---

# Summary and Test

Recap what a branch or PR changes, confirm it against the code, then walk the user through a manual test one step at a time, with concrete URLs. When testing ends, fix what failed and re-test it. Brevity is the point: this is a wrap-up tool, not a report.

It's the counterpart of `/iwol:beads-summary-and-test` for work that isn't a Beads wave: changes made in a session, on a branch, or in a PR. Run it before `/iwol:create-pr`, or while the PR is a draft.

Read `.claude/iwol-config.md` first. It holds everything specific to this repository: check commands, generated files, local services, files to leave alone and the PR rules. If it's missing, stop and ask the user to run `/iwol:setup`, which creates it. In these steps, `main` means the base branch the config names, and `origin/main` means that branch on `origin`.

## Step 0 — Find what to test

- **A PR number or URL in the args:** test that PR. The test runs this checkout's code, so its branch has to be the one checked out. If `git branch --show-current` isn't the PR's `headRefName` (`gh pr view <PR> --json headRefName,isDraft`), offer `gh pr checkout <PR>` when `git status --short` shows no changes to tracked files; otherwise stop and tell the user which branch to switch to. If `git worktree list` shows the branch checked out in another folder, such as a worktree `/iwol:start` made, move there instead, as *Entering one* in `../start/worktrees.md` says: git checks a branch out in only one worktree, and an earlier session's uncommitted fixes are there. *Entering one* stops and tells the user to run this skill from that folder when it isn't under `.claude/worktrees/`.
- **No args:** test the current branch: everything it changes against `origin/main`, committed or not. If it's `main` with nothing changed, say there's nothing to test and stop. If the branch has an open PR (`gh pr view --json number,isDraft`), that's the PR this skill saves a split plan on.
- **A Beads epic or bead ID:** that's `/iwol:beads-summary-and-test`. Say so and stop.

If the PR is ready rather than a draft, say in one line that `/iwol:create-pr` merges a ready PR once its review is clean, which could happen mid-test, and that `gh pr ready <PR> --undo` makes it a draft while testing.

## Step 1 — Gather the changes

- Run `git fetch origin main`. The committed changes are `git log --oneline origin/main..HEAD` and `git diff origin/main...HEAD`; the uncommitted ones are `git diff HEAD` and the untracked files, one by one, in `git status --short --untracked-files=all`, apart from those at or under a path the config's Files to leave alone section lists.
- What the change is meant to do comes from the PR's title and description, the commit messages, and this conversation when this session made the changes.
- Group the changes into a few units the user would recognise: a feature, a fix, a refactor.

Then look on the PR, if there is one, for its latest comment starting `Manual test plan` (see *Saving the plan*). If that plan has steps `not run`, an earlier session split the test into groups, and this session carries on with it. Skip Steps 2 and 3 for the changes the plan already covers, and go to *Continuing a split test*.

## Step 2 — Check the claims against the code

What the PR description, the commit messages or this conversation say happened isn't proof; the code is. For each unit, confirm the change is really there (`git diff`, `Read` the current file, committed and uncommitted). Note anything described but missing, done differently from how it's described, or reverted since, and any change nothing describes. Don't re-run full test suites here unless something looks contradictory.

Each mismatch is a finding (see *Recording findings*), with no test step. Keep this step internal: don't show diffs or file lists.

## Step 3 — Give a brief summary

- One line per unit: what changed, in plain terms.
- One line on Step 2's mismatches, or a half-line saying there were none.
- Nothing else, apart from a one-line transition into testing.

## Step 4 — Test one step at a time

Before showing anything, silently plan the manual test steps covering every unit's user-observable behaviour. Units with nothing to see on screen (docs, tests, internal changes) get no step: check them yourself where you can. Write, split and run the steps by `test-steps.md` in this skill's folder: how each step reads, groups of about 10 steps with one group per session, and one step per message. Read it before planning.

A plan split into groups needs a PR to be saved on, so the next session can pick it up. With no PR, don't split: run the whole plan in this session, and tell the user before step 1 how long it is.

### Recording findings

Keep the findings in this session, as a numbered list, until testing ends: the step, what it expected, what the user saw, and where (screen and URL, or `file:line` if known). Don't fix anything yet, even if it looks quick: a fix mid-test changes what the later steps test. If the user wants it fixed right away, say it'll be fixed as soon as this session's steps are done, or offer to end testing there.

### Saving the plan

Save the plan only when it's split into groups and the branch has a PR. Post it as one PR comment before sending the first step, with every step `not run`:

```
gh pr comment <PR> --body-file <file>
```

Then update that same comment whenever this session's testing ends (after its group's last step, or when the user stops), and again after *Fix and re-test*. Find it, the PR's latest plan comment, with `gh api --paginate repos/{owner}/{repo}/issues/<PR>/comments --jq '.[] | select(.body | startswith("Manual test plan")) | .id' | tail -n 1` (comments come oldest first), and edit it with `gh api -X PATCH repos/{owner}/{repo}/issues/comments/<id> -F body=@<file>`.

The comment starts `Manual test plan (<date>)`. Before each group's steps, a `Group <g> of <G> — <title>` line. One line per step: `Step <step> — <title>: start <URL, or device and screen or deep link>; setup <what you did first, such as the SQL you ran, or "none">; <action>; <expected> — pass | fail: <what the user saw> | fail: <what the user saw>; fix tried in <files> | fixed in <files>, re-test pass | blocked | not run`, where `<files>` lists the files the fix changed. It ends with a `Covers uncommitted: <file>@<blob>, …, or none` line and a `Head: <sha>` line, the commit the plan was made from. `Covers uncommitted:` holds what the plan covers beyond `Head:`: the uncommitted and untracked files it was made from, and the files its fixes changed. Each file's blob is what `git hash-object -w <file>` prints when the plan is saved after planning from that file or fixing it (`-w` keeps the blob, so a later session can diff against it), or `deleted` for a file that's gone, and replaces the file's earlier blob. No other save changes a blob, so a change nothing planned from or fixed stays uncovered. Never start a line of the comment with `/review`: that requests a review.

## Step 5 — Fix and re-test

When this session's steps are done, or the user stops early:

1. List the findings, one line each. With none, go to the close.
2. Offer to fix them now, in this session. On the user's yes, fix them, then run the narrowest checks that cover what you changed, per the config's Checks section. If the user declines, leave the findings listed (and in the saved plan) and go to the close.
3. Re-test: send each failed step again, and each step it blocked, one at a time and in the same format, checked against the code as it now is. A step that still fails is a finding again. Offer another fix, but after two tries at the same step, stop and leave it with the user.
4. Update the saved plan, if there is one, with the new blob of each file the fixes changed in `Covers uncommitted:`.

Don't commit or push. The fixes stay in the working tree for `/iwol:create-pr`, which commits them, or adds them to the PR when the user says so.

Then give a one-line close: how many of this session's steps passed, failed or were blocked, what was fixed and re-tested, and the next step. While groups remain, that's `/iwol:summary-and-test <PR>` in a new session for the next group (name it), started in this folder (give its path, from `git rev-parse --show-toplevel`): the branch is checked out here, and any fixes are uncommitted here. Otherwise it's `/iwol:create-pr`, to commit the work and open the PR or push to it. Nothing more unless the user asks for detail.

### Continuing a split test

When the PR's plan still has steps `not run`, carry on with it rather than planning again:

- First look for changes the plan doesn't cover: `git diff <the plan's Head: sha>` and the untracked files as Step 1 lists them: one by one, without the files to leave alone. A file the plan's `Covers uncommitted:` line lists is covered while it's unchanged, so drop it: `git hash-object <file>` still prints its blob, or it's still missing for `deleted`. If it has changed since, only the change since its blob is new: run `git hash-object -w <file>` on its own, then `git diff <its blob> <the new blob>`. A file deleted since, or back after `deleted`, is new as a whole. For a file the line gives no blob, or whose blob git no longer has (`git cat-file -e <blob>` fails: git prunes unreferenced objects after about two weeks), and for one a `fixed in` or `fix tried in` result names that the line doesn't list (as in a plan saved before blobs), compare the file's diff from `Head:` with the plan's steps, and treat what no step covers as new. Plan steps for the new changes with something to see as a new group at the end. Then set `Head:` to the current commit, give each file with new changes its current blob in `Covers uncommitted:`, and save the plan again before going on. Keep the existing steps' numbers: the plan's results refer to them.
- Run the first group with steps `not run`, starting from the first of them. Do each step's recorded setup again, and anything the group's earlier steps set up, yourself where you can.
- Before sending a step, check it against the current code: screens, labels and flows may have changed since the plan was saved, for example by an earlier session's fixes. The plan keeps one line per step, so write each one out in full in the format `test-steps.md` gives.
- Open with one line: the group this session runs, and what the earlier groups found and fixed.

## Non-goals

- Don't test a Beads epic or wave: that's `/iwol:beads-summary-and-test`.
- Don't fix anything until this session's steps are done.
- Don't commit or push, and don't open or merge a PR. That's `/iwol:create-pr`.
- Don't dump the full test plan at once: one step, wait, next step.
- Don't run more than one group of a split test in a session unless the user asks. The next group starts in a new session, from a small context.
- Don't pad the summary. If a one-liner covers a unit, use one line.
