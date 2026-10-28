---
name: side-quest
description: Do a single task (a bead or a free-form prompt) in its own named worktree off the base branch, commit it to a local branch without pushing, remove the worktree but keep the branch, and tell the user how to pick the branch up. Use when the user runs /iwol:side-quest <bead-id or prompt>.
argument-hint: <bead-id | prompt>
disable-model-invocation: true
---

# /iwol:side-quest

Do one task off to the side: a bead, or a free-form prompt such as "update the docs to mention the new tools". It's done in a worktree of its own, cut from the base branch, and committed to a local branch that's never pushed. Then the worktree is removed, the branch is kept, and the session is back where it started, with its own folder untouched.

Read `.claude/iwol-config.md` first. If it's missing, stop and ask the user to run `/iwol:setup`, which creates it. If a line this skill needs is missing from it, stop and ask the user to run `/iwol:setup` to add it. In these steps, `main` means the base branch the config names, and `origin/main` means that branch on `origin`. The lines it reads: in the Git and GitHub section, the base branch, `Worktrees:`, `Worktree setup:` and `PR title format:`; in Checks, `Full check:` and `Format:`; and the Files to leave alone section.

Read `../start/worktrees.md` before Step 2. It holds the naming rule and the steps for creating, entering, leaving and removing a worktree that every iwol skill follows. `cd` into the worktree isn't enough, since the shell goes back to the session's folder after every command; that's why it enters the worktree with `EnterWorktree`.

## Step 0 — Require an argument

If the argument is empty, stop and ask for a bead id or a prompt. Do nothing else until you have one.

## Step 1 — Work out what the argument is

Don't try to detect bead ids strictly. If the argument is a single token that looks like a bead id, try `bd show <argument>`. If it resolves, it's a bead: its title and description are the task. Otherwise (no match, `bd` isn't there, or the argument is clearly a sentence), the whole argument is the prompt.

## Step 2 — Work out the names

Branch `<prefix>/<description>`, worktree `.claude/worktrees/<prefix>-<description>`, as `/iwol:start` names them, so that `/iwol:start` and `/iwol:create-pr` take the branch afterwards. Pick the prefix from the table in *Branch and commit naming* in `../create-pr/SKILL.md` by what the work is; anything that doesn't clearly fit is `chore`, so don't ask.

- **A prompt:** `<description>` is the prompt shortened to a few words, dropping filler such as "the", "a" and "some", put through the `<description>` rule in `worktrees.md`.
- **A bead:** `<description>` is `<bead-id>-<title>`, where `<title>` is the bead's title put through the same rule. Example: bead `proj-45` "Add the export button" becomes `feat/proj-45-add-the-export-button`.

State the branch and the worktree in one line, so the user can correct them, but don't wait for a yes.

Run `git fetch origin`, then check that the names are free: no branch `<branch>` here or on `origin` (`git branch --list <branch>`, `git ls-remote --heads origin <branch>`), no PR that has used it (`gh pr list --head <branch> --state all --json number` prints `[]`), and no folder at the worktree's path. If any is taken, add a word that tells the two apart, say the new names, and check again.

## Step 3 — Create the worktree and move in

1. Check that `.claude/worktrees/` is git-ignored: `git check-ignore -q --no-index .claude/worktrees/probe` must succeed. If it doesn't, stop and ask the user to run `/iwol:setup`, which adds the rule.
2. Create the worktree as `worktrees.md` *Creating one* says, with `-b <branch> origin/main`, then enter it as *Entering one* says and prepare it there as the config's `Worktree setup:` line says. If setup fails, report the error and stop; the worktree stays for the user to fix or remove.

## Step 4 — Do the task

Do the work inside the worktree, and only the task. If it turns out to need a decision the bead or prompt doesn't settle, ask rather than guess.

## Step 5 — Check and commit

1. Run the config's `Format:` command (skip it when it's `none`), then its `Full check:`, and report the result.
2. **If the check fails,** show what failed and stop. Fix it within the task if the user asks; commit with the check still failing only if the user says so.
3. Stage the task's files by name (never `git add -A` or `git add .`, and never a path in Files to leave alone) and commit on `<branch>`, with a message written as create-pr's *Branch and commit naming* says for the config's `PR title format:`. Never push.

## Step 6 — Leave and remove the worktree

`git status --short` must list nothing, apart from the untracked files Files to leave alone lists. If it lists anything else, stop and show it, and keep the worktree.

Otherwise leave and remove the worktree as `worktrees.md` *Leaving and removing one* says: `ExitWorktree` with `action: "keep"`, then `git worktree remove "<path>"` from the folder the session is back in, never with `--force`. Delete no branch: the branch is what this skill hands over.

## Step 7 — Hand off

Tell the user:

- The branch, and what's on it: `git log --oneline origin/main..<branch>`.
- The Full check result, and, if the commit went in with it failing, that it did.
- Whether the worktree was removed, or why it was kept.
- How to pick the branch up: `/iwol:start <branch>` in a later session gives it a worktree again (its Step 2 offers to pick up a branch that exists but isn't checked out), and `/iwol:create-pr` run there ships it. To look at it or carry on by hand instead, `git switch <branch>` in the main checkout; create-pr opens a PR from the branch's commits only in the worktree `/iwol:start` made for it.

## Non-goals

- Never push, and never open a PR. That's `/iwol:create-pr`.
- Never switch the main checkout's branch or change its files, nor those of the folder the session started in.
- Never rename the session.
- Don't claim, update or close the bead. The hand-off names its branch; the bead's status is the user's call.
- Don't create the worktree with `EnterWorktree`'s `name`; `worktrees.md` says why.
