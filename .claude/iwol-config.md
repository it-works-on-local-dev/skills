# iwol config for this repository

## Checks

The orchestrator runs these itself; sub-agents never do.

- **Full check:** `actionlint && claude plugin validate . && claude plugin validate plugins/iwol`
- **Database:** none.
- **Finding every failure:** the three commands run in sequence, so the first failure stops the check; run each one on its own to see all failures.
- **Format:** none.
- **Format check:** none.
- **Post-wave refresh:** none.

## Generated files

None.

## Local services

None.

## Files to leave alone

Always untracked here. Never stage them: none.

## Review

- **Reviewer:** the shared review workflow in `it-works-on-local-dev/code-review` (`.github/workflows/code-review.yml@v2` there), called from this repository's stub `.github/workflows/code-review.yml`, whose job is named `Reviewer`, posting as `iwol-reviewer[bot]` (GraphQL login `iwol-reviewer`). "Reviewer" always means that stub job and the bot; the engine, Claude or Codex, is what runs a review, and shows only in the review check's name, the failure reply and the summary footer.
- **Review workflow ref:** `v2`. The ref the stub calls the workflow at: `v2` moves to each compatible 2.x release of `code-review.yml`; `v1` is frozen at 1.2.1, the last release of the workflow as `claude-code-review.yml`.
- **When it reviews:** by itself when a PR opens ready or is marked ready (not drafts, fork PRs or PRs opened by bots); after that only on request, so every later push needs a request.
- **Requesting a review:** comment a line starting `/review` on the PR, optionally `/review [claude|codex] [model] [effort] [sub|api]` in any order, each optional. A model implies its engine (`gpt-...` is Codex; `opus`, `sonnet`, `haiku`, `fable`, `best` and `claude-...` are Claude); with neither an engine nor a model, the engine is the stub's `default-reviewer`, `claude` here. Defaults: Claude `opus` at `xhigh` on `sub`. `opus` is an alias for the newest Opus model the workflow's Claude Code knows. Codex can't be used here: the stub passes neither of its secrets, so a request naming it, or one of its models, gets a request-failure reply. Efforts are `low`, `medium`, `high`, `xhigh` and `max`. `sub` and `api` name Claude's credentials, `CLAUDE_CODE_OAUTH_TOKEN` and `ANTHROPIC_API_KEY`. The stub passes `CLAUDE_CODE_OAUTH_TOKEN` and `ANTHROPIC_API_KEY`. A request that doesn't parse gets a bot reply starting "Review request failed:" and doesn't count: two engines or models, an engine that contradicts the model, `ultra` on a Claude review, the old words `oauth` and `api-key` (the reply says they are now `sub` and `api`), an engine with neither of its secrets passed (the reply names both), or a credential the stub doesn't pass. An automatic review (a PR opened ready or marked ready) reads the last line starting `/review` in the PR description the same way, engine, model, effort and credential included, except that a description line that doesn't parse, or names an engine with no secret passed, just means all the defaults, and a credential it names that isn't passed is ignored and the engine's default used.
- **In progress:** a running `Reviewer / Claude` check for an automatic review (GitHub's UI shows it as `Code Review / Reviewer / Claude`; match on `Reviewer / Claude`), or a 👀 on the request comment for a requested one. The 👀 stays, so only a later summary, failure reply or withheld notice ends a round; only bot comments newer than the request count. An automatic review also leaves a 👀 on the PR description, which stays from the first review on, so it never marks a review in progress.
- **Clean:** a summary comment from the bot that starts with the heading `### No issues found! 🚀`, then, each after a blank line, the reviewer's short note, one line per check result from the workflow's checks job, and `Reviewed by Claude with ... effort` followed by a last line `Reviewed head: <sha>` naming the PR's head by its full 40-character SHA, bare (no backticks). Summaries from before the engine names have a `Reviewed with ... effort` footer instead, and older ones still, on PRs reviewed before the workflow changed, start with the plain line `No issues found.` (no heading and no note, and a `Requested ... effort` footer); all count as clean. A summary naming an older head, or saying the review was partial, isn't clean: the workflow adds a line starting `Partial review:` when the reviewer didn't cover every changed file, and an older summary says so in the reviewer's own words. With the stub's `approve: true`, a clean review whose checks all passed, run while every review thread the bot started was resolved, also submits a GitHub approval on the reviewed head, and a later review that isn't clean dismisses it; the summary stays the clean signal, and a PR can count as clean with no formal GitHub approval: when it's clean only because its findings were resolved, since the review that raised them gave none and dismissed any earlier one, when its summary lists a check as unavailable, since the workflow approves only when every check passed, or when its review ran while a review thread the bot started was still unresolved, since that review gave none even if the thread is resolved afterwards.
- **Findings:** inline comments from the bot, one per finding. The summary starts with an `### N issue(s) found` heading (`### 1 issue found`, `### 2 issues found`) in place of the clean heading, then lists each finding as one `- ` line; an older summary lists them with no heading, in place of "No issues found.". No priority badges; take priority from the wording. A summary line with no inline comment is still a finding.
- **Counting review rounds:** count the bot's summary comments, the ones with a `Reviewed head: ` line. GitHub records an empty `COMMENTED` review per inline comment, so GitHub's review count overcounts.
- **Resolved findings count as clean:** yes, once every thread from the head's summary has an outcome reply and is resolved, unless that summary says the review was partial.
- **Usage limit:** no message of its own. A failed, timed-out or cancelled review gets a bot comment starting "Review didn't finish" for a requested review, or a red `Reviewer / Claude` check for an automatic one. When Claude's run itself ended in an error, such as a usage limit or "Credit balance is too low", which a retry won't fix, the comment goes on "Claude stopped with an error:" and quotes it, and the check carries it as a `Claude review failed` annotation (`gh run view <run-id>` lists it); without one, the comment says the review may have hit the 60-minute limit or a newer review replaced it. A red `Reviewer / request` check with a skipped review check also means the automatic review failed. A red `Reviewer / checks` stays out of it: the review still runs and reports the checks as unavailable. When the review's summary held a credential-shaped string (a JSON Web Token, or an `sk-`, `ghp_`, `gho_`, `ghu_`, `ghs_`, `ghr_` or `github_pat_` key), the summary isn't posted and the bot comments a notice starting "Review output withheld:" with a run link and no `Reviewed head:` line; it ends the round and isn't clean. A requested review gets it in place of "Review didn't finish", an automatic one gets it plus a red `Reviewer / Claude` check. Each run, a failed one included, uploads the artifact `code-review-pr<PR>-<sha>-attempt<n>` with the check output, the diffs it read and an execution log, kept for 14 days, except when the summary was withheld or one of those files holds a credential-shaped string. Replies from before the engine names start "Claude review request failed:" or "Claude review didn't finish", and their artifacts are named `claude-review-pr<PR>-<sha>-attempt<n>`. Retry an automatic review with `gh run rerun <run-id>` (all jobs, so every job runs again); retry a requested one with a new `/review`.
- **Review rules:** generic rules live in the shared workflow's prompt; this repo has no repo-specific rules.

## CI

- **Workflow:** `ci.yml`, gating job `check`.
- **Runs on:** hosted. CI uses GitHub-hosted runners only, so a queued job is only waiting for GitHub.
- **Recovering a stuck run:** none documented.
- **Waiting for CI:** after a push, a PR skill waits for the run on the pushed commit with this loop, run in the background. It prints `<run id> completed <conclusion>` when the run finishes, or each job's status after 20 minutes; a `null null null` there means no run started for the commit.

```bash
sha=$(git rev-parse HEAD)  # when merging: sha=$(gh pr view <PR> --json headRefOid --jq .headRefOid)
for attempt in $(seq 1 40); do
  run=$(gh run list --workflow ci.yml --commit "$sha" --json databaseId,status,conclusion --jq '.[0] | "\(.databaseId) \(.status) \(.conclusion)"')
  case "$run" in *" completed "*) echo "$run"; exit 0 ;; esac
  sleep 30
done
echo "not finished after 20 minutes: $run"
gh run view "${run%% *}" --json jobs --jq '.jobs[] | "\(.name): \(.status)"'
```

## Git and GitHub

- **Base branch:** `main`. The skills use it wherever they say `main`.
- **Worktrees:** `.claude/worktrees/`, which must be listed in the tracked `.gitignore` so fresh clones ignore it too.
- **Worktree setup:** none. A fresh worktree needs nothing: the checks use only tools installed on the machine.
- **Merge method:** squash. GitHub deletes the branch on merge, so the skills don't pass `--delete-branch`. Merge commits and rebase merges are disabled on GitHub.
- **PR title format:** conventional commit (`fix:` patch, `feat:` minor, `!` major), since release-please builds the changelog from PR titles. Keep `@word` out of PR titles: release-please links every `@word` in a title to that GitHub user's profile, which leaves a bogus link in the changelog.
- **Merge order:** none.
- **Auto-merge:** never.

## Releases

- **Release PR branch:** `release-please--branches--main`
- **Release workflow:** `release.yml`
- **Deploy workflow:** `none`
- **Production version:** `none`
- **Migrations to check before a release:** `none`
- **Rollback:** `none`
