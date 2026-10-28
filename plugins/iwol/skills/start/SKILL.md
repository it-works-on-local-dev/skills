---
name: start
description: 'Start work that isn''t a Beads wave in its own named git worktree: work out the branch name `<prefix>/<description>` the way /iwol:create-pr names PR branches, create `.claude/worktrees/<prefix>-<description>` on that branch from the base branch, prepare it the way the repository''s iwol config says, move the session into it, then do the work there and stop at checked, uncommitted changes; when the user asked only for the worktree, it stops there. Given the name of a worktree that already exists, it moves the session back into it and waits. Use when the user runs /iwol:start <description>, or asks to "start work on …", "start a named worktree for …", "set up a worktree for this fix", or "go back to the worktree for …". It replaces the random worktree names the Claude desktop app gives its sessions. For a Beads wave, /iwol:beads-implement makes the wave''s worktree itself.'
---

# /iwol:start

Give work that isn't a Beads wave a worktree named after it, before the work starts, move this session into it, then do the work there. The Claude desktop app makes a worktree for a session before the session starts, with a random name on branch `<branch prefix>/<random name>`, and no skill can rename it. So this skill makes its own: branch `<prefix>/<description>`, the branch `/iwol:create-pr` later opens the PR from, in `.claude/worktrees/<prefix>-<description>`.

It works best from a session started in the main checkout (in the desktop app, with the session's worktree option off). From a session the app started in its own worktree, it still works, but that worktree stays on disk, empty, until the session is archived.

Read `.claude/iwol-config.md` first. It holds everything specific to this repository. If it's missing, stop and ask the user to run `/iwol:setup`, which creates it. If a line this skill needs is missing from it, stop and ask the user to run `/iwol:setup` to add it. In these steps, `main` means the base branch the config names, and `origin/main` means that branch on `origin`. The lines it reads: in the Git and GitHub section, the base branch, `Worktrees:` and `Worktree setup:`; and the Checks section, for the checks in Step 4.

Read `worktrees.md`, next to this file, before Step 1. It holds the naming rule and the steps for creating and entering a worktree that the other iwol skills share.

## Step 1 — Work out the names

The args are a description of the work in words, or a ready-made branch name `<prefix>/<description>`.

- **No args:** ask what the work is, and list the worktrees already under `.claude/worktrees/` (`git worktree list`) in case the user wants one of those. Then wait.
- **A branch name** whose prefix is one of `/iwol:create-pr`'s (see *Branch and commit naming* in `../create-pr/SKILL.md`): use it, with its description put through the `<description>` rule in `worktrees.md` (so `fix/Date_Range` becomes `fix/date-range`).
- **Words:** pick the prefix from that table by what the work is; anything that doesn't clearly fit is `chore`, so don't ask. Shorten the words to a description of a few words, dropping filler such as "the", "a" and "some", then apply the `<description>` rule. "fix the transaction filter's date range" becomes `fix/transaction-filter-date-range`.

The worktree is `.claude/worktrees/<prefix>-<description>`. State the branch and the worktree in one line before creating anything, so the user can correct them. For names worked out from words, wait for a yes: Step 4 starts editing straight after setup, and renaming a branch and moving a worktree with changes in it, with the session inside, isn't cheap. A ready-made branch name needs no yes.

## Step 2 — Resume, or check the names are free

Run `git fetch origin`, then check in this order:

1. **The branch is checked out in a worktree, or it exists here (`git branch --list <branch>`) and its worktree is there on another branch** (`git worktree list --porcelain`): the user is resuming. Enter it as `worktrees.md` *Entering one* says, which stops if the folder isn't under `.claude/worktrees/` and switches a worktree left on another branch back when nothing would be lost. Skip Steps 3 and 4 and report what's there (Step 5): `git status --short` and `git log --oneline origin/main..HEAD`. Then wait for the user: an earlier session started that work, so don't carry it on from a guess.
2. **`.claude/worktrees/<prefix>-<description>` exists** but isn't a worktree in `git worktree list`, or is a worktree on another branch while `<prefix>/<description>` doesn't exist here: the name is taken. Stop, say what's there (for a worktree, the branch it holds), and offer to enter that worktree as it is or to choose another name.
3. **A PR, open or closed, already used the branch name** (`gh pr list --head <branch> --state all --json number,state,url`): `/iwol:create-pr` opens a PR from the session's branch only when no PR has used its name, so it would branch afresh and the names wouldn't match. Add a word that tells the two apart and say the new names. Unless the user asked only for a worktree, wait for a yes on them, since Step 4 starts editing under them; then check them again from 1.
4. **The branch exists, here or on `origin`, but isn't checked out anywhere** (`git branch --list <branch>`, `git ls-remote --heads origin <branch>`): ask whether to pick up that branch in the new worktree or choose another name. To pick it up:
   - **Bring the local branch up to date.** For one only on `origin`, run `git branch <branch> origin/<branch>`. For one here that's behind `origin/<branch>` (`git log --oneline <branch>..origin/<branch>` lists commits), move it up with `git branch -f <branch> origin/<branch>` when `git merge-base --is-ancestor <branch> origin/<branch>` succeeds; if the two have diverged, stop and say so.
   - **It has commits that aren't on `main`** (`git log --oneline origin/main..<branch>`): treat it like a resume once Step 3 has made its worktree. Skip Step 4, report what's there (Step 5) and wait, since an earlier session started that work.
   - **It has none:** move it to the current base with `git branch -f <branch> origin/main` before Step 3, which is safe since it isn't checked out anywhere, so Step 4 builds on today's `main`.

## Step 3 — Create the worktree and move in

1. Check that `.claude/worktrees/` is git-ignored: `git check-ignore -q --no-index .claude/worktrees/probe` must succeed. If it doesn't, stop and ask the user to run `/iwol:setup`, which adds the rule; without it, every file in the worktree shows up as untracked in the main checkout.
2. Note where the session is now: in a linked worktree when the two absolute paths `git rev-parse --path-format=absolute --git-dir --git-common-dir` prints differ, otherwise in the main checkout.
3. Create the worktree as `worktrees.md` *Creating one* says, with `-b <prefix>/<description> origin/main` (or the existing branch from Step 2), then enter it and prepare it as the config's `Worktree setup:` line says. Run the setup after entering, so its commands run in the worktree. If setup fails, report the error and stop; the worktree stays for the user to fix or remove.

## Step 4 — Do the work

If the user asked only for a worktree, such as "set up a worktree for this fix", or to plan or look around in it first, skip this step: report (Step 5) and wait.

Otherwise, do the work in the worktree. What to build comes from the args and from anything this conversation already holds about it, such as a plan, a plan file or earlier messages. If that isn't enough to start on, such as "rework auth" with nothing else, ask what the work should do and wait; the worktree is ready, so the answer goes straight into it. Don't ask about details the code can settle.

- Keep to what was asked. Work found along the way that's outside it goes in the report, not in this change.
- When it's done, run the narrowest checks that cover what changed, per the config's Checks section, and fix what they turn up. The full check runs in CI once `/iwol:create-pr` pushes the branch.
- Stop at uncommitted changes.

## Step 5 — Report

- The worktree's path and branch, and that this session now works there: its commands, file edits and sub-agents all run in the worktree, and the main checkout is untouched.
- What setup ran, or that the config needs none.
- What was built, the checks that ran and their results, and anything left undone or found outside the work. Skip this when Step 4 was skipped.
- **If the session started in the desktop app's own worktree** (a linked worktree in Step 3), one line: that worktree stays on disk, empty, until the session is archived or deleted, and starting the next session in the main checkout avoids it.
- The next steps: `/iwol:summary-and-test` to test the work, then `/iwol:create-pr` to ship it. create-pr opens the PR from this branch and, once the PR has merged, brings the session back to where it started and removes the worktree.
- In a later session, `/iwol:start <prefix>/<description>` moves that session into this worktree.

## Non-goals

- Don't rename, move or remove the desktop app's own worktree for this session, and don't switch the main checkout's branch or change its files.
- Don't rename the session. It keeps the title the app generated, which describes the work; the branch and the worktree are in the report.
- Don't create the worktree with `EnterWorktree`'s `name`; `worktrees.md` says why.
- Don't commit, push or open a PR. That's `/iwol:create-pr`.
- Don't start a Beads wave here. `/iwol:beads-implement` makes the wave's worktree.
