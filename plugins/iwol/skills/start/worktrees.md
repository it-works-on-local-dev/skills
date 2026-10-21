# Named worktrees

The rules every iwol skill follows for the worktrees it creates, enters and removes. A worktree is a second checkout of the repository in its own folder, on its own branch. Git lets a branch be checked out in only one worktree at a time.

`<main checkout>` below is the folder that holds the path `git rev-parse --path-format=absolute --git-common-dir` prints (that path, without its last `/.git`). It's the same from every worktree, and it needs git 2.31 or later.

## Names

Every named worktree lives in `<main checkout>/.claude/worktrees/`, which the config's `Worktrees:` line says is git-ignored.

- **Work that isn't a Beads wave** (`/iwol:start`): branch `<prefix>/<description>`, worktree `.claude/worktrees/<prefix>-<description>`. `<prefix>` is one of the branch prefixes in `/iwol:create-pr`'s *Branch and commit naming* table (`feat`, `fix`, `hotfix`, `docs`, `refactor`, `test` or `chore`), so the worktree's branch becomes the PR's branch. Example: branch `fix/transaction-filter-date-range` in `.claude/worktrees/fix-transaction-filter-date-range`.
- **A Beads wave** (`/iwol:beads-implement`): branch `feature/<sub-epic-id>-<title>`, worktree `.claude/worktrees/<sub-epic-id>-<title>`, as beads-implement's *Branch and worktree names* says.
- **`<description>`** follows the rule for a bead's `<title>`: lowercase, every run of characters other than a–z and 0–9 turned into one hyphen, no hyphen at either end, and cut at a hyphen to 40 characters or fewer.
- **The Claude desktop app's own worktrees,** the ones it makes for a session before the session starts (`<branch prefix>/<random name>`), keep the app's names. No skill renames or moves one, and none removes one mid-session: that breaks the session. The app removes it when the session is archived or deleted.

## Creating one

From any folder of the repository, each as its own command:

```
git fetch origin main
git worktree add "<main checkout>/.claude/worktrees/<name>" -b <branch> origin/main
```

To check out a branch that already exists, leave out `-b` and `origin/main`: `git worktree add "<main checkout>/.claude/worktrees/<name>" <branch>`.

Then enter it (below), and prepare it there as the config's `Worktree setup:` line says, such as installing dependencies or copying env files (nothing when it's `none`). The main checkout's branch and files are never touched.

Never create it with `EnterWorktree`'s `name` instead. That names the branch `worktree-<name>` rather than `<branch>`, and locks the worktree, so a plain `git worktree remove` refuses it.

## Entering one

Moving the session into a worktree is the only way to work there. `cd` doesn't do it: the shell goes back to the session's folder after every command, and sub-agents start in the session's folder, so their edits would land in the wrong checkout.

1. Find where the branch is checked out: in `git worktree list --porcelain`, it's the `worktree <path>` line of the entry with `branch refs/heads/<branch>`.
2. If this session is already there, there's nothing to do.
3. If the path is under `<main checkout>/.claude/worktrees/`, say in one line which folder you're entering, then call `EnterWorktree` with `path` set to that absolute path (load it with ToolSearch if it's deferred). It works from the main checkout and from another worktree, and in the desktop app the diff and file panes follow the session there.
4. **If the branch isn't checked out anywhere,** look for the worktree its name gives (*Names* above) in `git worktree list`. A session may have left it on another branch, as `/iwol:beads-pr` does when it stops while a bead branch is checked out in the wave's worktree.
   - **It's there, and `git branch --list <branch>` shows the branch:** enter it as in 3. If `git status --short` lists nothing, apart from the untracked files the config's Files to leave alone section lists, switch it back with `git switch --no-guess <branch>`. Otherwise stop and show what it lists, and leave the branch as it is.
   - **It's there, but the branch doesn't exist here** (it merged and was deleted, and the worktree moved on to other work): don't switch, and don't create the branch again. Report the branch the worktree holds, and offer to enter it as it is.
   - **It's not there:** the skill creates it on the branch (*Creating one*, without `-b`), when the skill says to.
5. If the branch is checked out in the main checkout or a folder outside `.claude/worktrees/`, or there's no `EnterWorktree` tool, don't move: stop, and tell the user to run the skill from that folder.

Git can't tell whether another session is still working in a worktree, and two sessions editing one folder overwrite each other's changes. Naming the folder before entering gives the user the chance to stop you.

## Leaving and removing one

Once the skill's own checks say nothing would be lost:

1. **Inside the worktree,** read anything the skill needs from it, such as its reflog, and note its path (`git rev-parse --show-toplevel`). Then detach it with `git switch --detach origin/main` and delete the branches the skill names, from there. Deleting them inside matters: `git branch -d` checks a branch against its upstream, or, once that's gone, against the current `HEAD`, which is `origin/main` here but in the folder the session goes back to is whatever that folder has checked out.
2. Call `ExitWorktree` with `action: "keep"`. Never use `"remove"`: it works only on worktrees `EnterWorktree` created with `name` in this same session, it deletes the branch without the skill's checks, and it refuses a squash-merged branch.
   - **If it moved the session** (its result says the session is back in another folder: the main checkout, or the desktop app's worktree the session started in), run `git worktree remove "<path>"` from there. Leave the folder the session is back in as it is: never switch its branch or change its files. The session goes on working.
   - **If it reports that no worktree session is active** (the session started in this worktree, or entered it before a restart), remove the worktree from inside, as the last command the session runs: `git worktree remove "$(git rev-parse --show-toplevel)"`. It deletes the session's working directory, so run nothing after it, and tell the user the session can be closed.

   Never add `--force` to `git worktree remove`: without it, git still refuses if anything uncommitted appeared since the checks. If it refuses, keep the worktree, and report what `git -C "<path>" status --short` lists.
