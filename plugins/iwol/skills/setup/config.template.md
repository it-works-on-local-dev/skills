# iwol config for this repository

<!--
Template for .claude/iwol-config.md. Every skill in the iwol plugin reads that file for everything
specific to a repository: the bead skills (beads-decompose-plan, beads-implement,
beads-summary-and-test, beads-pr) and the PR and release skills.

To adopt the plugin in a repository:
1. Enable it. The machine needs the iwol marketplace registered (see the README of
   it-works-on-local-dev/skills), and the plugin enabled, either through /plugin or by
   committing .claude/settings.json with {"enabledPlugins": {"iwol@iwol": true}}.
2. Start a new Claude Code session there and run /iwol:setup. It writes .claude/iwol-config.md
   from this template (migrating an old .claude/beads-config.md if there is one), enables the
   plugin in .claude/settings.json for everyone who clones the repository, writes the review
   workflow stub, and checks bd, gh, the worktree ignore rule and the review secrets.
   To fill it in by hand instead, save this template as .claude/iwol-config.md, replace every
   <placeholder>, and delete the sections or rows that don't apply. The short names inside the
   Review section's quoted patterns and the CI loop (<PR>, <sha>, <n>, <run-id>) are part of
   the text, not placeholders.
-->

## Checks

The orchestrator runs these itself; sub-agents never do.

- **Full check:** `<command that type-checks, lints, formats and tests everything>`
- **<Other suite, e.g. database>:** `<command>` whenever `<what changed>`.
- **Finding every failure:** `<how to see all failures if the full check stops at the first one, or "n/a">`
- **Format:** `<command that rewrites files into the repository's format, or "none">`
- **Format check:** `<command that only checks the format and changes nothing, or "none">`
- **Post-wave refresh:** `<command to run once after a wave's changes are in and formatted, such as a knowledge-graph refresh, or "none">`

The format and refresh lines run once after changes land, in the order listed, before the checks.

## Generated files

Rebuilt by tooling, never copied between branches, and allowed to be touched by more than one bead in a wave. Delete the table if there are none.

| File | Rebuild when | Rebuild with | Restore the wave branch with |
| --- | --- | --- | --- |
| `<lockfile>` | `<a manifest changed>` | `<install command>` | `<install command>` |
| `<generated file>` | `<its inputs changed>` | `<command>` | `<command, or "nothing">` |

## Local services

- **<Service>:** `<how to start it>`, and anything to know before resetting or reconfiguring it.
- **Ports and URLs:** `<where they're documented>`

## Files to leave alone

Always untracked here. Never stage them: `<paths, or "none">`.

## Review

Describe the bot that reviews PRs, or write "Reviewer: none". With no reviewer, a PR is clean once its CI passes and every review thread from people is answered.

The lines below describe the Claude review workflow in it-works-on-local-dev/code-review, which a repository calls from the stub `/iwol:setup` writes. For that reviewer, keep them and fill in the placeholders. For any other reviewer, replace each value with that reviewer's own signals, keeping the labels.

- **Reviewer:** the shared Claude review workflow, called from `.github/workflows/<stub file, e.g. claude-code-review.yml>`, posting as `<slug>[bot]` (GraphQL login `<slug>`), where `<slug>` is the stub's `review-app-slug` input, or as `<app>[bot]` read from the bot's comments in PR history when the stub has no `review-app-slug`.
- **Review workflow ref:** `<v1 | main>`. The ref the stub calls the workflow at: `v1` moves only to compatible releases of the workflow; `main` follows every merge.
- **When it reviews:** by itself when a PR opens ready or is marked ready (not drafts, fork PRs or PRs opened by bots); after that only on request, so every later push needs a request. If a reviewer reviews every push by itself, or can't be asked, the skills never request reviews.
- **Requesting a review:** comment a line starting `/review` on the PR, optionally `/review [model] [effort]` (default `<the stub's default-model, or opus>` at `<the stub's default-effort, or xhigh>`). A request that doesn't parse gets a bot reply starting "Claude review request failed:" and doesn't count.
- **In progress:** a running `Claude / review` check for an automatic review (GitHub's UI shows it as `Code Review / Claude / review`; match on `Claude / review`), or a 👀 on the request comment for a requested one. The 👀 stays, so only a later summary or failure reply ends a round; only bot comments newer than the request count. An automatic review also leaves a 👀 on the PR description, which stays from the first review on, so it never marks a review in progress.
- **Clean:** a summary comment from the bot that starts with the heading `### No issues found! 🚀`, then, each after a blank line, the reviewer's short note, one line per check result from the workflow's checks job, and `Reviewed with ... effort` followed by a last line `Reviewed head: <sha>` naming the PR's head by its full 40-character SHA, bare (no backticks). Older summaries, still on PRs reviewed before the workflow changed, start instead with the plain line `No issues found.` (no heading and no note, and a `Requested ... effort` footer); both count as clean. A summary naming an older head, or saying the review was partial, isn't clean: the workflow adds a line starting `Partial review:` when the reviewer didn't cover every changed file, and an older summary says so in the reviewer's own words. <Keep only the one of the next two sentences that matches the stub's `approve` input:> With the stub's `approve: true`, a clean review whose checks all passed, run while every review thread the bot started was resolved, also submits a GitHub approval on the reviewed head, and a later review that isn't clean dismisses it; the summary stays the clean signal, and a PR can count as clean with no formal GitHub approval: when it's clean only because its findings were resolved, since the review that raised them gave none and dismissed any earlier one, when its summary lists a check as unavailable, since the workflow approves only when every check passed, or when its review ran while a review thread the bot started was still unresolved, since that review gave none even if the thread is resolved afterwards. Without `approve: true` in the stub, the reviewer never submits a GitHub approval.
- **Findings:** inline comments from the bot, one per finding. The summary starts with an `### N issue(s) found` heading (`### 1 issue found`, `### 2 issues found`) in place of the clean heading, then lists each finding as one `- ` line; an older summary lists them with no heading, in place of "No issues found.". No priority badges; take priority from the wording. A summary line with no inline comment is still a finding.
- **Counting review rounds:** count the bot's summary comments, the ones with a `Reviewed head: ` line. GitHub records an empty `COMMENTED` review per inline comment, so GitHub's review count overcounts.
- **Resolved findings count as clean:** `<yes or no>`. Yes means a review on the head commit counts as clean once every thread from it has an outcome reply and is resolved, but never a review that requests changes or says it was partial. Say no if the reviewer always gives an explicit clean signal or approval.
- **Usage limit:** no message of its own. A failed, timed-out or cancelled review gets a bot comment starting "Claude review didn't finish" for a requested review, or a red `Claude / review` check for an automatic one. A red `Claude / request` check with a skipped `Claude / review` also means the automatic review failed. Only when the stub has a picker job (a `pick` job named `Runner` that the review job needs), a red `Runner / …` check (the `Claude` check then shows as skipped) means the automatic review failed too; a repository without a picker job drops that sentence. A red `Claude / checks` stays out of it: the review still runs and reports the checks as unavailable. Each run, a failed one included, uploads the artifact `claude-review-pr<PR>-<sha>-attempt<n>` with the execution log, the check output and the diffs it read, kept for 14 days. Retry an automatic review with `gh run rerun <run-id>` (all jobs, so every job runs again, and a picker job picks the runner again if the stub has one); retry a requested one with a new `/review`.
- **Review rules:** generic rules live in the workflow's prompt; repository-specific ones in `<file, e.g. AGENTS.md, or "none">`, read from the base branch.

## CI

Write "CI: none" if no workflow runs on PRs.

- **Workflow:** `<workflow file in .github/workflows/, e.g. ci.yml>`, gating job `<job name, e.g. check>`. Name every job a PR's CI has to pass, e.g. jobs `check` and `database`.
- **Runs on:** `<hosted | self-hosted: <how to tell a queued job is waiting for one, and where recovering a stuck run is documented>>`. `hosted` means GitHub-hosted runners only, so a queued job is only waiting for GitHub. `self-hosted` means some gating job runs on the repository's own runners, so a job still `queued` once its run is `in_progress` can be waiting for one of them; the line then says how to tell and where recovering a stuck run is documented. A job that picks a runner and falls back to GitHub-hosted ones when none is online is one example of how a repository ends up here.
- **Recovering a stuck run:** `<where it's documented, e.g. docs/DEPLOYMENT.md § CI runners, or "none">`
- **Waiting for CI:** after a push, a PR skill waits for the run on the pushed commit with this loop, run in the background. It prints `<run id> completed <conclusion>` when the run finishes, or each job's status after 20 minutes; a `null null null` there means no run started for the commit.

```bash
sha=$(git rev-parse HEAD)  # when merging: sha=$(gh pr view <PR> --json headRefOid --jq .headRefOid)
for attempt in $(seq 1 40); do
  run=$(gh run list --workflow <workflow file> --commit "$sha" --json databaseId,status,conclusion --jq '.[0] | "\(.databaseId) \(.status) \(.conclusion)"')
  case "$run" in *" completed "*) echo "$run"; exit 0 ;; esac
  sleep 30
done
echo "not finished after 20 minutes: $run"
gh run view "${run%% *}" --json jobs --jq '.jobs[] | "\(.name): \(.status)"'
```

## Git and GitHub

- **Base branch:** `<main>`. The skills use it wherever they say `main`.
- **Worktrees:** `.claude/worktrees/`, which must be listed in the tracked `.gitignore` so fresh clones ignore it too.
- **Worktree setup:** `<where preparing a new worktree (installing dependencies, copying env files, generating code) is documented, e.g. AGENTS.md § Worktrees, or "none">`. `/iwol:start` and `/iwol:beads-implement` follow it in each worktree they create. Clean-up needs no pointer: `/iwol:create-pr` removes a merged PR's worktree itself.
- **Merge method:** `<merge | squash>`. Then, on the same line:
  - for `merge`, the merge commit's subject and body format: `<format>`;
  - for `squash`, either "GitHub deletes the branch on merge" or "the skill passes `--delete-branch`".
- **PR title format:** `<free | conventional commit>`. With `conventional commit`, keep `@word` out of PR titles: release-please links every `@word` in a title to that GitHub user's profile, which leaves a bogus link in the changelog.
- **Merge order:** `<any rule, e.g. database migrations in timestamp order, or "none">`
- **Auto-merge:** never. The skills merge PRs themselves and never enable GitHub's auto-merge.

## Releases

Write "Releases: none" if the repository has no release flow for the skills to drive. For release-please:

- **Release PR branch:** `<the Release PR's head branch, e.g. release-please--branches--main--components--<component>>`
- **Release workflow:** `<workflow file that creates the tag and the GitHub Release when the Release PR merges, e.g. release.yml>`
- **Deploy workflow:** `<workflow file the tag push runs, e.g. deploy.yml>`, gated by `<what has to pass before it deploys, e.g. lint, typecheck and unit tests in the same workflow>`. May be `none`, or the release workflow itself when the only deploy step is moving a tag (as a workflow that moves a `v<major>` tag to each release does). With no deploy workflow, `/iwol:create-release` watches only the release workflow's run.
- **Production version:** `<URL that reports the running build's version, e.g. https://example.app/api/version>`, answering `<the response's shape, e.g. {"version":"X.Y.Z"}, where builds other than production add "+<commit>">`. May be `none`, for a repository whose releases deploy nothing. With no production version, `/iwol:create-release` skips verification and reports the created tag and GitHub Release instead.
- **Migrations to check before a release:** `<directories whose new migrations a release must be inspected for, e.g. prisma/migrations and prisma-auth/migrations, or "none">`
- **Rollback:** `<where rolling back is documented, e.g. docs/DEPLOYMENT.md § Rollback>`
