---
name: beads-pr
description: Take a tested wave to merged pull requests, one PR per bead, then close the wave's sub-epic. Covers splitting the local wave branch into per-bead branches, opening the PRs, the one refresh PR for generated files the config ships once per wave (or, in a wave with one bead PR, that PR carrying them instead), triaging review comments and CI failures across them, sending fixes through beads-implement's fix mode, replying to and resolving every thread, syncing with main, merging, and closing the wave once every bead's PR has merged. Use after /iwol:beads-summary-and-test, or when the user runs /iwol:beads-pr <sub-epic-id> or says things like "open the PRs for this wave", "handle the review", "address the review comments", "is the wave ready to merge", or "merge the wave". Always use this skill for a wave's PRs — do not open, fix, or merge them ad hoc.
---

# Beads PR

Take one tested wave from the local wave branch to merged pull requests, then close the wave. Each bead gets its own PR, because small PRs get thorough reviews. Wave-sized PRs of tens of thousands of lines have needed four to six review rounds each, because no single review covered them. Generated files the config ships once per wave go in one refresh PR after the bead PRs merge; in a wave with one bead PR, that PR carries them, and the wave needs no refresh PR. A wave is finished only when every bead's PR, and its refresh PR if it needs one, has merged and its sub-epic is closed.

Each run moves the wave's PRs forward and then ends. You run it again for the next step — ideally in a new session. Every run rebuilds its picture from git, GitHub and `bd`, so it never needs an earlier session's conversation. That's the point: a review round should start from a small context, not from the 300k–600k tokens an implementation or testing session ends with.

Read `.claude/iwol-config.md` first. It holds everything specific to this repository: check commands, generated files, local services, files to leave alone, the PR reviewer and how it signals, CI, and merge rules. If it's missing, stop and ask the user to run `/iwol:setup`, which creates it. Its CI section names the workflow and its gating jobs, where they run (`Runs on:`, `hosted` or `self-hosted:` with how to tell a queued job is waiting for one of the repository's own runners) and where recovering a stuck run is documented (`Recovering a stuck run:`), or says `CI: none` when no workflow runs on PRs. Under `CI: none` there's no CI to read or wait for: Step 1 reads none, *Stage — Triage* collects no failing checks, *Stage — Merge*'s CI condition doesn't apply, and *Stage — Close the wave* skips its CI read and says so in its report. Its Git and GitHub section says how PRs merge (`Merge method: merge` or `Merge method: squash`) and how they're titled (`PR title format: free` or `PR title format: conventional commit`). If either line is missing, as in a config written before they existed, stop and ask the user to run `/iwol:setup` to add it. Don't assume a method or a format. Below, "the reviewer" means the review bot the config names. In these steps, `main` means the base branch the config names, and `origin/main` means that branch on `origin`.

Two rules define the role. First, **you write no implementation code and dispatch no sub-agents.** Fixes go through beads-implement's fix mode. Your job is committing, splitting, pushing, triage, replies, review requests, merging and closing. Second, **nothing merges without the user's explicit yes in the current run.**

## Branches

- **Wave branch** `feature/<sub-epic-id>-<title>`: local only, **never pushed**. beads-implement creates it, and it holds the whole wave, tested together. Every code change lands here first: the original work and every fix round. Every check and re-test runs here.
- **Bead branches** `feature/<bead-id>-<title>`: one for each bead that changed files. Each is cut from `main` and pushed, and gets one PR against `main`. It carries only that bead's files, copied from the wave branch.
- **Refresh branch** `chore/<sub-epic-id>-refresh-generated-files`: only when the config ships a generated file once per wave. Cut from `main` once every bead's PR has merged, it carries only those files, rebuilt on the merged wave, and is pushed with one PR against `main`.
- **Beads stand alone.** Beads in a wave don't depend on each other and don't share files, so each bead branch works on `main` by itself. CI on its PR checks exactly that.
- **Who owns which file:** each bead's `Files:` comments, written by beads-implement, list the files it owns. Only beads whose PRs haven't merged own files; once a bead's PR merges, its files are on `main` and a later bead, such as a repair bead, can take them over. Two cases need special handling:
  - **Generated files**, marked `(generated)` and listed in the config. Never copy them. The config's Ships column says where each one goes:
    - `with each bead PR`, or no Ships value: rebuild it on the bead branch after copying its other files, with the config's rebuild command. Back on the wave branch, restore its state with the config's restore command for each file you rebuilt.
    - `once per wave`: leave it off bead branches. Don't rebuild it there, and before each push check that the bead branch leaves it as its merge base with `origin/main` has it (`git diff --quiet $(git merge-base HEAD origin/main) HEAD -- <file>`). If it doesn't, as on a PR opened before the file shipped once per wave, put the merge base's version back with `git restore --source=$(git merge-base HEAD origin/main) --staged --worktree -- <file>`, which also removes files the merge base doesn't have. Check the index against the merge base (`git diff --cached --quiet $(git merge-base HEAD origin/main) -- <file>`), then commit it with the rest. The wave's refresh PR ships it (*Stage — Refresh PR*).
      - **A one-bead wave**, one whose child beads include exactly one that changed files, is the exception: its one bead PR has nothing to conflict with, so it carries the file, and the wave needs no refresh PR. On that bead's branch, handle every row that ships once per wave as `with each bead PR` above, whether or not the bead's `Files:` comments list it: rebuild it after copying the bead's files, commit it with them, and restore the wave branch's state afterwards. Skip the merge-base check for that PR. Count the beads afresh each time you apply this bullet. Once a second bead with files exists, such as a repair bead, the wave isn't a one-bead wave any more, and every PR leaves the file out as above. The first bead's PR, if it's still open, gets the merge base's version back on its next push, as a PR opened before the file shipped once per wave does. If it merges with no push in between, it carries the file, which does no harm: no other PR carries it. Whenever a push puts the file on a bead PR or takes it off, add or remove the PR body's `Also carries` line (*Stage — Open the PRs*) to match, with `gh pr edit <n> --body-file <file>` and the body written to a file outside the worktree.
  - **A file listed by two beads**, usually a doc. Give each bead branch only its own hunks, using the beads' descriptions to tell whose is whose. Ask the user if a hunk is unclear.
- **Run every session for a wave in the worktree that holds the wave branch**, `.claude/worktrees/<sub-epic-id>-<title>`, where beads-implement checks it out. Git lets a branch be checked out in only one worktree, so a session elsewhere can't switch to it. Once the sub-epic resolves, enter that worktree as *Entering one* in `../start/worktrees.md` says, unless this session is already there. A wave started before waves had their own worktrees may be in the main checkout; *Entering one* then stops and tells the user to run from there.
- **Fetch first.** Whenever a step uses `origin/main` (merging it, branching from it or comparing with it), run `git fetch origin` immediately before, so it isn't stale.
- **Keep the wave branch current.** After any bead PR merges, and before copying files from the wave branch onto any bead branch, merge `origin/main` into the wave branch. Copying a whole file from a stale wave branch would quietly undo changes `main` made to it since. If the merge brings in changes to any of the wave's files, rebuild generated files and re-run the config's checks before copying anything. If a bead branch has commits that didn't come from the wave branch (pushed by hand), merge them into the wave branch before changing anything. If merging `origin/main` into the wave branch conflicts, under either merge method, follow the conflict rule in *Sync with main*.

### Branch and worktree names

- **Branches:** `feature/<bead-id>-<title>`. For a wave's branch, `<bead-id>` is the sub-epic's ID. The refresh branch is `chore/<sub-epic-id>-refresh-generated-files`, which the wave branch's pattern below doesn't match either.
- **Worktrees:** `<bead-id>-<title>`, under `.claude/worktrees/` (git-ignored in this repo), for any worktree made for a wave or a bead. Work that isn't a wave gets `<prefix>-<description>` from `/iwol:start` (see `../start/worktrees.md`). Worktrees the desktop app makes for its own sessions keep the app's names.
- **`<title>`** is the bead's title in lowercase, with every run of characters other than a–z and 0–9 turned into one hyphen, no hyphen at either end, and cut at a hyphen to 40 characters or fewer. For a sub-epic, leave out the leading `Wave <n> —`.
- **Examples:**
  - wave `acme-v1.6`, "Wave 6 — Group data and balances": branch `feature/acme-v1.6-group-data-and-balances`, worktree `.claude/worktrees/acme-v1.6-group-data-and-balances`;
  - bead `acme-v1.6.2`, "Build the web link session layer: server loading": branch `feature/acme-v1.6.2-build-the-web-link-session-layer-server`.
- **Find branches by ID**, never by working out the title again: a title can change after its branch exists. The wave branch is `git branch --list 'feature/<sub-epic-id>-*'`, which doesn't match its beads' branches; a bead branch is its PR's `headRefName`.

## Step 0 — Require the sub-epic ID (hard stop)

If no bead ID is in the conversation or the command args, stop and ask for it. Don't guess from the branch name or recent PRs.

- **Sub-epic** (a wave) — the normal case.
- **Epic** — stop, list its sub-epics with their status, and ask which wave.
- **Leaf bead** — stop. Runs cover a whole wave; ask for the sub-epic ID.
- **Doesn't resolve** — say so and stop.

Once it resolves to a sub-epic, name the session `[<sub-epic id>] pr-<n>`, where `<n>` numbers this run among the wave's `beads-pr` runs. Count the sub-epic's comments that start `PR session ` (`bd comments <sub-epic-id> --json`): `<n>` is one more. Add `bd comments add <sub-epic-id> "PR session <n>"`, so the next run counts this one, even where the session can't be renamed. Rename it with the Claude desktop app's session-title tool (`set_session_title` with the session `self`; load it with ToolSearch if it's deferred). Where there's no such tool, as in the CLI, skip the rename without mentioning it. If the app asks the user to approve the new title, because they named the session themselves, leave it to them. If this session already ran this skill for the same sub-epic, keep its number: add no comment and don't rename.

Then move into the wave branch's worktree (see *Branches*), before reading any state from git. If the wave branch is already gone, as after a clean-up that stopped partway, enter the wave's worktree if `git worktree list` still shows `.claude/worktrees/<sub-epic-id>-<title>` (match it by `<sub-epic-id>-`), so *Clean up* removes it as this session's own. If neither is there, stay where you are.

## Step 1 — Read the state

Gather everything fresh:

- The sub-epic and its child beads: status and comments (`bd show`, `bd children`, `bd comments <id>`), and each bead's files.
- **Open findings**: `Finding` comments on the wave's beads with no later `Resolved <id>`, `Deferred <id>` or `Declined <id>` comment (see *Finding comments*).
- The sub-epic's latest `Manual test plan` comment. If there is none, testing was skipped. Warn the user and ask whether to go on before opening any PRs. If it splits the test into groups and a group has no step run yet, and none re-tested since (a later `Re-tested` comment), testing isn't finished: name the groups left, suggest finishing them with `/iwol:beads-summary-and-test <sub-epic-id>` first, and ask whether to go on.
- **Re-test list**: steps named in `Re-test needed` comments on the sub-epic, plus steps the test plan marks `blocked` or `not run`. Leave out any step a later `Re-tested` comment covers.
- The wave branch and working tree: current branch, uncommitted changes.
- **Each bead's PR**, from the bead's external ref `gh-<n>`, set when its PR opens.
  - If a bead has no ref, look for a PR on its bead branch (`gh pr list --head <bead-branch> --state all`), in case an earlier run opened one but didn't record it. Record any you find.
  - If a bead's PR has merged but the bead has no `Merged in PR` comment (a run stopped after the merge, or someone merged it on GitHub), add it as the Merge stage would, taking the commit from `gh pr view <n> --json mergeCommit --jq .mergeCommit.oid`. Sync with main and Merge read the commit from that comment.
  - If a bead's PR was closed without merging, ask the user whether to reopen it (`gh pr reopen <n>`) or open a replacement from the same branch, then record the PR you end up with.
- **The wave's refresh PR**, when the config ships a generated file once per wave: the PR the sub-epic's latest `Refresh PR #<n>` comment names, and whether the wave's refresh is done (see *Stage — Refresh PR*). If it merged with no `Refresh merged in PR` comment after it, add that comment as the Merge stage would. If it was closed without merging, ask the user whether to reopen it or open a new one. A one-bead wave usually has no `Refresh PR #<n>` comment: its bead PR carries those files (see *Branches*), so its refresh is done once that PR has merged and the sub-epic records it, and there's no refresh PR to reopen. If the wave is still a one-bead wave, its bead's PR has merged, and the sub-epic has no `Refresh merged in PR #<n>` comment naming that PR (a run stopped between the Merge stage's two comments, or someone merged it on GitHub), add one as the Merge stage would, under the same check that the merge changed those files.

  For each open PR, the refresh PR included, gather:
  - its state, head commit, and whether it conflicts with `main` (GraphQL `mergeable`). `UNKNOWN` means GitHub is still working it out, so read it once more before relying on it;
  - CI on the head, read once with `gh pr checks <n> --json name,state,bucket,link` (nothing to read under `CI: none`). Its `state` tells a `QUEUED` job from an `IN_PROGRESS` one, which the plain output lumps together as pending. A gating job still `QUEUED` is handled as *A queued gating job* below says;
  - its review threads;
  - the reviewer's reviews and signals (see the config), plus any review of its that didn't finish (a failure reply, a failed review check, or a request's own run the config's `Usage limit:` line counts as failed) or message from it saying its usage limit is reached (see *A failed review or the reviewer's usage limit* under *Review requests*);
  - its latest review request, if no summary has come since: whether it shows in progress the way the config's `In progress:` line says (for the shared Claude review workflow, a 👀 on the request comment), its age (the comment's `created_at` against now), and any request-failure reply after it (see *A request that goes nowhere* under *Review requests*);
  - its **review round**: the number of rounds the reviewer has finished on it so far, counted the way the config's `Counting review rounds:` line says. For the shared Claude review workflow, that's the bot's summary comments, the ones with a `Reviewed head: ` line, never GitHub's review count, which records an empty review there for every inline comment. A PR's "<n>th summary" below means its <n>th round counted this way.

Then work through these in order, doing every one that applies. One run can move different PRs through different steps.

1. **Sub-epic closed:** if any of the wave's branches or worktrees are still there, do **Clean up**. Otherwise say the wave is already done and stop.
2. **The wave branch has no commits of its own yet** (the first run): commit the wave's work on it as `Wave <n>: <summary>`, with a body bullet per bead.
   - Stage only the files in the beads' `Files:` comments, and check the staged diff for keys or tokens.
   - Leave unrelated files alone, including the always-untracked files the config lists.
   - Ask the user about any changed file that no bead lists.

   This commit stays local. Doing it first means a fix round's changes can be told apart from the wave's.

   On later runs, uncommitted changes to tracked files aren't the wave's original work; they're left over from an interrupted fix round. Show them to the user and ask what to do with them before anything else.
3. **Open findings on beads that aren't `blocked`:** do the **Fix round**. Blocked beads wait for the user (see *Blocked beads* below).
4. **A bead that changed files has no PR yet** (and isn't `blocked`): do **Open the PRs** for those beads. This covers the first run, an attempt that stopped partway, and a blocked bead that has since been sorted out. If no PRs existed before this step, end the run; otherwise carry on with the existing PRs.
5. **A bead PR is missing a change the wave branch has.** For each open bead PR, run `git diff origin/<bead-branch> <wave-branch> -- <the bead's own files>`, leaving out generated files. For a file shared by hunk, check instead that the bead branch has each of this bead's hunks exactly as the wave branch has them. Any difference means a fix never reached the PR, for example because a push failed after fix mode had already closed the bead. Do the per-bead part of **Fix round** step 6 for it: copy, rebuild the generated files that ship with each bead PR, commit, push, and request a review.
6. **A PR conflicts with `main`, a generated file on it is stale after another bead merged, or its bead has a `Waits for <bead-id>` comment and the PR's branch doesn't have that bead's merge yet (its `Merged in PR` comment names the commit):** do **Sync with main** for it. For the refresh PR, sync as *Stage — Refresh PR* says.
7. **A PR has a thread that is unresolved or has no outcome reply, unanswered feedback outside threads, or failing CI on the head:** do **Triage**, covering every such PR together. Then do the **Fix round**.
8. **A PR needs a review request:** it was pushed after it opened with no request since, or its latest summary is on the head and says the review was partial, triage left nothing to push to it, and nothing has been requested since that summary. Request a review, if the reviewer needs requests (see *Catch missed requests* and *A partial review* under *Review requests*). For a PR whose latest request has no summary yet, do what *A request that goes nowhere* there says instead.
9. **A PR is clean:** do **Merge**.
10. **Every bead's PR has merged, the config ships a generated file once per wave, the wave's refresh isn't done, and no refresh PR is open:** do **Refresh PR**.
11. **Every bead's PR has merged, and the wave's refresh is done or the config ships no generated file once per wave:** do **Close the wave**.

Items 6–9 cover an open refresh PR too, with the differences *Stage — Refresh PR* lists.

**Blocked beads:** if a child bead is `blocked`, report it with its comment, and any changes set aside for it (`git stash list`). Its work failed checks, so it gets no PR until the user resolves it, and the wave can't close until then. Carry on with the other beads.

**A queued gating job:** when one of the config's gating jobs on a PR's head, or on `main` in *Stage — Close the wave*, is still `queued`, read its run once: on a PR, the check's `link` names the run (`…/actions/runs/<run-id>/job/…`), and on `main`, *Close the wave*'s `gh run list` does. `gh run view <run-id> --json startedAt,jobs --jq '.startedAt, (.jobs[] | "\(.name): \(.status)")'` gives when it started and its jobs. `startedAt` is the latest attempt's start, so a re-run starts the clock again.

- **Under `hosted`, or a run that started 10 minutes ago or less:** the job is only waiting. Report it as "CI queued" and don't wait for it; the next run reads it again.
- **Under `self-hosted`, and a run that started more than 10 minutes ago:** the job may be waiting for one of the repository's own runners that isn't coming. Run the diagnosis in `../create-pr/queued-ci.md` once, taking its steps for a skill that reads once, with the config's `Runs on:` line for how to tell and its `Recovering a stuck run:` line for where recovering one is documented:
  - **A matching runner is online:** report "CI queued behind other work" and don't wait for it.
  - **Every matching runner listed is `offline`, and every listing ran:** with the user's OK, recover the run as *Recovering the run* there says: cancel it, then re-run all its jobs. The next run reads the re-run.
  - **A listing errors, or no matching runner is listed at all:** ask the user whether the runner's machine is up. If it is, report the job as queued and don't wait for it; if not, recover the run with their OK, as above.

End with one line per PR: its number, its bead (or "refresh"), and where it stands, for example "waiting on review", "requested, not picked up yet", "CI running", "CI queued", "CI queued behind other work", "reviewer usage limit", "waiting for <bead-id> to merge", "check broken by #<n> on main", "ready to merge" or "merged".

**Do not wait, poll, sleep or schedule a wake-up** for the reviewer or CI. Report what's pending and tell the user to run `/iwol:beads-pr <id>` again, in a new session, once it has landed.

## Stage — Triage

Collect, across all the wave's open PRs:
- every review thread (the reviewer's or a person's) that is unresolved or has no outcome reply yet (see *Every thread gets a reply and a resolution*);
- feedback outside threads that has no outcome reply yet: review bodies and PR conversation comments that raise a problem or ask for a change. That includes a reviewer review body carrying a finding, not just its standard header. Leave out review requests, your own footer-signed comments, and bot notices that ask for nothing;
- every failing CI check on a PR's head, with the failing part of its log (none under `CI: none`). Leave out a check that a `Waits for` comment on the PR's bead covers while the PR of the bead it names hasn't merged.

For each item, check it against the actual code and against the review rules the config names. Note the reviewer's priority label, if it gives one. Also note whether the problem existed before the fixes or was introduced by one; the review rules may ask the reviewer to say which on re-reviews. The bead an item belongs to is the bead whose PR it's on; an item on the refresh PR belongs to no bead, and *Stage — Refresh PR* says how to sort it. Look across the PRs too: the same symptom on several of them, or a cause an item names in a file outside its bead's files, may be one shared cause (see *Shared causes*). Sort each item:

- **Fix** — valid, in this bead's scope, and fixable within this bead's own files (its `Files:` comments, plus new files). If the fix would need another bead's file, or would make this bead's code depend on another bead's code that isn't on `main` yet, say so in the triage. That's an undeclared dependency between sibling PRs, and the owner decides: fix it in both PRs and merge them together, defer it, or decline it. Don't send it to a fix round as an ordinary Fix.
- **Repair** — valid, but its cause is in shared code no bead owns (see *Shared causes*), so a fix in this bead's files could only work around it. Group it with the other items and open findings that share the cause, and offer one repair bead for the group. A failing check whose cause a PR from outside the wave put on `main` goes to **Broken by another PR** first.
- **Defer** — valid, but it belongs to a later bead. Name that bead. A failing check can't go to a later bead: it blocks this PR's merge until it passes, and the later wave can't start until this one closes.
- **Decline** — not valid, or a deliberate choice (for example, something the review rules say is intentional). Give the reason. This applies to review threads only: a failing check blocks the merge until it passes.
- **Re-run** — a failing check caused by flaky infrastructure, a timeout or a cancellation, not by the code, this bead's or `main`'s (see **Broken by another PR**). With the user's OK, re-run the whole run (`gh run rerun <run-id>`, not `--failed`): `--failed` keeps the runner a picker job chose the first time, so a run that failed because that runner went offline would wait for it again, while re-running all jobs lets the picker pick again. If it fails the same way again, sort it again, as a Fix or as **Broken by another PR**. A failed review check from a reviewer that *Review requests* covers isn't one: see *A failed review or the reviewer's usage limit* there.
- **Broken by another PR** — a failing check whose cause is in code a PR from outside the wave changed on `main`, not in this bead's code. A PR's `pull_request` run tests the PR merged with `main` as `main` stood when the run started, unless a job checks out the PR's head alone, so once another PR breaks `main`, every PR's run that starts after that fails the same way. A re-run keeps the original run's commit, so it tests that same `main` again. To sort one, read `main`'s CI once for the whole triage: run `git fetch origin`, then *Stage — Close the wave*'s `gh run list`. Below, the tested `main` is the `main` the PR's run tested, read from the failing job's log (`gh run view --job <job-id> --log`, with the job id from the check's `link`) and from the run's head and start, `<PR head>` and `<createdAt>` (`gh run view <run-id> --json headSha,createdAt`). The checkout step ends with `git log -1 --format=%H` (`--format='%H'` before actions/checkout v4.2.0, which prints the sha in quotes), and the line after it is the full sha of the commit the job checked out. Take the first one in the log, the first checkout's, with `grep -m1 -A1 -E "log -1 --format='?%H'?\$" | tail -1 | grep -oE '[0-9a-f]{40}'`, which matches a Windows runner's `git.exe" log -1 …` too. Go by that commit, not by the first `HEAD is now at` line: on a self-hosted runner that reuses its workspace, the checkout first prints lines naming the previous job's commit. If the commit is `<PR head>`, the job tested the PR's head alone, by sha or by branch, and the tested `main` is `git merge-base <PR head> origin/main`. Otherwise it's the run's test merge commit, and the `HEAD is now at <short sha> Merge <head sha> into <base sha>` line whose `<short sha>` starts it gives the tested `main` as that `<base sha>`. If neither holds, as for a job with no checkout step, the tested `main` is the last commit on `main` from before the run started, `git log --first-parent -1 --format=%H --until=<createdAt> origin/main`, which reaches what that run tested even on a reused branch, after a reopen or after a hand push.
  - **Find the commit.** Take the latest completed run in that list on the tested `main` or an earlier commit (`git merge-base --is-ancestor <that run's headSha> <tested main>` succeeds). If it failed the same way, `main` itself broke: find the commit as that stage's **Broken by another PR** bullet does, with that run as the run that tells. Otherwise only the PR's merge with `main` breaks, between this bead's code and a change `main` gained, such as a function that change renamed that the bead's code still calls. The branch usually took that change in just before the push the run tested, so look at the latest commits up to the tested `main` that changed the files the failure names or the code it calls: `git log --first-parent --format='%H %s' -10 <tested main> -- <files>`. Name the commit's PR with that bullet's `gh api` call, or, for a commit pushed straight to `main`, the commit and its author. A commit that is one of the wave's merges (a bead's `Merged in PR` commit) isn't from another PR: merged wave code no open bead owns is a **Repair**.
  - **Already fixed on `main`:** `git log --first-parent --format='%H %s' <tested main>..origin/main` shows a fix or revert of that commit, and, if `main` itself broke, the list's latest completed run passed. The PR's branch doesn't have that fix yet, so do **Sync with main** on the PR: its push starts a run on the fixed `main`.
  - **Not fixed yet:** when the break is between that PR and the wave's own code (this bead's, or a merged bead's on `main`), first ask the user which side fixes it: the wave, as a **Fix** in this bead's own files or a **Repair** for merged beads' files, or that PR. When it's that PR's to fix:
    - Don't offer a re-run: it fails the same way.
    - Don't create a repair bead or a `Finding` comment: the wave doesn't own that code, so its fix doesn't belong in this wave.
    - Tell the user which PR broke the check (its number, title, link and commit), and that it has to be fixed or reverted on `main`, outside this wave, such as by its author or in a session of its own with `/iwol:create-pr`. The PR stays unmerged until its check passes: once the fix is on `main`, a later run finds it as above and syncs the PR. Don't wait for it.

**Repeats.** A thread that repeats a finding already decided on the same PR, where the code shows nothing new, goes in the list as a repeat, with a link to the earlier decision (its outcome reply) and that decision's bucket. Once the user agrees, reply with a link to the earlier decision and resolve the thread, as the *New findings* bullet under `/iwol:create-pr`'s *Claude review loop* does (see *Every thread gets a reply and a resolution*). A repeat of a Deferred or Declined decision gets that decision's reply and reason. A repeat of a Fixed decision gets `Not changing this: already fixed in <sha> (<link>)`, resolved `INVALID`, since nothing changed for it. The user can still move it to another bucket.

**Round budget.** From a PR's third reviewer summary on (two fix rounds already, counted the way the config's `Counting review rounds:` line says), recommend this split:

- Fix the highest-priority issues (P1, if the reviewer uses P1–P3) and problems that earlier fixes introduced.
- Defer older, lower-priority issues to a later bead, or to a new follow-up bead.

Loops of 4–6 rounds came from fixing everything every time. The user decides, as with all triage.

**Round limit.** From a PR's 4th reviewer summary on (the first review plus 3 re-reviews, counted the way the config's `Counting review rounds:` line says), a summary that still lists findings stops that PR's loop: put none of its findings in Fix on your own. List them with your view of each, and ask the user, for each one, to proceed (a Fix, so one more review after the push), defer (to a later bead, or to a new follow-up bead), or decline. The limit applies again to every later summary. A failing check isn't one of these findings: it still blocks the merge until it passes.

Show the user the triage as one short list grouped by PR: item, priority, before or after the fixes, bucket, one-line reason. List each repair group once, after the PRs: the cause, the shared file, the items it covers on each PR, and the repair bead's labels. List each PR from outside the wave that broke a check once too: its number, title and commit, whether `main` itself is red, and the checks it fails on each PR. Wait for their go-ahead. They may move items between buckets. Deferring and declining are their decisions, not yours.

Then act on it:

- **Fix** items: add a `Finding` comment to the PR's bead. Then do the **Fix round** in this same run.
- **Repair** items: create the repair bead and defer the items to it, replying on each thread and resolving it, as in *Shared causes*. The repair bead goes into the **Fix round** in this same run.
- **Defer** items: add a comment on the later bead starting `Deferred from PR #<n> review:`, written so it stands alone (what to change, why, the thread URL). With the user's OK, you can instead create a follow-up bead in a later open wave. Use `bd create --parent <that sub-epic> --no-inherit-labels` with `agent:`, `effort:` and kind labels, and check that it shares no files with that wave's beads. Then reply on the thread and resolve it.
- **Decline** items: reply on the thread and resolve it.
- **Re-run** items: re-run each one's whole run, and check the result on the next run.
- **Broken by another PR** items: do **Sync with main** on each PR whose break is already fixed on `main`. Leave the rest as they are, with no re-run, bead or comment: they wait for the fix on `main`.

If triage leaves nothing to fix on a PR, nothing is pushed there and no new review is requested. Once every thread on it has its reply and is resolved, that PR can go to **Merge** on the reviewer's existing review. The exception is a partial review: once its threads are answered, request another review even though nothing was pushed (see *A partial review* under *Review requests*).

### Every thread gets a reply and a resolution

Each review thread on a PR ends with one reply from you that states its outcome, followed by resolving the thread with the reason that matches it, the convention `/iwol:create-pr` uses too (its *Claude review loop*). No exceptions: never resolve a thread without a reply, and never leave a replied-to thread unresolved.

- **Fixed:** `Fixed in <sha>: <what changed>`, resolved `ADDRESSED`. Post it only after the fix is pushed to that PR. A fix that failed verification gets no reply yet, and its thread stays open.
- **Deferred:** `Deferred to <bead-id>: <why it belongs there>`, resolved `WONT_FIX`. Post it as soon as the user agrees to the triage, or to deferring the finding the thread became (see *Finding comments*).
- **Declined:** `Not changing this: <reason>`, citing the owner's decision or the review rule where one applies. Resolve it `WONT_FIX` when it's a valid point deliberately not acted on, such as a choice the review rules say is intentional, and `INVALID` (shown as "Not an Issue") when it doesn't apply or is already handled. Post it as soon as the user agrees to the triage, or to declining the finding the thread became.

Never resolve a thread `ADDRESSED` when nothing changed. End every reply with the footer `_🤖 Addressed by [Claude Code](https://claude.com/claude-code)_`. That footer is how a later run, and the desktop app's Auto-fix, tell an outcome reply apart from other comments, because replies are posted from the user's own account. If a thread was resolved by hand without a reply, it still gets one: ask the user the outcome if it isn't clear.

Feedback outside threads can't be resolved, so it gets one PR comment instead: link each item and state its outcome in the same words (Fixed, Deferred, Declined), ending with the footer. An item counts as answered once a later footer-signed comment links to it.

After replying, read the threads again and confirm that every one is resolved and ends with an outcome reply, and that every item of feedback outside threads has been answered. Fix any you missed before ending the run.

## Stage — Fix round

1. Findings that didn't come through triage (manual test, codebase check) haven't been offered for deferral yet. List them first, and let the user defer or decline any (see *Finding comments*); fix the rest. A `CI, main` finding can't be deferred or declined: `main` is broken and the wave can't close until it's green, so it's fixed in this wave. Group them for shared causes too, and offer a repair bead for each group before running fix mode, instead of sending its findings to each bead (see *Shared causes*). A finding with a `Cause for` comment has been through fix mode already: don't send it back there unless the owner asks. Offer the repair bead, or Triage's undeclared-dependency choice if the cause is in another bead's file.
2. On the wave branch, kept current as described in *Branches*, group the open findings by bead. Run the **beads-implement** skill in this session with the args `fix <bead-id> <bead-id> …`. It reopens each bead and sends its open findings to a **new** sub-agent, which changes only what the findings need and reports causes outside its bead's files instead of working around them. It verifies everything centrally, adds a `Resolved` comment per fixed finding, updates the bead's `Files:` comment, and closes each bead that has no open findings left.
3. Handle what fix mode couldn't finish, then carry on with the beads that closed or got a new `Resolved` comment:
   - **A bead came back `blocked`:** set its partial changes aside so they don't mix with the others' (`git stash push --include-untracked -m "<bead-id> blocked fix" -- <its files, including any it created>`), and report what failed. Without `--include-untracked`, new files the fix created would stay behind.
   - **A shared cause fix mode reported:** its findings are still open and their beads still `in_progress`. Offer the owner the repair bead fix mode proposed, as in *Shared causes*. On a yes, create it, defer the findings to it, and run fix mode once more for just the repair bead before going on. If the owner would rather defer or decline the findings, record that as in *Finding comments*, including the answer on the thread or comment each one came from. Either way, close each bead that has nothing else open.
   - **A finding fix mode wouldn't take, or left open,** because it needs another bead's file: bring it to the user as Triage's undeclared-dependency case. To fix it in both PRs, give each bead a finding for its own part, and close the original with `Deferred <finding id>: split into <new finding ids>` so only the new findings stay open. The new finding on the original's bead keeps its `Context:`, so the thread or comment it came from is answered once that part lands. The other bead's finding gets `Context: split from <bead-id> (<finding id>)` instead, so nothing on its PR answers an item on the original's PR. If the original's bead has no part of its own, answer the item as a deferral to the other bead (see *Finding comments*). If one PR needs the other's code, add `Waits for <bead-id>: needs <what>` on the dependent PR's bead, naming the other bead. That one merges first, and Step 1 item 6 then does **Sync with main** on the dependent PR so its CI and review run again with the prerequisite on `main`; merge it once they pass. Merging the first PR doesn't re-run the second one's checks.
4. If any bead closed, or a bead still open got a new `Resolved` comment this round, commit on the wave branch as one commit. Stage only the files those beads' fixes changed (a bead still open has only verified fixes to commit: fix mode leaves its other findings untouched), and check the staged diff for keys or tokens. Subject: `Fix <source> findings in wave <n>: <short list>`.
5. Work out which manual test steps the fixes touch. Use each finding's `Re-test:` line and the beads named on each step of the `Manual test plan` comment. If any are touched, add a `Re-test needed after <sha>: steps <list>` comment on the sub-epic.
6. For each bead whose files changed that has an open PR:
   - Switch to its bead branch. If `origin/main` has moved on since the branch was cut, merge it in first, so the files you copy don't bring `main`'s newer content in as part of this PR. Then copy those files from the wave branch, and rebuild the generated files that ship with each bead PR (see *Branches*).
   - Commit with the subject `Address <source> on PR #<n>: <short list>`, where `<source>` is, for example, `second review`, `manual re-test` or `CI failure`. Add a bullet per fix in the body and the session's commit attribution line. Push.
   - If a changed file belongs to another bead, stop and ask the user. Triage and fix mode should have caught it, and that fix would have to go into the other bead's PR too.
   - Reply `Fixed in <sha>: …` on each fixed thread and resolve it, then confirm every thread on the PR is resolved and has its reply. The exception is a thread behind a finding still open on the bead: it gets no reply and stays unresolved until its fix lands.
   - Request a new review on each PR you pushed (see *Review requests*).

   Switch back to the wave branch and restore its state (see *Branches*).

7. Carry on with the rest of Step 1's list. Beads without a PR, such as before the first PRs open or a new repair bead, go on to **Open the PRs**. PRs this round pushed to wait for the reviewer; other PRs can still move on in this run.

## Stage — Re-test (before opening the PRs, and before merging)

If the re-test list isn't empty, run it on the wave branch, kept current, because a step can depend on more than one bead. The list covers steps the fixes touched, and steps that were blocked or not run during testing. Walk the user through those steps exactly as `/iwol:beads-summary-and-test` does. Take each step from the `Manual test plan` comment, send one at a time labelled `Re-test 1 of 3 — <title>`, and wait for pass or fail. Before sending each step, do the setup its line records (`setup …`). If the step relies on something earlier steps in its group created, such as data the user added, and it's gone (for example after a database reset), recreate it first: yourself with SQL or seed data where you can, or by walking the user through those earlier steps again. The comment keeps only one line per step, so write each step out in full in that skill's format: one numbered list with each action and its expected result, labels taken from the code, commands in `bash` blocks and URLs as links. If the list runs to more than about 10 steps, split it into groups of related steps the way that skill splits a long test, and re-test one group per run.

- All pass: add a `Re-tested <date>: steps <list> pass` comment on the sub-epic. If steps are left in other groups, end the run; the next run, in a new session, re-tests the next group. Otherwise carry on.
- Any fail: add a `Finding` comment for it on the step's bead, reopen the bead, and end the run. The next run starts with a fix round.

## Stage — Open the PRs

1. Re-test first, if the re-test list isn't empty.
2. Make sure the wave's work is committed on the wave branch (Step 1 does this on the first run).
3. Check that the beads' files cover every file the wave branch changes compared with `origin/main`, with each file owned by exactly one bead whose PR hasn't merged. The exceptions are generated files and shared files split by hunk (see *Branches*). Ask the user about any file with no owner.
4. For each bead that changed files and has no PR yet, skipping `blocked` beads. Pick up where an earlier attempt stopped:
   - If its bead branch already exists, reuse it, and push it if it isn't on GitHub yet. Otherwise create it from an up-to-date `origin/main`, after merging `origin/main` into the wave branch (see *Branches*).
   - Before opening a PR, check for one already on the branch (`gh pr list --head <bead-branch> --state all`). If there is one, record it instead of creating another.
   - Copy in its files, or its hunks of shared files, from the wave branch, and rebuild the generated files that ship with each bead PR (see *Branches*).
   - If that changed anything, commit with the bead's title as the subject, a short body, and the commit attribution line. Push.
   - Open its PR against `main`:
     - Title, by the config's `PR title format`:
       - `free`: `Wave <n>: <bead title>`, where `<n>` is the wave the bead runs in. Don't derive it from the bead's ID: a bead moved between waves keeps its old ID.
       - `conventional commit`: `<type>: <summary>`, with the summary taken from the bead's title. Propose the type from the bead's task-kind label: `docs` gives `docs:`, `test-only` gives `test:`, and `code-change` gives `feat:` or `fix:`, judged from the bead's title (`fix:` when it repairs existing behaviour, otherwise `feat:`). If the bead has none of those labels, ask the user for the type. Keep any `@word` out of the title, by dropping the `@` or rewording: release-please builds the changelog from these titles and would turn it into a link to a GitHub profile. Before creating the run's first PR, list the proposed title for each bead and wait for the user to confirm or change each one. Create no PR whose title the user hasn't confirmed, and check any title they write for `@word` too.
     - Body: `Implements bead **<bead-id>** (wave <n> of <epic title>)` and what it does in a few lines. When its commit includes generated files the config ships once per wave (a one-bead wave, see *Branches*), add the line `Also carries <files>, rebuilt with <command>, since this is the wave's only PR.`
     - Add its acceptance criteria and how they were verified (from the bead's close reason), and the manual test results for the steps naming this bead.
     - List any findings deferred to later beads, and end with the session's PR attribution line.
     - Only when the user asked this run for a review model, effort or credential: a `/review` line just before the attribution line (see *Requested review options* under *Review requests*).
   - Record the PR on the bead: `bd update <bead-id> --external-ref gh-<n>`.
5. Switch back to the wave branch and restore its state (see *Branches*).
6. Don't request reviews if the config says the reviewer reviews new PRs by itself; otherwise request one on each new PR (see *Review requests*). If the desktop app offers its Auto-fix for these PRs, don't enable it. CI failures go through triage and fix rounds, and a second fixer would push competing commits.
7. If these were the wave's first PRs, end the run. Tell the user to run `/iwol:beads-pr <id>` in a new session once the reviews are in. Mention that a wave's worth of PRs means as many reviews at once, so the reviewer's usage limit may come up.

## Stage — Sync with main

A bead PR needs a sync when GitHub reports it conflicts with `main`, or when another bead's merge changed a generated file it also carries.

1. Merge `origin/main` into the bead branch. Resolve conflicts: for docs, keep both beads' text; for generated files, take either side, then rebuild them with the config's commands.
2. Commit and push. It's a push, so request a review if the reviewer needs one (see *Review requests*).
3. Merge `origin/main` into the wave branch too, and restore its state (see *Branches*).

**If merging `origin/main` into the wave branch conflicts**, the rule is the same under `Merge method: merge` and `Merge method: squash`. A bead branch is cut from `origin/main` and its files are copied in from the wave branch as new commits, so no wave commit reaches `main` under either method, and the merge compares the same three trees either way. This holds wherever this skill merges `origin/main` into the wave branch. Diagnose each conflicted file with `git log <merge base>..origin/main -- <conflicted file>`, where `<merge base>` is `git merge-base HEAD origin/main`:

- **A generated file from the config**, whatever commits touched it: several beads may carry it, so it's expected to differ between the wave branch and `main`. Take either side, then rebuild it with the config's commands, as step 1 does on bead branches. Then re-run the config's checks, as *Branches* says.
- **Only commits from outside the wave changed the file**, usually another PR on `main` that edited one of the wave's files: resolve the conflict, keeping both sides' intent. Then rebuild generated files and re-run the config's checks, as *Branches* says.
- **Every commit touching the file is one of this wave's merged bead PRs** (the merge or squash commit named in its bead's `Merged in PR #<n> as <sha>` comment, or a commit from that PR's branch): stop and show the conflict (the conflicted files and their hunks) rather than resolving it, then abort the merge (`git merge --abort`) so the wave branch stays as it was. A conflict there means the wave branch and its merged PR really differ, for example a fix that never reached its PR. If any conflicted file is in this case, stop, even if others could be resolved.
- **Both an outside commit and one of this wave's merged bead PRs changed the file:** classify it by hunk. Show its conflicted hunks, and find which commits changed `main`'s side of each with `git blame <merge base>..origin/main -- <conflicted file>` (lines unchanged since the merge base are marked `^`). If every line of a hunk comes from outside commits, resolve that hunk, keeping both sides' intent. If any line of a hunk comes from one of this wave's merged bead PRs, the wave branch and its merged PR differ there: stop and show that hunk, then abort the merge (`git merge --abort`), as for a file only this wave's PRs touched, even if other hunks or files could be resolved. If every hunk was resolved, rebuild generated files and re-run the config's checks, as *Branches* says.

## Stage — Merge

A PR is ready when all of these hold:

- **The reviewer is clean on its head.** Its latest review covers the head: its summary names the PR's `headRefOid` in the marker line the config's `Clean:` line describes (for the shared Claude review workflow, `Reviewed head: <full sha>`), or, for a reviewer whose `Clean:` line describes no such marker, the review's `commit_id` is `headRefOid` or its clean signal came after the head commit. And one of two things holds:
  - That review is the config's clean signal.
  - Or, only if the config says resolved findings count as clean: that review doesn't request changes, and every thread it raised has an outcome reply and is resolved. This happens when triage deferred or declined everything, so nothing was pushed.

  A review that covers an older head ends that round but doesn't cover the new head; *Catch missed requests* under *Review requests* requests a review. A review that says it was partial isn't clean under either condition, even with no findings (see *A partial review* under *Review requests*). A review still in progress isn't clean yet. If the config says "Reviewer: none", skip this condition.
- Every CI check on the head commit passed. If the config says `CI: none`, skip this condition.
- Every review thread on the PR has an outcome reply and is resolved, every item of feedback outside threads has been answered, and its bead has no open findings and is closed.
- It doesn't conflict with `main`.
- Its branch has every change the wave branch has to the bead's own files (the check in Step 1, item 5).
- Its bead has no `Waits for <bead-id>` comment whose bead's PR hasn't merged, or whose merge commit (named in that bead's `Merged in PR #<n> as <sha>` comment) the PR's branch doesn't contain yet. Under `squash`, that commit is the squash commit, which a branch contains once `origin/main` has been merged into it since. Passing CI doesn't settle this: a PR can pass without its prerequisite, for example when its tests mock a runtime dependency. Step 1 item 6 syncs it once the prerequisite has merged.

Run **Re-test** first if the re-test list isn't empty.

Then list every ready PR and ask the user for one explicit yes to merge them. If the config says the reviewer leaves no formal GitHub approval, tell them plainly that the PRs have none, along with anything else the config notes about that.

On a yes, merge them one at a time:

- **Order:** follow the config's merge-order rule, if it has one. Others can go in any order.
- After merging a PR that carries a generated file from the config, don't merge any other PR in this run that carries the same file, even if it doesn't conflict. Its copy is stale now and may merge cleanly and still be wrong, so it goes through **Sync with main** first.
- After merging a PR, don't merge any other PR in this run whose bead has a `Waits for` comment naming the merged PR's bead. Its CI and review never ran with the prerequisite on `main`, so it goes through **Sync with main** first.
- Before each merge, check again that the PR doesn't conflict with `main`. Stop at the first one that does; the next run syncs it.
- Merge the way the config's `Merge method` says, pinned to the head you checked. Never use a rebase merge.
  - `merge`: a merge commit, with the config's subject and body format. Never paste PR text, such as its title, into a shell string: backticks, `$VAR` and `$(…)` in it would be expanded. Fill `$SUBJECT` and `$BODY` with `gh pr view <n> --json title,number,headRefName --jq '<the format as a jq string>'`, taking the PR's fields from the JSON (`\(.title)`, `\(.number)`, `\(.headRefName)`), and pass them as `--subject "$SUBJECT" --body "$BODY"`. Run the reads and the merge in one Bash call, since variables don't carry over between calls, and chain them so the merge runs only if both reads succeeded and the subject isn't empty: `SUBJECT=$(…) && BODY=$(…) && [ -n "$SUBJECT" ] && gh pr merge …` (see the command list below). Otherwise a failed read leaves the subject empty, and GitHub falls back to its default subject, which lands on `main` for good.
  - `squash`: a squash, with the subject the PR's title and number. Never paste the title into a shell string: backticks, `$VAR` and `$(…)` in it would be expanded, and the mangled subject would land on `main` for good. Read it into a variable in the same Bash call as the merge, which also reads it just before merging, and chain them so the merge runs only if the read succeeded and the subject isn't empty:

    ```bash
    SUBJECT=$(gh pr view <n> --json title,number --jq '"\(.title) (#\(.number))"') &&
      [ -n "$SUBJECT" ] &&
      gh pr merge <n> --squash --match-head-commit <full sha> --subject "$SUBJECT" --body ""
    ```

    Setting the subject keeps the squash commit's subject the PR's title: for a one-commit PR, GitHub would otherwise take that commit's subject. `--body ""` leaves its body blank: otherwise the repository's squash settings can fill the body with the PR's commit messages or its description, which then land on `main` for good. Where release-please builds the changelog, that gives it exactly one entry per PR. Add `--delete-branch` only if the config says the skill deletes branches, not if it says GitHub deletes them on merge. gh's `--delete-branch` also force-deletes the local bead branch, so before passing it, check that the local branch, if it's still there, holds nothing beyond the head you checked: `git log <full sha>..<bead-branch>` prints nothing. If it prints commits, stop and show them.
- Record it on the bead: `bd comments add <bead-id> "Merged in PR #<n> as <merge sha>"`. `<merge sha>` is the commit the merge made on `main`: the merge commit under `merge`, the squash commit under `squash`. Either way, `gh pr view <n> --json mergeCommit --jq .mergeCommit.oid` gives it. Record the refresh PR on the sub-epic instead: `bd comments add <sub-epic-id> "Refresh merged in PR #<n> as <merge sha>"`. When the config ships a generated file once per wave and the wave is still a one-bead wave (see *Branches*), check that the merge changed one of those files: run `git fetch origin`, then `git diff --quiet <merge sha>^ <merge sha> -- <files>`, which fails when it did. If it did, the bead's PR carried their rebuild: after its `Merged in PR` comment, also add `bd comments add <sub-epic-id> "Refresh merged in PR #<n> as <merge sha>"` with the same number and sha, which makes the wave's refresh done (see *Stage — Refresh PR*). If it didn't, as for a PR opened before the files shipped this way or a rebuild that had nothing to change, add nothing: the wave's refresh isn't done, and *Stage — Refresh PR* checks `main` for it.
- Never enable auto-merge. The config's `Auto-merge: never` says the same.

Afterwards, merge `origin/main` into the wave branch. If every bead's PR has now merged, go straight on to Step 1's items 10 and 11 (**Refresh PR**, then **Close the wave**).

## Stage — Refresh PR

Only when the config's Generated files table ships a file `once per wave`. Bead PRs leave those files out (see *Branches*); one refresh PR, cut from `main` once every bead's PR has merged, rebuilds them on the merged wave and ships them, so the bead PRs never conflict over them. It carries nothing else. A wave with one bead PR needs none: that PR carries the files (see *Branches*), and a merge that changed them records the refresh, so the wave's refresh is done and Step 1 goes straight to **Close the wave**, unless a repair bead's PR merges after it. A merge that didn't change them records nothing, so this stage still runs, and ends in `Refresh not needed` when `main` already has them current. Whenever this stage switches back to the wave branch, restore its state with the config's restore command for each file it rebuilt.

The sub-epic records it: `Refresh PR #<n>` when it opens, and `Refresh merged in PR #<n> as <merge sha>` when it merges. **The wave's refresh is done** once the latest `Refresh merged in PR #<n> as <sha>` or `Refresh not needed at <sha>` comment names a commit that contains every bead's `Merged in PR` commit (`git merge-base --is-ancestor <bead's merge sha> <sha>` for each). A bead PR that merged after it, such as a repair bead's from *Close the wave*, needs another refresh PR.

**Opening it**, once every bead's PR has merged, the wave's refresh isn't done and no refresh PR is open:

1. Run `git fetch origin`. In the wave's worktree, create the refresh branch from `origin/main`, or, if it's still there from an earlier attempt or an earlier refresh, switch to it and merge `origin/main` into it. A conflict in its files is expected once `main` has changed them since: take `main`'s side with `git checkout --no-overlay --theirs -- <files>`, then `git add -A -- <files>`, and commit the merge, since step 2 rebuilds them anyway. `--no-overlay` also removes files `main` deleted, where a plain `--theirs` fails. If an open PR is already on it (`gh pr list --head <refresh-branch> --state open`), record it with `Refresh PR #<n>` and handle it as below instead.
2. Run the rebuild command of each row that ships once per wave. If that changed files outside those rows, stop and show them. Stage those files (`git add -- <files>`) and compare them with `main`, not with the branch, which an earlier attempt may have left holding an unpushed or unmerged refresh commit: `git diff --cached --quiet origin/main -- <files>`. If they match `main`, it already has them current: put the branch's files back (`git restore --staged --worktree -- <files>`), add `Refresh not needed at <sha>` on the sub-epic, with `<sha>` the `origin/main` commit you rebuilt on, switch back to the wave branch, and carry on with Step 1's list.
3. If anything is staged (`git diff --cached --quiet` fails), commit those files with the subject `chore: refresh <files> after wave <n>` under `conventional commit`, so it stays out of the changelog, or `Wave <n>: refresh <files>` under `free`. In the body, name the command and the wave's bead PRs, then add the commit attribution line. Push with `-u`, even when there was nothing new to commit: the branch still differs from `main`.
4. Open the PR against `main`, with the commit's subject as its title. Body: `Refreshes <files> for wave <n> of <epic title> (<sub-epic-id>), now that its bead PRs have merged: #<n>, #<n>, …`, the command it ran, and the PR attribution line, with a `/review` line just before it only when the user asked this run for a review model, effort or credential (see *Requested review options* under *Review requests*). Add `Refresh PR #<n>` on the sub-epic. Request a review only if the reviewer doesn't review new PRs by itself (see *Review requests*), and don't enable the desktop app's Auto-fix.
5. Switch back to the wave branch, and end the run. Tell the user to run `/iwol:beads-pr <id>` again, in a new session, once its CI and review are in.

**Moving it on.** Step 1's items 6–9 treat the open refresh PR like a bead PR, except:

- **Sync with main** when it conflicts with `main`, or once a bead's PR merges that its branch doesn't contain: merge `origin/main` into the refresh branch, resolving a conflict in its files as step 1 does, then run step 2's rebuild again, commit, push and request a review (see *Review requests*). Switch back to the wave branch afterwards.
- **Triage:** it has no bead, so its items get no `Finding` comments and never go to fix mode. A Fix item that a fresh rebuild settles, such as stale output, is fixed by syncing as above, then replying `Fixed in <sha>: …` and resolving the thread. An item that needs any other change can't be fixed on this PR: it's a Repair (see *Shared causes*), or a Defer to a later bead. A failing check the wave's merged code causes is a Repair, and the refresh PR syncs once the repair bead's PR merges. One a PR from outside the wave broke goes to Triage's **Broken by another PR**, where the wave's side is a Repair.
- **Merge:** ready under the Merge stage's conditions, except the parts about its bead (its findings and status, matching the wave branch, `Waits for` comments). In their place, every bead's PR has merged and its branch contains each one's `Merged in PR` commit. Record the merge on the sub-epic, as the Merge stage says.

## Stage — Close the wave

1. Confirm that every child bead is closed, every bead with files has a merged PR, and, when the config ships a generated file once per wave, the wave's refresh is done (see *Stage — Refresh PR*).
2. If the config says `CI: none`, skip this step: no workflow checks the merged wave, and the report below says so. Otherwise, find the one CI run that tells whether the merged wave is green on `main`. Each PR's CI ran against an older `main`, so the merged result hasn't been checked yet. **The wave's merges** are every bead's `Merged in PR` commit and, when there is one, the sub-epic's latest `Refresh merged in PR` commit. A run **covers** a merge when its `headSha` contains it (`git merge-base --is-ancestor <merge sha> <headSha>`).

   Read the runs once: run `git fetch origin`, then `gh run list --workflow <workflow file> --branch main --event push --limit 50 --json databaseId,headSha,status,conclusion`, with the CI section's workflow file, so no other workflow's run, such as a release run or a review run for another PR, stands in for CI. `--event push` keeps only runs from pushes to `main`: `--branch` alone also matches a `pull_request` run from a fork's own `main`, whose commit isn't on `main`, and `schedule` or `workflow_dispatch` runs, which say nothing about whether a run is coming for a merge. The list is newest first. If no run in it is on `main`'s head (`git rev-parse origin/main`) and the head landed 10 minutes ago or less (`git log -1 --format=%cI origin/main`), CI may not have started on it: say so and end the run. Older than that, no run is coming for it, and `main`'s earlier commits are older still, so from here on a commit with no run is one the workflow skipped: through a `paths` filter, a skip-CI marker in its commit message, or no trigger on pushes to `main` at all.

   **The run that tells** is then the first run in the list that covers any of the wave's merges. Every push to `main` adds to the same history, so that run is on the latest commit CI ran on since the wave began merging, and it checked together every merge it covers. A merge it doesn't cover landed after it and got no run of its own, or that run would come first in the list: the workflow skipped it, as a `paths` filter may for a docs-only bead or for the refresh PR, which always merges last. An older run, red or green, says nothing the run that tells doesn't: a failure a later run superseded doesn't block the close, and when CI skipped every commit after a red run on the wave's earlier merges, that red run is the run that tells. Then:
   - **No run covers any of the wave's merges:** nothing on `main` has checked the wave, for example because the workflow doesn't run on pushes to `main`, or skipped every one of the wave's merges and every commit since. This is the only case for skipping the CI read. Tell the user, and with their OK skip this step as under `CI: none`, with the same line in the report below; without it, end the run.
   - **The run that tells hasn't completed** (its `status` is `queued`, `in_progress` or anything but `completed`): say so and end the run. A run shows `in_progress` once any of its jobs has started, such as a picker job, even while a gating job waits for a runner, so first read its jobs once (`gh run view <run-id> --json startedAt,jobs`, with the list's `databaseId`). A gating job still `queued` there is handled as *A queued gating job* under *Step 1 — Read the state* says, with this run's `startedAt`.
   - **The run that tells completed with `success`:** the merged wave is green on `main`. Note the wave's merges it doesn't cover, if any, for the report below, and go on to **Clean up**.
   - **The run that tells completed with any other conclusion** (`failure`, `cancelled`, `timed_out`, `startup_failure` and the like): don't close anything, even when it doesn't cover every one of the wave's merges. Show the user the failure and sort it out with them:
     - **Flaky** (infrastructure, a timeout, a cancellation, such as a recovery whose re-run GitHub refused while the cancellation was still finishing): re-run the whole run with their OK (`gh run rerun <run-id>`, not `--failed`, for the reason *Stage — Triage*'s **Re-run** gives), and check again on the next run.
     - **Broken by another PR**: the failure is in code a PR from outside the wave changed, not in the wave's, as when that PR merged after the wave's merges and the run that tells is on its commit or a later one. Find that PR among the commits the run covers, leaving out the wave's merges. `--first-parent` keeps to `main`'s own commits, so under `merge` a bead branch's commits don't show up as outside ones. When an older run in the same list covers every one of the wave's merges that the run that tells covers, and passed, `main` was green with the wave there, so the break is in `git log --first-parent --format='%H %s' <that run's headSha>..<headSha>`; otherwise look at the latest commits that changed the files the failure names, `git log --first-parent --format='%H %s' -10 <headSha> -- <files>`. `gh api 'repos/{owner}/{repo}/commits/<sha>/pulls' --jq '.[] | "#\(.number) \(.title) \(.html_url)"'` names the PR a commit came from. For a commit pushed straight to `main` it prints nothing: name the commit and its author instead (`git log -1 --format='%H %an %s' <sha>`), and handle it as that PR below.

       When the break is between that PR and the wave's own code, such as a function the PR renamed that a bead's code still calls, first ask the user which side fixes it: the wave, with a repair bead whose `Files:` comment lists only the wave's own files, as under **Caused by the wave**, or that PR, as below. When it's that PR's to fix:
       - Don't offer a re-run: the code on `main` is broken, so a re-run fails the same way.
       - Don't create a repair bead or a `Finding` comment: the wave doesn't own that code, so its fix doesn't belong in this wave.
       - Tell the user which PR broke `main` (its number, title, link and commit) and what has to happen before the wave can close: that PR fixed or reverted on `main`, outside this wave, such as by its author or in a session of its own with `/iwol:create-pr`. The push that lands the fix gets a run that covers the wave's merges too; once it passes, the next run of this skill reads it as the run that tells and closes the wave. End the run without waiting for it.
     - **Caused by the wave**, such as two merged beads that only break together: create a repair bead under the sub-epic (`bd create --parent <sub-epic> --no-inherit-labels`, with `agent:`, `effort:` and `code-change` labels). Add a `Finding [CI, main]` comment describing the failure, and a `Files:` comment listing the files the repair needs. Their beads' PRs have merged, so no open PR owns them.
     - Then do a **Fix round** for the repair bead and **Open the PRs** for it. The wave closes once its PR merges, any refresh it then needs has merged (see *Stage — Refresh PR*), and `main` is green.
3. Do **Clean up**. If it stops, end the run without closing the sub-epic. The next run comes back here and the clean-up picks up where it stopped.
4. Close the sub-epic: `bd close <sub-epic> --reason "All bead PRs merged: #<n>, #<n>, …. <one line on verification and manual testing>"`, naming any refresh PRs after the bead PRs (`refresh #<n>`). A `Refresh merged in PR` comment that names a bead PR, as in a one-bead wave, isn't a refresh PR: don't list that PR again.
5. Report briefly:
   - The wave is done, with the PR links.
   - Under `CI: none`, or when the user agreed to skip the CI read because no run on `main` covered any of the wave's merges, that no CI checked the merged wave, so nothing has run the beads together on `main`.
   - When the run that tells didn't cover every one of the wave's merges, the PRs whose merges it left out: the workflow skipped them on `main`, so no CI ran with them there.
   - What the clean-up removed, and anything it left for the user to decide.
   - The findings deferred to later beads, one line each.
   - The next step: `/iwol:beads-implement <next sub-epic id>` in a new session.

   If this was the epic's last open sub-epic, say the whole epic is complete and ask whether to close it.

## Stage — Clean up

Runs once every bead's PR has merged, just before the sub-epic closes. Their work is on `main`, so the clean-up needs no confirmation. It stops and asks at the first thing that doesn't check out. Every step skips what's already done, so a re-run after a stop picks up where it left off; if the wave branch is already gone, skip step 2. Never use `--force` (the one exception is `git branch -D` on the wave branch and the refresh branch in step 5; under `squash`, also on the branches steps 5 and 7 have checked), and never touch `main` or any branch or worktree that isn't this wave's without asking.

1. **Collect the wave's branches.** The wave branch is `git branch --list 'feature/<sub-epic-id>-*'`, and each bead branch is its PR's `headRefName`. Check that every one of those PRs is `MERGED`. Find the refresh branch by its name, not through its PRs, since every refresh PR, and a `Refresh not needed` attempt, uses the one name: `git branch --list 'chore/<sub-epic-id>-refresh-generated-files'`, and on GitHub `git ls-remote --heads origin chore/<sub-epic-id>-refresh-generated-files`. Steps 3 and 4 treat it like a bead branch; steps 5 and 6 delete it by their own checks. Under `squash`, also note each PR's `headRefOid` (`gh pr view <n> --json state,headRefName,headRefOid`) for steps 5 and 6, and expect bead branches that are already gone locally, on GitHub, or both: GitHub, or gh's `--delete-branch`, may have deleted them at the merge. Later steps skip those.
2. **Check nothing was left behind.** Run `git fetch origin --prune`, merge `origin/main` into the wave branch (other PRs may have landed while the wave waited; if it conflicts, follow the conflict rule in *Sync with main*), then run `git diff --stat origin/main <wave-branch>`, leaving out any generated file the config ships once per wave with a `':(exclude)<file>'` pathspec after `-- .`: the refresh PR shipped it, and a rebuild on the wave branch can differ from it once other PRs have landed. It must print nothing. If it lists files, stop and show them: part of the wave never reached a PR.
3. **Step off the branches.**
   - In the wave's worktree, where this session runs once it has entered it: `git switch --detach origin/main`, because `main` is checked out in the main checkout.
   - In the main checkout, only when it holds the wave branch or a bead branch, as for a wave started before waves had their own worktrees: `git switch main && git pull --ff-only`.
   - Leave the branch of any other checkout this session runs in alone, such as a main checkout on other work or the desktop app's worktree for the session.

   If git refuses because of local changes, stop and show them.
4. **Remove the wave's worktrees.** For every worktree checked out on the wave branch or a bead branch (`git worktree list`), other than this session's own, run `git worktree remove <path>`. If git refuses because of uncommitted changes, stop and show `git -C <path> status --short`. Finish with `git worktree prune`.
5. **Delete the local branches.**
   - Bead branches, skipping any already gone locally:
     - `merge`: `git branch -d`. They're merged, so git agrees.
     - `squash`: git never sees a squash-merged branch as merged, so `-d` refuses. First check that the branch holds nothing beyond what merged: `git log <the PR's headRefOid>..<bead-branch>` must print nothing. If it prints commits, stop and show them. Otherwise delete it with `git branch -D <bead-branch>`. If git doesn't have the `headRefOid` commit, fetch it first with `git fetch origin pull/<n>/head`.
   - The wave branch: `git branch -D`. Its commits aren't the ones on `main`, but step 2 proved all its content is there. `-D` is a forced delete, and this is the one place the clean-up allows it under `merge`, only after step 2 has passed.
   - The refresh branch, if it's there, under either merge method. It may hold several refreshes and merges of `origin/main`, and a later PR may have changed its files on `main`, so it passes when either check does:
     - it holds nothing beyond the latest merged refresh PR's head: `git log <that PR's headRefOid>..<refresh-branch>` prints nothing;
     - merging it into `main` would change nothing: `git merge-tree --write-tree origin/main <refresh-branch>` succeeds and prints the same tree as `git rev-parse 'origin/main^{tree}'`.

     Then delete it with `git branch -D`, which the checks stand in for. If neither passes, stop and show `git diff origin/main...<refresh-branch>`.
6. **Delete the bead branches on GitHub:** `git push origin --delete <bead-branch>` for each, skipping any already gone. Under `squash`, first check that `git log <the PR's headRefOid>..origin/<bead-branch>` prints nothing, as in step 5, and stop and show any commits it prints. Delete the refresh branch on GitHub with `git push origin --delete <refresh-branch>`, if it's there, once one of step 5's checks passes for `origin/<refresh-branch>`. A merged PR's branch can be restored from the PR page if it's ever needed.
7. **List leftovers, and ask.** Look for other worktrees whose branch is already merged into `origin/main` and that have no uncommitted changes, for example a session worktree left on one of this wave's commits.
   - `merge`: `git branch --merged origin/main` lists those branches.
   - `squash`: git doesn't see squash-merged work as merged, so check each other worktree's `HEAD` by content instead, detached ones included. It's a leftover when merging it into `main` would change nothing: `git merge-tree --write-tree origin/main <HEAD>` succeeds and prints the same tree as `git rev-parse 'origin/main^{tree}'` (git 2.38 or later).

   They may belong to other sessions, so list them and remove only the ones the user names, with `git worktree remove` and `git branch -d` for their branch (under `squash`, `git branch -D`, which the content check above allows). **This session's own worktree**, which step 4 left alone, goes without asking when it's the wave's (`.claude/worktrees/<sub-epic-id>-<title>`), since it's this wave's; any other, such as a desktop app worktree the session started in, goes only if the user names it. Before removing it, delete the branch it started on, such as the `claude/<name>` branch the desktop app makes for a session: the `<a>` in the oldest `checkout: moving from <a> to …` line that `git reflog --format=%gs HEAD` prints (the last such line), left alone when it's `main`, a commit SHA, gone, checked out in another worktree, or one of this wave's branches, which step 5 handled. Delete it with `git branch -d`, never `-D`: step 3 left the worktree detached at `origin/main`, so git refuses if the branch has commits that aren't on `main`; then keep it and report them. Remove the worktree itself last, as `/iwol:create-pr` does: once the sub-epic is closed, the report given and, on the epic's last wave, the epic closed or left open as the user says (*Close the wave*, steps 4 and 5), leave and remove it as *Leaving and removing one* in `../start/worktrees.md` says. Note its path (`git rev-parse --show-toplevel`) and call `ExitWorktree` with `action: "keep"`.
   - **If that moved the session** (it entered the wave's worktree with `EnterWorktree`), run `git worktree remove "<path>"` from the folder it's back in, and leave that folder's branch and files alone. Tell the user the worktree is gone and the session is back in that folder.
   - **If it reports that no worktree session is active**, run `git worktree remove "$(git rev-parse --show-toplevel)"`. It deletes this session's working directory, so run nothing in this session after it, and tell the user the worktree is gone and the session can be closed.

   Never add `--force`, so git still refuses if anything uncommitted is there.

## Review requests

How to request a review, and when the reviewer reviews by itself, are in the config. Skip this section entirely if the config says "Reviewer: none", or that the reviewer reviews every push by itself or can't be asked; then nothing ever needs a request. Otherwise these rules apply to the common case: a reviewer that reviews a PR once when it opens, and later only on request.

- **Every push to a PR's branch after the PR exists needs one request on that PR.** This covers fix rounds, syncs and any other push. Post the request after the push, and after replying to and resolving the threads it fixes.
- **Never request a review when opening a PR** if the reviewer reviews new PRs by itself; it's already starting one.
- **Don't request again while one is pending.** The config says how the reviewer shows a review in progress.
- **Catch missed requests.** If a PR was pushed to after it opened, and there's no review request since that push (for example, someone pushed by hand), request one at the start of the run. A PR that hasn't changed since it opened needs none; it was reviewed then.
- **A partial review:** a summary on the head that says the review was partial isn't clean, even with no findings. The config's `Clean:` line says how it shows (for the shared Claude review workflow, a line starting `Partial review:`, or in an older summary the reviewer's own words). Triage its findings as usual. If a fix round pushes to the PR, that push's request covers it. If triage leaves nothing to push, request another review once every thread from that summary has its outcome reply and is resolved, even though nothing was pushed, as the *Summary for the head with no findings* bullet under `/iwol:create-pr`'s *Claude review loop* does. From the PR's 4th summary on (see *Round limit* under *Stage — Triage*), ask the user before requesting it.
- **A request that goes nowhere:** Step 1 reads each PR's latest request once. None of these cases retries; a retry happens only under *A failed review or the reviewer's usage limit*.
  - **A request-failure reply** (for the shared Claude review workflow, a bot reply starting "Claude review request failed:", as the config's `Requesting a review:` line says): the request didn't parse and doesn't count, so *Catch missed requests* would request again. A plain request always parses, so the options that request named were wrong (asked for in an earlier run, or typed by hand), or something else broke. Stop and ask the user what to request, quoting the reply's error and its run link, and post nothing on that PR until they say. Then request with the options they give, and don't reuse the failed request's options unless they say so.
  - **No in-progress sign after 5 minutes** (the request comment's `created_at` against now; for the shared Claude review workflow, still no 👀 on it): do what the config's `In progress:` and `Usage limit:` lines say. Where the `Usage limit:` line looks up the request's own run (for the shared Claude review workflow, only with a picker job and the stub's `run-name:`), look it up once by the comment's id, the way that line shows:
    - **Not `completed` yet:** the review is still starting. Report it as pending and leave that PR for the next run.
    - **Concluded `failure`, `cancelled` or `timed_out`:** a review that didn't finish; *A failed review or the reviewer's usage limit* covers it.
    - **Any other conclusion** (`startup_failure`, `skipped`, or `success` with still no 👀), **or no matching run:** stop and tell the user, with the run's link, and post no further request on that PR until they say.

    Where it doesn't (for the shared Claude review workflow, without a picker job or without that `run-name:`), no run can be tied to the comment: stop and tell the user, with the request comment's link, and post no further request until they say.
  - **No in-progress sign yet, under 5 minutes:** report it as "requested, not picked up yet" and leave that PR for the next run. Don't request again; the next run reads it afresh.
- **A failed review or the reviewer's usage limit:** a reply saying the reviewer has hit its limit, or a review that didn't finish, didn't count. A review that didn't finish is a failure reply, a failed review check, or a request's own run that the config's `Usage limit:` line counts as failed (for the shared Claude review workflow with a picker job, a request with no 👀 whose run concluded `failure`, `cancelled` or `timed_out`). If it quotes an error a retry won't fix, such as a usage limit, a billing error like "Credit balance is too low", or a rejected credential (for the shared Claude review workflow, a "Claude review didn't finish" reply that goes on "Claude stopped with an error:", or a red `Claude / review` check's `Claude review failed` annotation, which `gh run view <run-id>` lists), tell the user, quoting the error, and retry, the way the config's `Usage limit:` line says, once they say it's sorted, even when the failure was itself a retry's. A failure that quotes no error, or one a retry may fix, such as an overloaded API, gets one retry the same way. A run doesn't wait for a retry, so record each one as you make it, those the user OKs included: `bd comments add <bead-id> "Review retried on PR #<n> at <head sha>"`, on the sub-epic for the refresh PR. On a head such a comment names, only a failure after the latest one counts: a failure reply posted after it, a red review check (the rerun replaced the one it retried), or the retry request's own run that the `Usage limit:` line counts as failed. A later run that finds such a failure, quoting no error or one a retry may fix, doesn't retry again: it tells the user, with the run link.
- **Requested review options:** only when the user asked this run for a review model, effort or credential, and the config's `Requesting a review:` line takes `/review [model] [effort] …`. Every `/review` comment the run posts names them, and every PR the run opens, bead PRs and the refresh PR alike, gets a line `/review <what they asked for>` on its own just before the attribution line in its description: the first review, which a reviewer that reviews new PRs by itself starts when the PR opens, reads the description's last line starting `/review`, so without the line it uses the config's defaults. Name a credential only where that line offers `oauth|api-key` and says the stub passes that credential; otherwise leave it out and tell the user it can't be picked. Without such a request, add nothing to a request or a description.
- **A PR that changes the review workflow** (the file the config's `Reviewer:` line names, or a workflow that file calls by local path, `./.github/workflows/…`): don't post a request comment. GitHub runs a comment-triggered workflow from `main`, so the review would never exercise the PR's own workflow. Wherever these rules request a review on it, toggle the PR to draft and back instead, checking `mergeable` first, as *A PR that changes the review workflow* under `/iwol:create-pr`'s *Claude review loop* (`../create-pr/SKILL.md`) describes; a PR that conflicts with `main` goes through **Sync with main** first. The toggle stands in for the request comment: a later run counts the PR's latest `ready_for_review` event as its request (`gh api --paginate repos/{owner}/{repo}/issues/<n>/events --jq '.[] | select(.event == "ready_for_review") | .created_at'`), and reads the review's progress and failures as for an automatic review.

## Finding comments

beads-summary-and-test, this skill and beads-implement's fix mode all use one comment format, so any session can tell what's open:

```
Finding [<source>]: <one-line summary>
Where: <file:line, or screen and URL>
Expected: <what should happen>
Actual: <what happens>
Re-test: <the manual step to re-run after the fix, or "none">
Context: <PR thread, review or comment URL, CI run URL, or test session date>
```

`<source>` is one of `manual test, step 3 of 6` (`step 2.3` for a step the test plan numbers `<group>.<step>`), `codebase check`, `review by <login>, PR #<n>` (the reviewer's or a person's), `CI, PR #<n>` or `CI, main`. A finding stays open until a later comment on the same bead starts with `Resolved <finding comment id>`, `Deferred <finding comment id>` or `Declined <finding comment id>`. Use the first 8 characters of the comment's id, from `bd comments <id> --json`.

To defer a finding, with the user's agreement, add `Deferred <finding id>: to <later bead-id>, <reason>` on its bead. `CI, main` findings are the exception and can't be deferred, because the wave can't close while `main` is red. A `CI, PR #<n>` finding can't be deferred to a later wave: its check would stay red, so the PR couldn't merge and the later wave could never start. Within the wave it can go to a repair bead (see *Shared causes*) or be split into per-bead findings (Fix round step 3), and either way its PR's bead gets a `Waits for <bead-id>` comment naming the bead whose PR has to merge first. Then add a comment on the later bead starting `Deferred from <bead-id>:`, written so it stands alone (what to change and why). To decline a finding, also only with the user's agreement, add `Declined <finding id>: <reason>` on its bead, citing the owner's decision. `CI, main` and `CI, PR #<n>` findings can't be declined: a failing check blocks the merge until it passes. If the bead was reopened only for findings that are now all resolved, deferred or declined, close it again.

A Triage **Fix** item becomes a finding whose `Context:` is the URL of the thread, review body or PR comment it came from, and that item stays unanswered until the finding's fix lands. Whenever such a finding is deferred or declined outside Triage (in a fix round, or to a repair bead), also answer its item as soon as the owner agrees, with `Deferred to <bead-id>: …` or `Not changing this: …` (see *Every thread gets a reply and a resolution*): reply on a thread and resolve it, and answer a review body or PR comment in a footer-signed PR comment that links it. Bead comments alone would leave the item unanswered: the PR couldn't merge, and the next run would triage it again and could offer a second repair bead for it. Splitting a finding (Fix round step 3) is the exception when the original's bead keeps a part: the item waits for that part's fix, whose finding keeps its `Context:`.

## Shared causes

Each bead's fix may change only its own files. When findings on several beads come from one cause in shared code (a UI kit component, a data-layer hook, a layout), a fix in each bead can only work around it: sizes estimated from the window, timers that wait out a race, copies of the shared hook. The symptom stays on every other screen that uses that code, and the workarounds have to come out again once the cause is fixed. Fix the cause once instead, in one repair bead.

**Spot them before the fix round.** Group the open findings and triage items that may share a cause:

- the same symptom on several beads, such as a sheet that won't scroll or a value that flashes back as it saves;
- a cause a finding names in a file outside its bead's files, including a `Cause for <finding id>` comment that fix mode left on its bead, or a `Needs outside this bead` comment its worker left during implementation.

For a group whose cause is in a file no bead owns (in no `Files:` comment of a bead whose PR hasn't merged, usually because it's already on `main`), offer the owner one repair bead up front, instead of sending the findings to each bead: the likely cause, the file, the findings it would replace, and its labels. Don't offer fixing each bead in its own files as the recommended choice; that's what produces the workarounds. If the file belongs to another bead, it's Triage's undeclared-dependency case instead.

Fix mode can find shared causes too. Its sub-agents leave those findings open and report the cause, with file:line evidence and the change the file needs. Fix mode records it on the bead as a `Cause for` comment and proposes a repair bead. Offer it to the owner in the same way.

On a yes:

1. Create the repair bead under the wave's sub-epic: `bd create --parent <sub-epic> --no-inherit-labels`, with `agent:`, `effort:` and `code-change` labels. Its description stands alone: the cause with file:line evidence, the change the shared file needs, every screen or bead it shows on (including ones already on `main`), and that the fix goes in the shared file with no per-screen workaround.
2. Add a `Finding` comment per cause. Its `Re-test:` covers the steps of every finding it replaces, and its `Context:` names what it replaces: `moved here from <bead-id> (<finding id>), …` for findings, the thread or comment URL for triage items. It stands in for the `Deferred from` comment.
3. Add a `Files:` comment listing the shared files. Check that none of them is in the `Files:` comments of another bead whose PR hasn't merged. If the repair also needs a change in another bead's file, give that bead a finding for its own part and handle the pair as Triage's undeclared-dependency case.
4. Defer what it replaces. For a finding, add `Deferred <finding id>: to <repair bead-id>, shared cause in <file>` on its bead, and close the bead again if nothing else on it is open. For a triage thread, reply `Deferred to <repair bead-id>: …` on it and resolve it. A finding whose `Context:` is a thread, review or comment URL gets the same answer on that item (see *Finding comments*). Deferring doesn't make a failing check pass, so for a failing CI check, whether a triage item or a `CI, PR #<n>` finding, also add `Waits for <repair bead-id>: <check> fails on this cause` on the PR's bead. That PR waits for the repair bead's PR: once that merges, do **Sync with main** on it so its CI and review run again, and merge it once they pass.

The repair bead then goes through the **Fix round** and **Open the PRs** like any other bead. Its PR fixes the cause for every screen at once.

## Commands that have worked on this repo

```bash
# Review threads, with each thread's comments
gh api graphql -F owner='{owner}' -F name='{repo}' -F n=<n> -f query='query($owner:String!,$name:String!,$n:Int!) { repository(owner:$owner, name:$name) { pullRequest(number:$n) { headRefOid mergeable reviewThreads(first:100) { nodes { id isResolved path line comments(first:20) { nodes { databaseId author { login } body createdAt } } } } } } }'

# Reply to a thread (use the first comment's databaseId), then resolve it with the reason that matches the reply
# (ADDRESSED, WONT_FIX or INVALID; see *Every thread gets a reply and a resolution*). Write the body to a file outside
# the worktree, such as one mktemp makes, and pass it as -F body=@<file>: -f body="..." breaks on newlines, backticks,
# and code fences, and a file left in the worktree shows up as a change no bead lists and stops *Stage — Clean up* from removing it
gh api -X POST repos/{owner}/{repo}/pulls/<n>/comments/<comment id>/replies -F body=@<file>
gh api graphql -f query='mutation($id:ID!,$reason:PullRequestReviewThreadResolutionReason!) { resolveReviewThread(input:{threadId:$id,resolutionReason:$reason}) { thread { isResolved } } }' -f id=<thread id> -f reason=<reason>

# The reviewer's reviews and reactions (its REST login is in the config)
gh api repos/{owner}/{repo}/pulls/<n>/reviews --jq '.[] | select(.user.login == "<reviewer login>") | {submitted_at, commit_id, state}'
gh api repos/{owner}/{repo}/issues/<n>/reactions --jq '.[] | select(.user.login == "<reviewer login>") | {content, created_at}'

# Does a bead branch contain a prerequisite's merge commit? (exit status 0 means yes; fetch first)
git merge-base --is-ancestor <merge sha> origin/<bead-branch>

# Read CI once (never --watch). state tells QUEUED from IN_PROGRESS, which the plain output lumps together as pending
gh pr checks <n> --json name,state,bucket,link

# Copy one bead's files from the wave branch onto its bead branch
git switch <bead-branch> && git checkout <wave-branch> -- <file> <file>   # git rm any the bead deleted

# Merge with a merge commit, pinned to the checked head (subject and body format from the config, written as jq strings
# over the PR's fields, so no PR text goes through the shell). Run it as one chained Bash call: nothing merges if a read fails or the subject is empty
SUBJECT=$(gh pr view <n> --json title,number,headRefName --jq '<subject format as a jq string>') &&
  BODY=$(gh pr view <n> --json title,number,headRefName --jq '<body format as a jq string>') &&
  [ -n "$SUBJECT" ] &&
  gh pr merge <n> --merge --match-head-commit <full sha> --subject "$SUBJECT" --body "$BODY"

# Or squash, pinned to the checked head (Merge method: squash). Add --delete-branch only if the config says the skill deletes branches.
# Run it as one chained Bash call: the title is read just before merging and never goes through the shell, and nothing merges if the read fails or the subject is empty.
# --body "" keeps the body blank: otherwise the repository's squash settings can fill it with the PR's commit messages or its description, which land on main for good
SUBJECT=$(gh pr view <n> --json title,number --jq '"\(.title) (#\(.number))"') &&
  [ -n "$SUBJECT" ] &&
  gh pr merge <n> --squash --match-head-commit <full sha> --subject "$SUBJECT" --body ""

# Under squash: does a bead branch hold anything beyond what merged? (must print nothing before deleting it)
git log <headRefOid>..<bead-branch>

# Under squash: would merging a leftover worktree's HEAD change nothing on main? (same tree means yes)
git merge-tree --write-tree origin/main <HEAD>; git rev-parse 'origin/main^{tree}'
```

## Non-goals

- Don't write implementation code or dispatch sub-agents. Fixes go through `beads-implement fix`.
- Don't resume earlier sub-agents with `SendMessage`. Fix rounds always start new agents.
- Don't push the wave branch, and don't put two beads in one PR.
- Don't use `--force` in the clean-up, apart from `git branch -D` on the wave branch after step 2 has passed, on the refresh branch after one of its checks passed, and, under `squash`, on a bead branch after `git log <headRefOid>..<bead-branch>` printed nothing or on a leftover branch the user named after its content check. Don't delete a branch or worktree that isn't this wave's without the user naming it.
- Don't poll or wait for the reviewer or CI. Report and end the run.
- Don't merge without an explicit yes in the current run, and never enable auto-merge.
- Don't defer or decline a review item without the user's agreement.
- Don't send findings that share a cause in shared code to each bead's fix. Offer the owner one repair bead.
- Don't resolve a thread without an outcome reply, or leave a replied-to thread unresolved.
- Don't edit bead descriptions. Findings, deferrals and re-test records go in comments.
- Don't start the next wave. That's `/iwol:beads-implement`, in a new session.
