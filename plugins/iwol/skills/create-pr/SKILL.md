---
name: create-pr
description: 'Branch off the base branch, commit the current Claude session''s changes, push, and open a PR against it (draft by default) — or, if the current branch already has an open PR, push follow-up commits to it. Also waits for CI, runs the Claude review loop, marks drafts ready for review, and merges the PR and cleans up its branch the way the repository''s iwol config says. Use when the user asks to "ship this", "open a PR", "create a PR against main", "draft a PR", "add this to the PR", "push more commits to this PR", "mark the PR ready", or "merge the PR", or otherwise wants the session''s changes turned into (or added to, promoted out of draft in, or merged as) a pull request targeting the base branch. For releasing to production (merging the Release PR), use /iwol:create-release instead.'
---

# /iwol:create-pr

Turn this session's changes into one or more PRs against the base branch. It's the only long-lived branch, so every PR targets it.

## Read the config first

Read `.claude/iwol-config.md` before anything else. It holds everything specific to this repository, and every step below looks its values up there. If it's missing, stop and ask the user to run `/iwol:setup`, which creates it. If a line this skill needs is missing from it, as in a config written before that line existed, stop and ask the user to run `/iwol:setup` to add it. Never guess a value.

In this skill, `main` means the base branch the config's Git and GitHub section names, and `origin/main` means that branch on `origin`. The lines it reads:

- **Checks:** `Format:` and `Format check:` (either may be `none`), and the full check and other suites, for targeted tests.
- **Review:** the reviewer and its bot login, when it reviews, how to request a review, and its in-progress, clean, findings, round-counting and failure signals. `Reviewer: none` means there's no review loop.
- **CI:** the workflow file and its gating jobs, `Runs on:` (`hosted`, or `self-hosted:` with how to tell a queued job is waiting for one of the repository's own runners), where recovering a stuck run is documented, and the wait loop. `CI: none` means no workflow runs on PRs.
- **Git and GitHub:** the base branch, `Merge method:` (`merge` with its subject and body format, or `squash` with who deletes the branch), and `PR title format:` (`free` or `conventional commit`).
- **Releases:** `none`, or release-please with its Release PR branch.

Some parts of this skill apply only under a condition, and say so where they start: "**Under `conventional commit`**" (the PR title format), "**Under `squash`**" or "**Under `merge`**" (the merge method), "**Under `self-hosted`**" or "**Under `hosted`**" (where CI runs), and "**With releases**" (Releases isn't `none`). Skip a part whose condition doesn't hold.

## Conventions

### Draft or ready

The user's wording decides. Say which mode you're using before `gh pr create`, so they can correct you.

- **Draft** (`--draft`): "draft a PR", "put up a draft", **and any ambiguous wording** ("ship this"). A draft isn't reviewed and doesn't notify anyone.
- **Ready** (no `--draft`): the user explicitly asked to create or open a PR ("create a PR", "open a PR for this").

### Branch and commit naming

| Category              | Branch prefix | Commit type | Covers                                          |
| --------------------- | ------------- | ----------- | ----------------------------------------------- |
| New feature           | `feat/`       | `feat:`     | New functionality or behavior                   |
| Bug fix               | `fix/`        | `fix:`      | Routine bug fixes                               |
| Urgent production fix | `hotfix/`     | `fix:`      | Critical fixes that need to ship fast           |
| Documentation         | `docs/`       | `docs:`     | README, docs, agent instructions, comment-only  |
| Refactor              | `refactor/`   | `refactor:` | Restructuring with no behavior change           |
| Tests                 | `test/`       | `test:`     | Test-only changes not tied to a feature/fix     |
| Everything else       | `chore/`      | `chore:`    | Deps, config, tooling, cleanup (the fallback)   |

- No other prefixes exist. Anything that doesn't clearly fit is `chore/` — don't ask.
- Branch names are `<prefix>/<short-kebab-description>`, e.g. `fix/transaction-filter-date-range`, `chore/bump-vite`.
- **Under `free`:** the commit type column doesn't apply. Write each commit's subject and the PR title as a plain, specific sentence, in the style `git log origin/main` shows.
- **Under `conventional commit`:**
  - Every commit's subject starts with its type from the table. No other types exist (there's no `hotfix:` type); anything that doesn't clearly fit is `chore:`.
  - The PR title is a conventional commit too. Give it the type of the PR's most significant change (`feat` over `fix` over the rest), whatever the branch prefix, and a `!` (e.g. `feat!:`) for a breaking change. **With releases**, the title becomes release-please's changelog entry for the PR, and a `chore:`, `docs:`, `refactor:` or `test:` title leaves the PR out of the changelog.
  - **Under `squash`**, the PR title is the only message that reaches `main`: the squash commit takes it as its subject (and, **with releases**, release-please's one changelog entry for the PR).
  - Keep `@word` out of PR titles. release-please links every `@word` in the title to that GitHub user's profile, so it becomes a bogus link in the changelog. Rephrase it, e.g. "Claude review".
  - Each commit's type fits that commit, not the branch: a `feat/` branch can hold a `test:` or `chore:` commit.
- End every commit with the `Co-Authored-By:` trailer for the model you're running as, per the session's attribution instructions (e.g. `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`).
- End every PR body with `🤖 Generated with [Claude Code](https://claude.com/claude-code)`.

### CI check

If the config says `CI: none`, there's no CI run to wait for: skip every CI wait and CI condition in this skill, and say so when you report a push or a merge.

CI is the workflow the config's CI section names (`.github/workflows/<workflow file>`). It runs on every push to a PR into `main`, drafts included. `gh pr checks <PR>` reports its gating jobs, the ones the config names. If the workflow cancels a PR's in-progress run when a new push arrives, a new push supersedes the previous run.

**After every push to a PR, wait for its CI run yourself; never ask the user to check CI.** Right after the push, start the config's wait loop (CI section, `Waiting for CI`) in the background (`run_in_background`), exactly as the config gives it, with the config's workflow file. It finds the run for the pushed head and exits when the run completes, or after 20 minutes.

The loop's `sha=$(git rev-parse HEAD)` is the commit you just pushed. When merging, the checkout may be on another branch, so wait on the PR's head instead (the command in the loop's comment). This applies in the Claude desktop app too, whose PR monitor reports a failed run but not a green one, so this wait is how you learn CI passed. When it exits, act on the result without waiting for the user:

- **`success`:** move on to the next step (a re-review in [Claude review loop](#claude-review-loop) step 7, or the merge).
- **`failure`:** read the log (`gh run view <run-id> --log-failed`) and handle it where you are in the flow.
- **`cancelled`:** a newer push superseded the run, if the workflow cancels in-progress runs. If the PR's head has moved, wait on the new head's run instead. If it hasn't, someone cancelled the run, so tell the user.
- **Not finished:** the loop prints each job's status. A gating job still `in_progress` is only slow, so wait again. A gating job still `queued`:
  - **Under `self-hosted`:** it may be waiting for one of the repository's own runners (the run itself can show `in_progress`, because other jobs, such as a job that picks a runner, already ran). Handle it as below, then wait again.
  - **Under `hosted`:** it's only waiting for GitHub's hosted runners. Wait again; re-running would only queue it again. If it's still queued after a second wait, tell the user.
- **`null null null`:** CI never started for that commit, so tell the user.

**Under `self-hosted`**, a gating job still `queued` after the wait may be waiting for one of the repository's own runners; the config's `Runs on:` line says how to tell. Take the job's labels from the queued job itself, where GitHub has already resolved its `runs-on` (the workflow file may hold only an expression, such as one reading a runner-picking job's output):

```
gh api --paginate repos/{owner}/{repo}/actions/runs/<run-id>/jobs --jq '.jobs[] | select(.status == "queued") | "\(.name) \(.labels | join(","))"'
```

A runner can take the job only when it carries all of the job's labels, compared without regard to case, as GitHub does. Check the runners registered to the repository itself, and, when the repository's owner (`<owner>`, from `gh repo view --json owner --jq .owner.login`) is an organisation (`gh api repos/{owner}/{repo} --jq .owner.type` prints `Organization`), the organisation's runners too; a personal account (`User`) has no organisation runners, so check only the repository's. In each command, `<labels>` is the job's labels as a JSON array, such as `["self-hosted","linux"]`. Listing an organisation's runners needs a token that can read its self-hosted runners: an organisation admin's classic token with the `admin:org` scope (a default `gh auth login` doesn't grant it), or a fine-grained token with "Self-hosted runners: Read".

```
gh api --paginate repos/{owner}/{repo}/actions/runners --jq '.runners[] | select((<labels> | map(ascii_downcase)) - [.labels[].name | ascii_downcase] == []) | "\(.name) \(.status) busy=\(.busy)"'
gh api --paginate orgs/<owner>/actions/runners --jq '.runners[] | select((<labels> | map(ascii_downcase)) - [.labels[].name | ascii_downcase] == []) | "\(.name) \(.status) busy=\(.busy)"'
```

- **Any matching runner `online`** in either list, even if the other listing errored: the job is queued behind other work, since a busy runner still counts as online. Wait again; re-running would only queue it again. If it's still queued after a second wait, tell the user.
- **None `online`**, and every listing that applies ran without an error (every matching runner `offline`, or no matching runner listed on two checks a minute apart, since an ephemeral runner is briefly unregistered between jobs): the job waits until a runner is back, or until GitHub cancels it after 24 hours, so don't wait it out. Recover it per the documentation the config's `Recovering a stuck run` line points to (if it says `none`, the steps here are all there is): cancel the run, then re-run all its jobs (`gh run rerun <run-id>`, not `--failed`) so a job that picks a runner, where the repository has one, runs again and can fall back to hosted runners. If it's still stuck after that, tell the user.
- **A listing command errors** and no other list shows a matching runner `online` (a 403 when the token can't read runners): you can't tell a busy runner from an offline one, so don't guess. Tell the user the job is queued and ask whether the runner's machine is up: if it is, wait again; if not, recover the run as above. Never suggest widening the `gh` token's scopes, since `admin:org` grants full organisation admin to read one list.

Green CI is policy, not something GitHub enforces: GitHub's Free plan can't make a check required on a private repository, so nothing but this skill stops a merge past a red or missing check. Wait for CI after every push to an open PR, and never merge unless it's green for the exact head being merged. It's in addition to the targeted tests in step 6.

## Opening a PR

1. **Confirm scope.** Run `git status` and `git diff --stat` and list every changed file (staged, unstaged, untracked). On a branch that step 3 opens the PR from as it is, its commits count as changes too: list them as well (`git fetch origin main`, then `git log --oneline origin/main..HEAD`). If nothing changed, say so and stop. Git can't tell which changes this session made, so unless the conversation clearly shows you made all of them (or the user already scoped it), list them and ask before staging. Flag anything that looks like a secret (`.env`, credentials, tokens), and never stage the paths the config's "Files to leave alone" section lists.
2. **Decide on one PR or several**, and state the plan before continuing. Default to one.
   - Split when parts are unrelated and independently mergeable, when one part is urgent or higher-risk (e.g. a `hotfix/`), or when the combined diff would be hard to review.
   - Keep together changes that are related across layers (an endpoint and its UI, a feature and its tests), or where one part depends on another.
   - Ask if it's genuinely ambiguous.
3. **Branch from fresh `main`**, once per PR:

   ```
   git fetch origin main
   git checkout -b <prefix>/<description> origin/main
   ```

   Uncommitted changes carry over; confirm with `git status`. Files for another PR stay uncommitted until that PR's branch. If the changes are already committed on another local branch, cherry-pick or apply them instead.

   **On the branch of a worktree `/iwol:start` made**, open the first PR from it as it is and skip the commands above. It's that branch when all of these hold, and otherwise branch from `origin/main` as above:
   - the current branch is `<prefix>/<description>` with a prefix from [Branch and commit naming](#branch-and-commit-naming);
   - this session is in that branch's own worktree: `git rev-parse --show-toplevel` is `.claude/worktrees/<prefix>-<description>` in the main checkout (see *Names* in `../start/worktrees.md`);
   - no PR, open or closed, has used the branch yet (`gh pr list --head <branch> --state all --json number` prints `[]`).

   Its commits go into the PR along with the uncommitted changes, and an earlier session for the same work may have made some, so list them with step 1's files (`git fetch origin main`, then `git log --oneline origin/main..HEAD`), and ask about any this session didn't make. Keep the branch's name even if another prefix would now fit the changes better: the worktree is named after it. Any further PR from this session branches from `origin/main` as above.

4. **Format** with the config's `Format:` command once, before grouping, then confirm with its `Format check:` command. If the check fails anyway, run the format command again. Its fixes go into the commits of the files they touch. Make a separate format commit (`chore:` **under `conventional commit`**) only if formatting is the PR's only change. Skip whichever of the two commands the config gives as `none`; with both `none`, skip this step.
5. **Commit in logical groups, not one squash.** Reviewers read them one by one (**under `squash`**, the merge squashes them into one commit on `main`; **under `merge`**, they all reach `main`). Each commit is a coherent, revertable unit: a fix stays separate from an unrelated dependency bump, and tests stay with the change they verify. Stage files by name, never with `git add -A` or `git add .`. A single commit is fine when everything is one concern.
6. **Test:** run targeted tests for the changed files, the narrowest commands that cover them, per the config's Checks section and the repository's own test conventions. Run any other suite the Checks section lists whose condition the changes meet. The full check runs in CI once the branch is pushed ([CI check](#ci-check)).
7. **Push and open the PR:**

   ```
   git push -u origin <prefix>/<description>
   gh pr create [--draft] --base main --title '<title>' --body-file - <<'EOF'
   ## Summary
   - <bullet points>

   ## Test plan
   - [ ] <targeted tests run>

   🤖 Generated with [Claude Code](https://claude.com/claude-code)
   EOF
   ```

   `--base main` is passed explicitly even when it's also the repository's default base. Use `--draft` per [Draft or ready](#draft-or-ready), and a title per [Branch and commit naming](#branch-and-commit-naming). The title goes in single quotes, with `'\''` for an apostrophe, and the body through `--body-file -` from the quoted heredoc (`<<'EOF'`), so the shell expands none of the PR's text. Then start the CI wait ([CI check](#ci-check)), for drafts too.

8. **Report** each PR's URL, what it contains, and whether it's a draft or ready. If you split the work, say why. For the next PR, return to step 3. A ready PR goes straight into the [Claude review loop](#claude-review-loop).

## Changes to an already-open PR

If the current branch already has an open PR against `main`, stay on that branch: skip steps 2–3, and `git push` updates the PR.

- **Wait for an explicit go-ahead before committing or pushing any change, review fixes included.** Make the edit in the working tree and tell the user it's waiting locally. Run steps 4–6 and push only after they say "push it", "commit that", "ship it", or "go ahead". Every follow-up commit gets step 4's format check again (unless the config's `Format check:` is `none`), even a one-line review fix.
  - "Add this to the PR" says where the change goes; it isn't a go-ahead. A request that bundles both ("fix X and push it") is.
  - The other exception is a fix for a failing CI check during a merge (see [Merging](#merging)).
- **After each push:**
  - Wait for CI ([CI check](#ci-check)) and report the result. A failure here is reported to the user, not fixed automatically, except during a merge (see [Merging](#merging)).
  - If the reviewer has already reviewed the PR, that review no longer covers the new head. Request a re-review ([Claude review loop](#claude-review-loop) step 7) before any auto-merge.
  - Compare the PR's title and body (`gh pr view <PR> --json title,body`) with its full diff. If they'd now mislead a reviewer, or (**under `conventional commit`**) the title's type no longer fits the most significant change (see [Branch and commit naming](#branch-and-commit-naming)), run `gh pr edit <PR> --title '…' --body-file -` with the new body in a quoted heredoc (`<<'EOF'`), as in step 7, keeping `## Summary`, `## Test plan`, and the footer. Otherwise leave them.

## Marking a draft ready

Only when the user says so ("mark it ready", "take it out of draft"), never just because the work looks done: `gh pr ready <PR>`. If several PRs from this session are open and the user didn't say which, ask. This triggers the reviewer's review when the config's `When it reviews` line says marking ready does.

## Claude review loop

If the config's Review section says `Reviewer: none`, skip this loop: there's no review to wait for, so never auto-merge; merging needs the user's explicit confirmation, after every review thread from people is answered.

Otherwise, the reviewer is the one the config's `Reviewer:` line names, from the workflow file it names (`.github/workflows/<review workflow>`). In the commands below:

- `<bot>` is its REST login from the `Reviewer:` line (of the form `<app>[bot]`, e.g. `my-review-app[bot]`).
- `<summary marker>` is the line that identifies a summary comment, from the `Clean:` and `Counting review rounds:` lines (for the shared Claude review workflow, `Reviewed head: `, followed by the full 40-character SHA of the commit the review started from).
- The in-progress signal, the request-failure reply, the didn't-finish reply and the failed checks are the ones the config's `In progress:`, `Requesting a review:` and `Usage limit:` lines describe.

The `When it reviews:` line says when it reviews by itself: for the shared Claude review workflow, when a PR into `main` opens ready or a draft is marked ready. It reviews with the default model, effort and credential from the `Requesting a review:` line unless the PR description has a line starting `/review [model] [effort] [credential]` (a credential only where that line offers `oauth|api-key` and says the stub passes that credential; otherwise tell the user it can't be picked); don't add one unless the user asks. It doesn't review later pushes on its own; step 7 re-requests it. (If the config says the reviewer reviews every push by itself, or can't be asked, never request a review: wait for its next review of the new head instead.) Tell the user you're watching for its review.

Only the reviewer the config names is part of this loop. Don't wait for, request, or reply to any other review bot that still posts on the PR.

1. **Poll.** Note the UTC time the round started (when you open the PR ready, mark it ready, or request a re-review in step 7: posting the request comment, or the `gh pr ready <PR>` that ends a toggle) as `date -u +%Y-%m-%dT%H:%M:%SZ`. That's GitHub's own format, and the filters below compare it as text, so any other format matches the wrong comments. Also note the PR's head SHA (`gh pr view <PR> --json headRefOid --jq .headRefOid`). Then check:

   ```
   gh api --paginate repos/{owner}/{repo}/issues/<PR>/comments --jq '.[] | select(.user.login=="<bot>" and .created_at > "<round start>") | "\(.created_at) \(.body)"'
   gh api --paginate repos/{owner}/{repo}/pulls/<PR>/comments --jq '.[] | select(.user.login=="<bot>" and .in_reply_to_id == null and .created_at > "<round start>") | "\(.id) \(.path):\(.line)"'
   gh pr checks <PR>
   ```

   `--paginate` is required: our own replies count as review comments, so a few rounds can push the reviewer's newest activity past the first page of 30. It applies `--jq` to each page separately, so filter with `select`, and never compute a `length` or `last` across pages inside `--jq`.

   Only bot activity created after the round started counts: the summary, the failure notices and the inline comments. Earlier rounds' comments stay on the PR, and a re-request on an unchanged head (after a partial or failed review) would otherwise find the old summary or notice at once and end the round. Each round ends with one summary comment, shaped as the config's `Clean:` and `Findings:` lines describe. For the shared Claude review workflow that's a heading (`### No issues found! 🚀`, or `### <n> issue(s) found` followed by one `- ` line per finding), the reviewer's note, one line per check the workflow's checks job ran, `Reviewed with ... effort`, then a last `<summary marker>` line naming the commit the review started from by its full SHA, bare and outside backticks (GitHub renders it as a short commit link). Summaries from before that shape have no heading and start with a findings list or "No issues found.". That footer line is what identifies a summary, so never match on the heading to find one. Compare its full SHA with the PR's `headRefOid`. If it names an older head because the PR was pushed while it ran, the round is over but doesn't cover the new head: request a re-review (step 7).

   - **No summary for the head yet:** reviewing; keep polling. An automatic or toggle-triggered review shows the pending check the config's `In progress:` line names, and a requested review gets a 👀 on your request comment. The 👀 marks the start and stays after the review ends, so only a summary or a failure notice ends the round, or, for a request that still has no 👀 after a few minutes, what the config's `In progress:` and `Usage limit:` lines say about it (for the shared Claude review workflow with a picker job, the outcome of the request's own run, looked up the way the `Usage limit:` line says).
   - **Summary for the head with no findings** (for the shared Claude review workflow, starting with the heading `### No issues found! 🚀`, or with the plain line `No issues found.` in a summary from before that heading): a clean pass, unless it also says the review was partial (a changed file it couldn't inspect; for the shared Claude review workflow, a line starting `Partial review:`, or in an older summary the reviewer's own words). Then request another review (step 7).
   - **Summary for the head listing issues:** findings. Each one is an inline comment from the bot created since the round started; note the ids. If the summary lists an issue with no inline comment, it's still a finding: answer it in a PR comment. From the PR's 4th summary on, step 2's round limit applies.
   - **The request-failure reply** (for the shared Claude review workflow, a bot comment starting "Claude review request failed:"): your request comment didn't parse. Fix it and post again.
   - **The didn't-finish reply** (for the shared Claude review workflow, a bot comment starting "Claude review didn't finish"), **or a failed review check** the config's `Usage limit:` line names, **or a request's own run** that line counts as failed (for the shared Claude review workflow with a picker job, a request with no 👀 whose run concluded `failure`, `cancelled` or `timed_out`): the run failed, hit its time limit, or was cancelled. First look for the error the `Usage limit:` line says it quotes (for the shared Claude review workflow, after "Claude stopped with an error:" in the reply, or in the red check's `Claude review failed` annotation, which `gh run view <run-id>` lists, with the run found as below):
     - **An error a retry won't fix**, such as a usage limit, a billing error like "Credit balance is too low", or a rejected credential: don't retry. Stop and tell the user, quoting the error, and retry only once they say it's sorted.
     - **No error, or one a retry may fix**, such as an overloaded API: retry once, the way the `Usage limit:` line says:
       - **A failed automatic or toggle-triggered review** (a red review check): re-run the whole run with `gh run rerun <run-id>`, finding the run with `gh run list --workflow <review workflow> --branch <branch>`. Not `--failed`: that keeps the runner picked on the first attempt, so a review that failed because that runner went offline would wait for it again; re-running all jobs lets any runner-picking job pick again. That also clears the red check, which a request comment can't.
       - **A failed requested review:** post the request again (step 7).

       If the retry fails too, stop and give the user the run link.
   - **A request the `Usage limit:` line says to stop on** (for the shared Claude review workflow, a request with no 👀 whose own run ended any other way, or for which no run matches): don't post another request; stop and tell the user, with the link that line names.

   A review takes 5–10 minutes. Poll every ~30 s for the first ~6 minutes, then every 1–2 minutes, ideally in a background loop (`run_in_background`) that exits on a summary, a failure notice, a failed review check, or, once a request has had no 👀 for a few minutes, whatever the `Usage limit:` line says ends it (for the shared Claude review workflow with a picker job, its own run ending, no run matching it, or, without the stub's `run-name:`, the missing 👀 itself). A review started by opening, marking ready or toggling posts no failure notice; a red check is its only sign. If nothing has arrived after ~20 minutes, tell the user instead of polling indefinitely.

   **Clean → merge without waiting for confirmation** (green CI for the head still gates it; see [Merging](#merging)).

2. **Evaluate each finding on its merits.** The reviewer can be wrong, can flag something already handled, or can clash with repository conventions; don't apply a suggestion just because it was made. Tell the user your call on each finding.

   When a finding looks like something an earlier pass should have caught, read that round's execution log before calling it a reviewer miss. Each run uploads the 14-day artifact the config's `Usage limit:` line names (for the shared Claude review workflow, `claude-review-pr<PR>-<reviewed head>-attempt<run attempt>`), holding the reviewer's execution log (every tool call, plus the result block's `permission_denials` list and `total_cost_usd`), the check outputs and the diffs it read (`pr-diff/`). The artifact name is the handle: comment-triggered runs are listed under `main`, not the PR's branch, so filtering runs by branch misses them. List the repository's artifacts and keep the ones for that round's reviewed head (the full SHA from its summary's footer):

   ```
   gh api --paginate 'repos/{owner}/{repo}/actions/artifacts?per_page=100' --jq '.artifacts[] | select(.name | startswith("claude-review-pr<PR>-<reviewed head>")) | "\(.workflow_run.id) \(.name)"'
   gh run download <run-id> -n <name> -D <dir>
   ```

   Each output line gives a workflow run id and the artifact name; download each one you need into its own new directory with the second command.

   **Round limit.** From the PR's 4th summary on (counted as step 7 says), stop here once each finding is evaluated: summarize each round (findings, resolutions, what's still open), and ask the user, for each finding, with your call on it, to **proceed** (fix it, as steps 3–5 say, even where your call was to decline it; once the fix is pushed, step 7 requests one more review), **defer** (reply that it's left for a follow-up and why, resolve it `WONT_FIX`, and list it as a follow-up for the user), or **decline** (reply why, and resolve it as step 5 says). Fix, reply to or resolve none of them until they answer.

3. **Fix the findings you agree with** in the working tree, then wait for the go-ahead to push ([Changes to an already-open PR](#changes-to-an-already-open-pr)).
4. **Reply to every finding in its own thread**, after its fix is pushed, never before. Never combine findings into one PR comment, and never leave one unanswered.
   - **Fixed:** "Fixed in <commit>: <what changed and why>".
   - **Declined:** explain why ("Already handled by X", "Not applicable because Y", "Keeping the convention in <the file the config's `Review rules:` line names>").

   ```
   gh api repos/{owner}/{repo}/pulls/<PR>/comments/<comment-id>/replies -F body=@reply.md
   ```

   - The `<PR>` number is required. `pulls/comments/<id>/replies` doesn't exist and returns a bare 404, even for a valid id.
   - The route is POST-only, so a plain `gh api` GET also returns 404. To check that a comment exists, use `pulls/<PR>/comments`.
   - Write the body to a file (`-F body=@file`); `-f body="..."` breaks on newlines, backticks, and code fences.

5. **Resolve every thread with a reason that matches your reply:**

   ```
   # Thread id: the node whose comments[].databaseId contains the finding's comment id.
   gh api graphql -f query='
   query($owner: String!, $repo: String!, $number: Int!) {
     repository(owner: $owner, name: $repo) {
       pullRequest(number: $number) {
         reviewThreads(first: 100) {
           nodes { id isResolved comments(first: 20) { nodes { databaseId } } }
         }
       }
     }
   }' -f owner=<owner> -f repo=<repo> -F number=<PR>

   gh api graphql -f query='
   mutation($threadId: ID!, $reason: PullRequestReviewThreadResolutionReason!) {
     resolveReviewThread(input: {threadId: $threadId, resolutionReason: $reason}) {
       thread { id isResolved }
     }
   }' -f threadId=<thread-id> -f reason=<REASON>
   ```

   - `ADDRESSED`: you fixed it. Never use this reason when nothing changed.
   - `WONT_FIX`: a valid point you're deliberately not acting on (out of scope, an accepted trade-off).
   - `INVALID` (shown as "Not an Issue"): it doesn't apply, or it's already handled.

6. **All findings declined, nothing pushed?** Once the user agrees with those calls, reply and resolve right away, then **end the loop without a re-review**. The reviewer would re-review the same commit and most likely repeat itself, and a clean pass on unchanged code proves nothing. Merging needs the user's explicit confirmation. A round with at least one pushed fix goes on to step 7.
7. **Request a re-review** once every thread is resolved and the fix is pushed and CI is green for the new head. The CI wait ([CI check](#ci-check)) tells you when; request it then, without asking the user. **Check the stop limits first.** Stop and hand back to the user, with no re-review, if either applies:
   - **The round just handled was all repeats** of already-decided findings; the next round won't differ. A round where the user chose **proceed** at the round limit (step 2) gets its one more review anyway.
   - **The PR already has 4 reviewer summaries** (the first review plus 3 re-reviews). This stop is hard: count, don't estimate. Count the way the config's `Counting review rounds:` line says: list the PR's issue comments, keep the bot's, and count the bodies containing the summary marker:

     ```
     gh api --paginate repos/{owner}/{repo}/issues/<PR>/comments --jq '.[] | select(.user.login=="<bot>" and (.body | contains("<summary marker>"))) | .id' | wc -l
     ```

     Partial reviews count, which errs toward stopping early; failure notices don't match. GitHub's own review count runs high (it records an empty review for every inline comment), so don't use it. When the count reaches 4, stop, unless the user chose **proceed** for a finding at the round limit (step 2): then request the one more review it asks for, and the round limit applies again to that review's summary. A PR that keeps producing new findings needs rescoping, and that's the user's call.

   When stopping, summarize each round (findings, resolutions, what's still open). There's no auto-merge in this case; merging needs explicit confirmation. Otherwise, request the re-review the way the config's `Requesting a review:` line says (for the shared Claude review workflow, a `/review` comment):

   ```
   gh pr comment <PR> --body "/review"
   ```

   - Add a model, effort or credential only if the user asks for them, e.g. `/review fable max` or `/review api-key`, and a credential only where the config's `Requesting a review:` line offers `oauth|api-key` and says the stub passes that credential; otherwise tell the user the credential can't be picked.
   - **A PR that changes the review workflow** (the file the config's `Reviewer:` line names, or a workflow it calls by local path, `./.github/workflows/…`): don't post a request comment. GitHub runs comment-triggered workflows from `main`, so it would review with `main`'s workflow and never exercise the PR's. Toggle the PR to draft and back instead; `ready_for_review` runs the PR's own workflow. GitHub runs no `pull_request` workflows on a PR that conflicts with its base, so check `gh pr view <PR> --json mergeable --jq .mergeable` first:
     - **`MERGEABLE`:** toggle, each as its own command:

       ```
       gh pr ready <PR> --undo
       gh pr ready <PR>
       ```

     - **`CONFLICTING`:** the toggle would start nothing. Tell the user `main` has to be merged into the branch first; that's a push, so it needs their go-ahead.
     - **`UNKNOWN`:** GitHub computes it lazily after a push. Check again after a few seconds.

     The PR ends ready, as it was, so this isn't [marking a draft ready](#marking-a-draft-ready). Model, effort and credential come from the description's last `/review` line, as on the first review, so when the user asks for any of them, set that line first (`gh pr edit <PR> --body-file -`, with the whole body in a quoted heredoc, `<<'EOF'`). Name a credential there only where the config's `Requesting a review:` line offers `oauth|api-key` and says the stub passes that credential; otherwise leave it out and tell the user it can't be picked: without the choice a description line naming one doesn't parse, so the review would fall back to all the defaults, and a credential the stub doesn't pass is ignored and the default used. Editing the body starts no review. The line applies to every later toggle too, so once that round's summary arrives, put it back as it was unless the user wants it kept.

   Then poll again (step 1) for a summary that names the new head.
   - **New findings:** handle them with steps 2–6, with step 2's round limit from the PR's 4th summary on. If a finding repeats one that was already decided, reply with a link to the earlier decision and resolve it with the same reason. Then return to step 7, which checks the stop limits before any further request.
   - **Clean:** merge without waiting for confirmation once CI is green for the head, the same as a clean first review.

## Merging

Merge only when:

- the user explicitly confirms ("merge it", "merge PR #39"), or
- the latest review from the reviewer (the first review or a re-review) came back clean **for the current head**: its summary's `<summary marker>` line names the PR's head SHA. No confirmation is needed then, but CI must still be green for that head (step 1).

Everything else waits for the user: open findings, an all-declined round, a stopped loop, or a clean review that predates the current head (request a re-review first).

- Never merge a draft; ask whether to mark it ready.
- If several PRs are open and the user didn't say which, ask.
- **With releases:** the Release PR (head branch: the config's `Release PR branch`) is merged only by `/iwol:create-release`.

1. **Confirm CI is green for the exact commit being merged.** Get the PR's head SHA with `gh pr view <PR> --json headRefOid --jq .headRefOid`, and check that every gating job the config names, as `gh pr checks <PR>` reports it, ran on that SHA (`gh run list --workflow <workflow file> --branch <branch> --json databaseId,headSha,status,conclusion` lists each run's head SHA) and passed. If it's still running or queued, wait for it with the loop in [CI check](#ci-check), with `sha` set to the PR's head. Don't run anything locally in its place, and never merge on a run for an older head.
2. **If it fails,** read the failed run's log (`gh run view <run-id> --log-failed`).
   - **Suspected flake:** re-run the CI run once with `gh run rerun <run-id>`, then return to step 1.
   - **Real failure:** fix it, then commit (normal conventions, with step 4's format and format check) and push **without waiting for a go-ahead**. The merge decision covers what it takes to make this PR mergeable, and nothing unrelated. Return to step 1.
     - For an auto-merge, the push invalidates the clean review, so request a re-review (review loop step 7) and merge only on a clean result for the new head. For a merge the user confirmed, their confirmation covers the fix.
   - **Still failing after 2 fix attempts:** stop and report what you tried, with the latest output.
3. **Once CI is green for the head, merge** by the config's `Merge method:`. Run the merge as a command of its own, never chained (`;`, `&&`, `|`) with the cleanup. `<headRefOid>` in both merge commands is the head SHA step 1 read and checked CI for: `--match-head-commit` makes GitHub refuse the merge if the head has moved since. If it refuses, go back to step 1 for the new head; for an auto-merge, the clean review doesn't cover the new head either, so request a re-review (review loop step 7) and merge only on a clean result for it.

   - **Under `squash`:**

     ```
     gh pr merge <PR> --squash --delete-branch --match-head-commit <headRefOid> --subject '<title> (#<n>)' --body ""
     ```

     Pass `--delete-branch` only when the config says the skill passes it. When it says GitHub deletes the branch on merge, leave it out. When you pass it and `git branch --list <prefix>/<description>` shows the local branch, first run `git fetch origin pull/<PR>/head`, so git has the `<headRefOid>` commit even when the head was pushed from elsewhere or updated on GitHub, then `git log --oneline <headRefOid>..<prefix>/<description>`, each as its own command: `gh` deletes the local branch with `git branch -D`, so a commit made after the PR's last pushed head would be lost, and step 4's check would find the branch already gone. If it prints commits, report them and don't merge with `--delete-branch` until the user decides. An error from either command isn't an empty result: report it, and don't pass `--delete-branch` until the check has run cleanly and printed nothing.

     Read the title in its own call, `gh pr view <PR> --json title --jq .title`, and put it in `--subject` with the PR's number as `<n>`. The subject goes in single quotes, with `'\''` for an apostrophe, as [Opening a PR](#opening-a-pr) step 7 passes the title to `gh pr create`; never put the PR's title inside double quotes (see why under `merge`).

     `--subject` and `--body ""` land the PR as one commit titled with the PR title and number and with a blank body, whatever the repository's squash-merge settings, which can instead take a one-commit PR's commit subject and fill the body with the commit messages. **With releases**, that gives release-please exactly one changelog entry per PR. Never use `--merge` or `--rebase` unless the user asks after you've told them what it does: every commit on the branch lands on `main` (**with releases**, every conventional commit on the branch gets its own changelog entry, and a merge commit's default body repeats the PR title, which release-please lists a second time).

   - **Under `merge`:**

     ```
     SUBJECT=$(gh pr view <PR> --json title,number,headRefName --jq '"<subject format>"') &&
       BODY=$(gh pr view <PR> --json title,number,headRefName --jq '"<body format>"') &&
       [ -n "$SUBJECT" ] &&
       gh pr merge <PR> --merge --match-head-commit <headRefOid> --subject "$SUBJECT" --body "$BODY"
     ```

     Build the subject and body from the format the config's `Merge method:` line gives, written as jq strings: `\(.title)`, `\(.number)` and `\(.headRefName)` for the PR's title, number and branch, `\n` for a line break, `\"` and `\\` for a literal `"` and `\`, and `\u0027` for an apostrophe (the jq program is single-quoted). Never paste the PR's title or any other PR text into the command yourself: inside double quotes the shell runs backticks and `$(…)` and expands `$VAR`, so a title like ``fix: handle `null` cart ids`` would land on `main` as `fix: handle  cart ids`. Run the reads and the merge as one Bash call, since variables don't carry over between calls, so the merge runs only if both reads succeeded and gave a subject; it's still never chained with the cleanup. Never use `--squash` or `--rebase` unless the user asks after you've told them what it does: the branch's commits don't reach `main` as they are.

   Claude Code checks each part of a chained command against the allow rules. `Bash(gh pr *)` covers the merge, but no rule covers the `git` cleanup, so a chained command goes to auto mode's classifier, which has blocked a merge as "Merge Without Review". If the classifier still denies an auto-merge, don't retry it in another form: report the clean review and ask the user to confirm the merge.

   **With releases:** the one exception to the merge command above is release-please's `Release-As` override after a hand-cut hotfix. Only when the user asks for it, put the footer `Release-As: X.Y.Z` in the merge's commit: **under `squash`**, pass `--body "Release-As: X.Y.Z"` in place of `--body ""`, since the footer must be in the squash commit to reach `main`; **under `merge`**, end the merge commit's body with that line by adding `\n\nRelease-As: X.Y.Z` to the end of the body's jq string (X.Y.Z is the version the user gave, never PR text).

   If the merge itself fails (conflicts, branch protection, another required check), report it. Never use `--admin` or force it.

4. **Clean up.** When run from the PR's branch, `gh pr merge --delete-branch` usually does this itself: it switches to `main`, fast-forwards it, and deletes the local branch. Check with `git status -sb` and `git branch --list <prefix>/<description>`, then run only the steps it skipped, each as its own command:

   ```
   git checkout main
   git pull --ff-only origin main
   git branch -D <prefix>/<description>
   ```

   - **Under `squash`**, a squash merge never puts the branch's own commits on `main`, so `git branch -d` refuses to delete it. With `--delete-branch`, the check below already ran before the merge (step 3), since `gh` deletes the local branch with `-D` itself. Otherwise, before `-D`, check that nothing was committed after the PR's last pushed head: run `git fetch origin pull/<PR>/head`, so git has the head commit, then `git log --oneline <headRefOid>..<prefix>/<description>`, each as its own command, with the head from `gh pr view <PR> --json headRefOid --jq .headRefOid`; the log must print nothing. If it prints commits, report them and keep the branch. An error from either command isn't an empty result: report it, and don't run `git branch -D` until the check has run cleanly and printed nothing.
   - **Under `merge`**, the branch's commits are on `main` once it's pulled, so delete it with `git branch -d <prefix>/<description>` in place of `-D`. If git refuses, something was committed after the merged head: report it and keep the branch.
   - **The branch on GitHub:** the merge deletes it when the skill passed `--delete-branch` or the config says GitHub deletes it on merge. Otherwise (as under `merge`), delete it only while the branch on GitHub is still at the PR's merged head, checked against GitHub itself rather than this clone's possibly stale `origin/<prefix>/<description>`. Run it as one Bash call, with the head from `gh pr view <PR> --json headRefOid --jq .headRefOid`:

     ```
     REMOTE=$(git ls-remote --exit-code origin refs/heads/<prefix>/<description>) &&
       [ "${REMOTE%%[[:space:]]*}" = "<headRefOid>" ] &&
       git push origin --delete <prefix>/<description>
     ```

     If it deletes nothing, `git ls-remote --heads origin <prefix>/<description>` says why: no output means the branch is already gone; a SHA other than the head means commits were pushed after the merge, so report them and keep the branch.

   **In a linked worktree** (the two absolute paths `git rev-parse --path-format=absolute --git-dir --git-common-dir` prints differ, which needs git 2.31 or later), skip the local steps above; the branch on GitHub is still handled as its bullet says. `main` is checked out in the main checkout, so `git checkout main` fails, and `gh` skips the local branch with a warning (it still deletes the remote one). Remove the worktree and its branches instead, only if nothing would be lost, and only once the session is done with the worktree: it's on the PR's branch (`git branch --show-current` prints `<prefix>/<description>`) and no other PR this session opened is still open. Otherwise keep the worktree and delete only the PR's branch, if `git branch --list <prefix>/<description>` still shows it: run step 2's log for that branch alone, then, if it's the branch checked out here, `git switch --detach origin/main` (if git refuses because of local changes, keep the branch and report why), and delete it as step 3 says. Tell the user the worktree stays, and why. In the main checkout, run the steps above as usual and never this. Steps 1 and 2 below run inside the worktree, before step 3 leaves it.

   1. **Find the branches:**
      - the PR's branch, if `git branch --list <prefix>/<description>` still shows it;
      - the branch the worktree started on, such as the `claude/<name>` branch the desktop app makes for a session: the `<a>` in the oldest `checkout: moving from <a> to …` line that `git reflog --format=%gs HEAD` prints (the reflog lists newest first, so it's the last such line). Leave it out when it's `main`, a commit SHA, gone, checked out in another worktree (`git worktree list`), or a branch this session opened a PR from.
   2. **Check what removing them would lose,** from the worktree's root (`git rev-parse --show-toplevel`), each as its own command:

      ```
      git fetch origin main
      git status --porcelain
      git log --oneline <base>..<branch>
      ```

      - `git status --porcelain` lists uncommitted changes and untracked files that aren't gitignored. Gitignored files (dependencies, env files, build output) don't count; removing the worktree deletes them.
      - Run the log once per branch from step 1. `<base>` is `origin/main`, except for the PR's branch **under `squash`**: a squash merge never puts the branch's own commits on `main`, so `origin/main..<branch>` would list them all. Use the PR's `headRefOid` there, so the log lists only commits made after its last pushed head, after `git fetch origin pull/<PR>/head` so git has that commit.
      - An error from any of these isn't an empty result: report it and delete nothing.

      If `git status` lists anything, or the log lists commits on the PR's branch, delete nothing: report each file and commit, tell the user the worktree and its branches are still there, and stop. If only the session's branch has commits, keep that branch, report them, and go on without it.
   3. **Remove them** as *Leaving and removing one* in `../start/worktrees.md` says. First, inside the worktree, note its path (`git rev-parse --show-toplevel`), then run each as its own command:

      ```
      git switch --detach origin/main
      git branch -d <branch>...
      ```

      Delete the branches here, not after leaving: once a branch's upstream is gone, `git branch -d` checks it against the current `HEAD`, which is `origin/main` only inside this worktree. **Under `squash`**, delete the PR's branch with `git branch -D`: git never sees a squash-merged branch as merged, and step 2's log stands in for its check. If git refuses to delete a branch, keep it and report why.

      Then call `ExitWorktree` with `action: "keep"` (load it with ToolSearch if it's deferred), never `"remove"`:
      - **If it moved the session** (this session entered the worktree with `EnterWorktree`, as `/iwol:start` does), run `git worktree remove "<path>"` from the folder it's back in. Don't run the local steps above there, even when it's the main checkout: its branch and files are the user's. Tell the user the worktree is gone and the session is back in that folder.
      - **If it reports that no worktree session is active** (the session started in this worktree, as in a desktop app session, or entered it before a restart), run `git worktree remove "$(git rev-parse --show-toplevel)"`. It deletes this session's working directory, so run it last and run nothing in this session after it. Tell the user the worktree is gone and the session can be closed.

      Never add `--force` to `git worktree remove`: without it, git still refuses if anything uncommitted appeared since step 2. If it refuses, keep the worktree, and report what `git -C "<path>" status --short` lists.

   **The PR's branch checked out in a worktree this session isn't in** (`git worktree list` shows `[<prefix>/<description>]` on another folder), seen from the main checkout or from any other worktree: git can't delete a branch that's checked out, and another session may still be working in that folder, so don't remove it unasked. Leave the PR's branch out of the steps above, tell the user where it's checked out, and offer to remove that worktree. On a yes, run steps 1 and 2 of *In a linked worktree* for it with `git -C "<path>"` in place of `git` (for the PR's branch only), then `git worktree remove "<path>"`, then delete the PR's branch with `git branch -D`, which step 2's log has already checked: `-d` would compare it with this folder's `HEAD`, not `origin/main`.

## Guardrails

- **No direct pushes:** never force-push, push to `main`, or skip hooks (`--no-verify`). **With releases**, releases happen only by merging the release-please Release PR via `/iwol:create-release`.
- **No stacked PRs:** every branch comes from `origin/main`, and every PR uses `--base main`.
  - When a PR's branch is deleted at merge (by `--delete-branch` or by GitHub), GitHub closes (not retargets) any PR based on that branch, and a closed PR can't be reopened.
  - Keep sequential work in one PR, or wait for the first PR to merge.
- **Leave other PRs alone:** never repoint an existing PR that targets a different base; branch fresh instead. Never let one PR's branch include another PR's files.
- **Stay within the ask:** invoking this skill is the go-ahead to commit and open the PR. Beyond that, never commit, push, mark ready, or merge without the user, except for:
  - fixing a failing CI check during a merge;
  - auto-merging after a clean review of the current head.
