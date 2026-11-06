---
name: setup
description: Set up the iwol plugin in a repository, or re-check a repository that already uses it. Writes .claude/iwol-config.md, the file every iwol skill reads for the repository's checks, reviewer, CI, merge method and releases, migrating an old .claude/beads-config.md; enables the plugin in .claude/settings.json; asks which reviewer the repository uses and, for the public review workflow (Claude, Codex or both), writes its stub; and reports what's still missing, such as bd, gh, the worktree ignore rule, the review secrets, and local skill or agent copies to delete. Use when adopting the plugin in a repository, when another iwol skill stops because .claude/iwol-config.md is missing, when the user runs /iwol:setup or says things like "set up iwol", "set up the plugin in this repo", "configure beads for this repo" or "create the iwol config", and to re-check the setup after the repository's tooling changes.
---

# Setup

Set up a repository to use the iwol plugin, or re-check one that already does. This is the single entry point for adopting the plugin. It writes up to three files, `.claude/iwol-config.md`, `.claude/settings.json` and the review stub `.github/workflows/code-review.yml` (`claude-code-review.yml` in a repository kept on v1 of the review workflow), and checks everything else the other iwol skills rely on. It doesn't decompose plans, create beads, open PRs or run anything.

Three rules:
- **Work it out, don't guess.** Every value comes from a file in the repository, a command's output, or the user. When something can't be determined, ask.
- **Nothing is saved without the user's yes, and a re-check shows only the differences.** Show each file before saving it. If it already exists, show only what would change, and propose a change only where a value differs in substance, never to reword one that already says the same thing.
- **Stay in your lane.** Don't install tools, change repository or GitHub settings, set secrets or variables, edit the plugin's skills, delete files or commit. With the user's OK you may write the three files above, rename `.claude/beads-config.md`, and a v1 stub moving to v2, with `git mv`, and add `.claude/worktrees/` to `.gitignore`.

## Step 1 — Check the plugin and find what's there

- **The plugin is loaded.** Look at the skills and agent types this session offers. The plugin's skills carry an `iwol:` prefix (`iwol:beads-decompose-plan`, `iwol:beads-implement`, `iwol:beads-summary-and-test`, `iwol:beads-pr`, `iwol:grill-me`, `iwol:plan-digest`, this one, and any others it ships; `iwol:side-quest` isn't listed at all, since its `disable-model-invocation` setting hides it from Claude, so never take its absence as a sign). Its eight worker agents are `iwol:bead-<model>-<effort>`, for each `<model>` in `opus`, `sonnet` and each `<effort>` in `low`, `medium`, `high`, `xhigh`; beads-implement dispatches every bead to one of them. If any are missing, the plugin isn't enabled, or this session started before it was. The fix is enabling it (Claude Code needs the `iwol` marketplace registered on this computer, as the README of `it-works-on-local-dev/skills` describes, and Step 6 writes the repository's setting), then `/reload-plugins` or a new session. Carry on with the other steps either way, and report it at the end.
- **The reviewer.** Right after the plugin check, ask one question: which reviewer does this repository use?
  1. **The public review workflow** in `it-works-on-local-dev/code-review`, which reviews with Claude, Codex or both, and which the repository calls from a stub this skill writes (Step 3).
  2. **Another review bot.**
  3. **None.**

  On a re-check, a migration, or wherever a review workflow caller is already in place, infer the answer first and ask the user to confirm it rather than asking afresh:
  - a workflow in `.github/workflows/`, usually the stub `code-review.yml` (`claude-code-review.yml` from v1), has a job whose `uses:` names a workflow path ending in `/.github/workflows/code-review.yml` (v2) or `/.github/workflows/claude-code-review.yml` (v1): the first answer, whatever owner, repository or ref that path names today. A stub that still calls the workflow from its old home is recognised this way, and Step 3 proposes the public path in its place. In the review workflow's own repository (below), the local stub that calls it by its path counts the same;
  - otherwise, the existing or old config's Review section describes another bot: the second;
  - it says `Reviewer: none`: the third;
  - nothing to infer from: ask.

  The answer gates the steps: Step 3 (the stub) and Step 5 (the review secrets and the app) run only for the first answer, and Step 2's Review bullet fills the config's Review section for each of the three. If the user picks the second or third while a workflow still calls the review workflow, say that it keeps reviewing PRs until they remove it; this skill deletes nothing.
- **Plugin settings.** Read `.claude/settings.json`, if there is one. The plugin is enabled for everyone who clones the repository when it has `"enabledPlugins": {"iwol@iwol": true}`. If it doesn't, propose it:
  - no file: create one holding only `{"enabledPlugins": {"iwol@iwol": true}}`;
  - a file without it: merge `"iwol@iwol": true` into its `enabledPlugins`, adding that key if needed, and keep every other key and every other plugin's entry as it is;
  - a file that isn't valid JSON: show the error and leave it alone.

  Never add `extraKnownMarketplaces`: the marketplace's source differs per machine, a local clone on one and GitHub on another. If the file already has an entry for `iwol` there, point it out and suggest removing it. Also read `.claude/settings.local.json`: `"iwol@iwol": false` there turns the plugin off in this checkout only, so tell the user.
- **Existing config:**
  - `.claude/iwol-config.md` exists: this is a re-check. Compare it with what Step 2 finds, and propose only the differences.
  - Only `.claude/beads-config.md` exists: this is a migration. Step 2 starts from it, and Step 6 renames it with `git mv .claude/beads-config.md .claude/iwol-config.md` (a plain rename if git doesn't track it) before writing the new content, so its history follows.
  - Both exist: tell the user. Re-check `.claude/iwol-config.md`, take nothing from the old file without asking, and suggest they delete it.
  - Neither: write a new config from the template.
- **The review workflow's own repository.** If `.github/workflows/code-review.yml` or `.github/workflows/claude-code-review.yml` is triggered by `on: workflow_call`, that file is the reusable review workflow itself, and the repository reviews its own PRs through a separate local stub that calls it by its path. Step 3 writes no stub here.
- **Local copies of what the plugin ships.** A local copy loads beside the plugin under its bare name, so the session offers two versions of the same skill or agent. Find them, delete nothing, and list them in the report:
  - in `.claude/skills/`: `beads-decompose-plan`, `beads-implement`, `beads-summary-and-test`, `beads-pr`, `grill-me`, `plan-digest`, `summary-and-test`, `create-pr` and `create-release`, each only if this session offers the plugin's own `iwol:<name>` (otherwise say the plugin doesn't ship it yet, and it stays); `side-quest`, whenever the session offers the plugin's other skills, since it ships in the same plugin but its `disable-model-invocation` setting keeps `iwol:side-quest` out of the list; and `beads-setup`, which `/iwol:setup` replaces;
  - in `.claude/agents/`: bead workers under any naming, such as `bead-opus-xhigh.md` or `bead-worker-high.md`. That's every `bead-*.md`, and any other agent whose description says beads-implement dispatches beads to it.

## Step 2 — Work out the config

Follow the sections of `config.template.md`, in this skill's folder. For each value, note where it came from. When migrating, start from the old file: keep every value in it that still holds, fill in the sections and lines it lacks, rewrite its Review section as below, and reword nothing else. Its old **Merging** line becomes **Merge method** and **Auto-merge**; drop any old text that limits the skills to one merge method.

- **Checks:**
  - Read the build scripts (`package.json`, `Makefile`, `pyproject.toml` or similar) and the CI workflows (`.github/workflows/`).
  - The full check is what CI runs, as one local command if the repository has one.
  - List any other suites CI runs, such as database or end-to-end tests, and when they're needed.
  - Note anything that hides failures, such as a recursive test run that stops at the first failing package.
  - **Format** and **Format check:** propose them from the build scripts: the script that rewrites files into the format (`format`, `fmt`, `prettier:write` or similar) and the one that only checks it (`format:check`, `prettier`, `fmt:check` or similar), or the equivalent targets in a `Makefile` or other build file. Write each as a command with the package manager the repository uses (`packageManager` in `package.json`, else the lockfile: `pnpm format`, `npm run format`, `yarn format`). A script that both rewrites and checks, or one you can't tell apart, isn't a match: ask. `none` when the repository has no formatter. Mark them as inferred. For reference: a pnpm repository might format with `pnpm format` and check with `pnpm format:check`; an npm one might format with `npm run format` and check with `npm run prettier`.
  - **Post-wave refresh:** propose a command only when the repository shows a derived artifact that a command refreshes, such as a committed knowledge-graph or index output directory with its refresh script, or a hook that runs that script. Name the evidence. Otherwise ask the user, defaulting to `none`. The output it refreshes is a generated file too, so it gets a row in Generated files.
  - On a re-check, or a migration whose old file already has these three lines, compare them with what the above finds and show the current values for the user to confirm.
- **Generated files:**
  - Look for lockfiles (`pnpm-lock.yaml`, `package-lock.json`, `yarn.lock`, `poetry.lock`, `Cargo.lock`, `go.sum` and the like).
  - Look for generated code: files with a "generated" header, scripts named like `codegen`, `generate` or `*:types`, and anything CI checks for drift.
  - For each, record what makes it need rebuilding, the rebuild command, and how to restore the wave branch afterwards.
  - **Ships**, for each: `with each bead PR` by default. Propose `once per wave` for output that nearly every bead changes and that no bead's PR needs in order to pass CI, such as the knowledge-graph or index directory the post-wave refresh rebuilds: carried on every bead PR, it makes each merge conflict with the rest. Name the evidence, mark it as inferred, and ask. A lockfile always ships with each bead PR, since a bead that changes a manifest needs it. On a re-check, or a migration, of a table without the Ships column, propose adding it: `once per wave` for a row whose text already says bead PRs leave the file out, and `with each bead PR` for the rest.
  - Note whether CI installs with a frozen lockfile.
- **Local services:** databases, emulators, Docker Compose files and dev servers. Record how to start each one, anything to know before resetting it, and where ports are documented.
- **Files to leave alone:** run `git status --short` and ask which of the untracked files are always there.
- **Review:** fill the section by Step 1's answer.
  - **The public review workflow:** use the template's Review section as it stands, with its *A stub on v1* changes for a stub kept on v1 (Step 3), and fill its placeholders from the stub as Step 3 works it out (in the review workflow's own repository, from the local stub that calls it by its path):
    - the stub's file;
    - the bot's login, which the **Reviewer** line names in place of the template's placeholders and its explanation of where a login comes from, as in posting as `my-review-app[bot]` (GraphQL login `my-review-app`):
      - when the stub passes `review-app-slug`: `<slug>[bot]`, with GraphQL login `<slug>`;
      - otherwise, read the review app's `<app>[bot]` login from PR history, as the bot that posts the review summaries: `gh api 'repos/{owner}/{repo}/issues/comments?sort=created&direction=desc&per_page=100' --jq '.[] | select(.user.type == "Bot" and (.body | contains("Reviewed head: "))) | .user.login' | sort -u`, with the GraphQL login the same without `[bot]`;
      - **when that gives no login, ask the user for the review app's login, as a question of its own alongside the others.** That covers a repository with no PR history yet, or none the review app has posted on, and one where the lookup can't run at all: no GitHub repository (no `origin`, or `gh repo view` fails) or a refused call. If the app doesn't exist yet, ask for the login it will have, and point at Step 5's README for creating it.
      - Never save the Reviewer line with `<app>[bot]`, `<slug>` or any other placeholder in it, or with a login marked unconfirmed or left to fill in later. Never fill in a login you haven't read or been told;
    - **Review workflow ref:** the ref the stub calls the workflow at once saved: `v2` for a new stub or one moving from v1, `v1` for one the user kept on v1, or another ref the user kept (Step 3); for a ref other than `v2` and `v1`, replace the template's explanation with one saying what that ref pins. Write this line whether or not the template has it yet. In the review workflow's own repository, where the local stub calls the workflow by its path with no ref, leave the line out, and on a re-check there propose dropping it;
    - the defaults: on v2, the `default-reviewer` (`claude` unless the stub sets it) and, for each engine the stub passes a secret for, its model and effort, the workflow's own unless the stub sets `default-<engine>-model` or `default-<engine>-effort`, leaving out the defaults of an engine it passes no secret for; on v1, the model and effort, the workflow's own unless the stub sets `default-model` or `default-effort`;
    - in the **Requesting a review** line, on v2: each engine's default credential, the stub's `default-<engine>-credential` when it's `sub` or `api`, otherwise the one of the engine's two secrets the stub passes (with both passed, `auto` or no input means `api`), and the engine secrets the stub passes. On v1, where the workflow at the stub's ref offers the credential choice (Step 3): the default credential, the stub's `default-credential` or else the one Claude credential it passes (with both passed, `auto` or no `default-credential` means the API key), and the Claude credential secrets the stub passes (with `secrets: inherit`, the ones that are set). Where it doesn't, drop the line's `[oauth|api-key]`, its default credential, the credential in its PR description sentence and its last sentence, about the two words. In the review workflow's own repository, the local workflow file decides whether the choice is offered;
    - the review rules file: `AGENTS.md` if the repository has one; otherwise ask;
    - whether resolved findings count as clean: ask the user, unless the existing or old config already says;
    - in the **Clean** line, only the approval sentence that matches the stub's `approve` (Step 3): the `approve: true` sentence when the stub passes it, otherwise the sentence "Without `approve: true` in the stub, the reviewer never submits a GitHub approval.";
    - in the **Usage limit** line, the sentences about a red `Runner / …` check (the review check, `Reviewer / Claude` on v2 or `Claude / review` on v1, then shows as skipped) and about a requested review whose `Runner / pick` fails only when the stub has a picker job (Step 3), with the picker job's `name:` in place of `Runner` if it has another one. Of the two cases that follow for a request with no 👀, keep the one that looks up the request's own run, with the stub's file name (as `--workflow` takes it, such as `code-review.yml`) in place of `<stub file>`, only when the stub as it will be saved also has the `run-name:` that names each comment's run (Step 3); otherwise keep the one that tells the user, since the lookup would match a run name the stub never sets. A picker stub without it is, for example, one whose re-check `run-name:` the user declined, or the local stub in the review workflow's own repository, which Step 3 doesn't write. Without a picker job, drop all those sentences, and the **In progress** line's clause about a request that never got its 👀 with them.

    When migrating, rewrite the old Review section to the workflow's signals the same way, and when a v1 stub moves to v2 (Step 3), rewrite a Review section that describes v1's signals to the v2 lines. They replace an older reviewer's, such as `@claude review` requests, a summary with a different heading, or a SHA in backticks. Keep anything repository-specific the old section says that still applies.
  - **Another review bot:** look at recent merged PRs (`gh pr list --state merged --limit 10`), with their reviews, comments and reactions, and record that bot's signals under the template's labels: its REST and GraphQL logins, when it reviews and how a review is requested, its in-progress and clean signals and whether it leaves a formal GitHub approval, how findings arrive and how to count rounds, any usage-limit message, and the rules file it follows. Leave out the **Review workflow ref:** line, which belongs to the public review workflow. Ask the user what the history doesn't show. If the history shows no bot reviewing PRs, ask the user which bot it is, or whether the answer is really none.
  - **None:** the section is `Reviewer: none`. Tell the user what that means for the other skills: a PR is clean once its CI passes and every review thread from people is answered, and merging needs the user's explicit yes.
  - On a re-check, compare the existing section with what the above gives in substance only.
- **CI** and **Releases:** read what the repository shows, propose it marked as inferred, and ask only for what it doesn't show. Explain each line you ask about with the template's wording. On a re-check, or a migration whose old file already has them, compare the current values with what's found and show them for the user to confirm.
  - CI: list `.github/workflows/` and read each workflow's `on:` triggers and `jobs:`. The workflows that run on `pull_request` are the candidates; propose the gating one and the job or jobs a PR has to pass (for example `ci.yml`, jobs `check` and `database`, or `ci.yml`, job `check`), using each job's `name:` as the check shows it, or its job ID when it has none. Leave out the review stub (`code-review.yml`, or `claude-code-review.yml` on v1), any workflow that calls the review workflow (by its public path, `it-works-on-local-dev/code-review/.github/workflows/code-review.yml` or `…/claude-code-review.yml`, or by its local path, `./.github/workflows/code-review.yml` or `./.github/workflows/claude-code-review.yml`, in the workflow's own repository), and any workflow that only labels, comments or releases. If several workflows could gate, say which you chose and why, and ask. A job that calls a reusable workflow shows as `<job> / <inner job>`: give the name as the PR's checks list it. "CI: none" if no workflow runs on PRs.
  - `Runs on`: `hosted`, or `self-hosted: <how to tell a queued job is waiting for one, and where recovering a stuck run is documented>`. Look at the gating CI workflow the CI line names and its jobs, and follow the local reusable workflows those jobs call (`uses: ./.github/workflows/…`); a review workflow's jobs don't count. Propose `hosted` when every gating job's `runs-on:` names GitHub-hosted labels (`ubuntu-latest` and the like). Propose `self-hosted` when a gating job's `runs-on:` names `self-hosted` or another label that isn't GitHub-hosted, or reads a runner-picking job's output: a job that picks one of the repository's own runners, falling back to GitHub-hosted ones when none is online, is one way a repository ends up on self-hosted runners. Then fill in the rest of the line from the repository's docs (`AGENTS.md`, `CLAUDE.md`, `docs/`, `README`), or ask the user what they don't say. On a re-check, an old `picker` value becomes `self-hosted: …`, filled in the same way.
  - `Recovering a stuck run`: where the repository's docs (`AGENTS.md`, `CLAUDE.md`, `docs/`, `README`) say how to recover a stuck or queued CI run, such as a CI runners section. If they don't say, ask, defaulting to `none`.
  - `Waiting for CI`: the template's wait loop with the gating workflow's file name filled in; nothing to ask.
  - Releases: when a workflow has a step that `uses:` an action ending in `/release-please-action` (under `googleapis/` or `google-github-actions/`) or runs the `release-please` CLI, or the repository has a `release-please-config.json` or `.release-please-manifest.json`, propose what can be read, marked as inferred. Read the config from the action's `config-file` input (`release-please-config.json` by default); with no config file, the action's `release-type` input stands in for it:
    - the Release PR branch: release-please names it `release-please--branches--<base branch>` (the action's `target-branch` input, when it sets one, in place of `<base branch>`), plus `--components--<component>` for the root package's component: the `component` key when the config has one (top-level, or on a single package in `packages`), otherwise the component release-please defaults to for the release type (for `node`, `package.json`'s `name` without its `@scope/`). Confirm it against past Release PRs' head branches (`gh pr list --state all --search "head:release-please" --json headRefName`), and ask when they disagree or there are none. With several components or packages, ask which one the skills drive;
    - the release workflow and the deploy workflow from `.github/workflows/`: the one with a step that `uses:` an action ending in `/release-please-action` (under `googleapis/` or `google-github-actions/`) or runs the `release-please` CLI is the release workflow (when a config file exists but no workflow matches, ask which workflow releases); the one triggered by a tag push (`on: push: tags`) or by the release being published is the deploy workflow. When no workflow is triggered that way, propose `Deploy workflow: none`, or the release workflow itself when its only deploy step moves a tag (as a workflow that moves a `v<major>` tag to each release does), marked as inferred, with no gate;
    - the production version: when the deploy workflow is `none` or the release workflow itself, the repository's releases deploy nothing, so propose `Production version: none`, marked as inferred;
    - the migration directories that exist (`prisma/migrations`, `migrations/`, `db/migrate`, `supabase/migrations` and the like), or `none` when there are none.

    Ask for the rest: when there's a deploy workflow other than the release workflow, the production version URL and the shape of its response, or `none` if the deploy has no URL that reports its version, and what gates the deploy (propose what the deploy workflow shows, such as jobs it `needs:` or steps it runs first, and have the user confirm); and where rollback is documented (propose a docs heading that mentions rollback, if one exists). With no release-please workflow, config file or manifest and no workflow that looks like a release flow, propose `Releases: none`. If the repository releases some other way (another release tool, a manual tag, a publish workflow), tell the user the Releases section only describes release-please flows, and ask what to record: `none`, or a short description of their flow.
- **Git and GitHub:**
  - **Base branch:** `gh repo view --json defaultBranchRef`.
  - **Worktree setup:** look for a section in the repository's instructions (`AGENTS.md`, `CLAUDE.md`, `docs/`) on preparing a new worktree, such as installing dependencies, copying env files or generating code, and ask the user to confirm it. `none` if there isn't one. `/iwol:start` and `/iwol:beads-implement` follow it in each worktree they create. Clean-up isn't asked for: `/iwol:create-pr` removes a merged PR's worktree itself. An old config's **Worktree setup and clean-up** line becomes this line, keeping its pointer only if that section covers setup.
  - **Merge method:** `merge` or `squash`; the skills never rebase-merge. Work it out from four things:
    1. Which merge methods GitHub allows: `gh api repos/{owner}/{repo} --jq '{allow_merge_commit, allow_squash_merge, allow_rebase_merge, delete_branch_on_merge}'`.
    2. Whether the base branch requires linear history. That rules out `merge`, since a merge commit isn't linear, but not `squash`. Check both:
       - rulesets: `gh api repos/{owner}/{repo}/rules/branches/<base> --jq '[.[].type]'`, looking for `required_linear_history`;
       - branch protection: `gh api repos/{owner}/{repo}/branches/<base>/protection --jq .required_linear_history.enabled`.

       A 404 from the protection call means the branch isn't protected. A 403 saying the plan doesn't include the feature means no such rule can exist.
    3. Which methods the base branch's rulesets allow. A `pull_request` rule ("Require a pull request before merging") has its own `allowed_merge_methods`, which can forbid `merge` or `squash` even when the repository settings allow it: `gh api repos/{owner}/{repo}/rules/branches/<base> --jq '.[] | select(.type == "pull_request") | .parameters.allowed_merge_methods'`. Drop any method it doesn't list. No `pull_request` rule, or one without the parameter, limits nothing. If the call is refused, say so and don't treat the methods as unrestricted.
    4. How recent PRs landed on the base branch itself, not on whatever is checked out: `git fetch origin`, then `git log origin/<base> --first-parent --merges --format=%s -5` for merge commits and `git log origin/<base> --first-parent --no-merges --format=%s -10` for single commits, where a PR squashed on GitHub ends its subject with ` (#<n>)` by default. `--first-parent` leaves out merges that happened inside feature branches, such as "Merge main into feature".

    If only one of `merge` and `squash` is left after the first three, that's the method. If both are, recent history decides when it's consistent; ask when it's mixed, when there's none, or when it disagrees with an old config. If neither is (only rebase merges allowed, or linear history required with squash merges off), tell the user the skills can't merge here as written. Then:
    - for `merge`, the subject and body format, from the recent merge commits (`git log origin/<base> --first-parent --merges --format='%s%n%b%n--' -5`); confirm it with the user;
    - for `squash`, whether GitHub deletes branches: `delete_branch_on_merge` true gives "GitHub deletes the branch on merge", false gives "the skill passes `--delete-branch`".
  - **Merge queue:** the skills merge each PR directly and never use auto-merge, so a merge queue on the base branch doesn't fit them. Check with `gh api graphql -F owner='{owner}' -F name='{repo}' -f query='query($owner:String!,$name:String!) { repository(owner:$owner, name:$name) { mergeQueue(branch:"<base>") { id } } }'`. Anything other than `null` means the base branch has one; tell the user the merge stage won't work here as written.
  - **PR title format:** ask the user: `free`, or `conventional commit` (titles like `feat: …`, usually because release-please builds the changelog from them). On a re-check, show the current value and ask the user to confirm it.
  - **Merge order:** rules such as database migrations by timestamp. Ask the user.
  - **Auto-merge:** always `never`.

  If there's no GitHub repository to ask (no `origin`, or `gh repo view` fails), say so, take these values from the old config or the user, and mark them unconfirmed.

## Step 3 — Work out the review stub

**Run this step only when Step 1's answer is the public review workflow.** With another review bot or none, write, replace and propose no stub, and say so in the report.

**In a repository whose `.github/workflows/code-review.yml` or `.github/workflows/claude-code-review.yml` is triggered by `on: workflow_call`, skip this step too.** That file is the reusable review workflow itself, and the repository reviews its own PRs through a separate local stub that calls it by its path (`uses: ./.github/workflows/code-review.yml`, or `claude-code-review.yml` before v2). Write, replace and propose nothing for either; report that the stub doesn't apply here.

Everywhere else, the stub is `.github/workflows/code-review.yml`, calling the workflow at v2, from *Review stub* below. A repository the user keeps on v1 keeps its stub, usually `.github/workflows/claude-code-review.yml`, calling `claude-code-review.yml@v1`, with the v1 differences *Review stub* lists. First check what's there, in either file:

- **A stub file has a job whose `uses:` names a workflow path ending in `/.github/workflows/code-review.yml` or `/.github/workflows/claude-code-review.yml`**, whatever owner, repository or ref: this is a re-check of the stub. The path tells its version: `code-review.yml` is v2, `claude-code-review.yml` is v1.
  - **A v1 stub:** propose moving it to v2, the recommended version, and ask; `@v1` keeps working, so the user may keep it. Moving means `git mv .github/workflows/claude-code-review.yml .github/workflows/code-review.yml` when the stub has that name; the path `it-works-on-local-dev/code-review/.github/workflows/code-review.yml@v2`; the `review` job's `name: Reviewer` in place of `Claude`; `default-model`, `default-effort` and `default-credential` renamed `default-claude-model`, `default-claude-effort` and `default-claude-credential`, with `oauth` and `api-key` becoming `sub` and `api`; the `issue_comment` comment as in the v2 stub; and the *Engines* question below, with Claude as the proposed answer. The config's Review section moves to the v2 lines with it (Step 2). Say what changes for the repository's PRs: the review check becomes `Reviewer / Claude`, so a branch protection rule or ruleset that requires `Claude / review` has to drop it (nothing should require the engine-named check), and `/review` lines take `sub` and `api` and refuse `oauth` and `api-key`. If the user keeps v1, compare the stub with the v1 stub instead, and propose nothing about engines.
  - If the path isn't `it-works-on-local-dev/code-review/.github/workflows/<its file>`, as with a stub still pointing at the workflow's old home, propose the public path in its place, at the ref the ref item below settles. Compare its inputs, its `secrets:`, its jobs (a `pick` job, its `if:` included, and the `review` job's `needs:` and `runs-on`) and, with a `pick` job, its `run-name:` with what this step works out, and also its workflow `name:` and the `review` job's `name:` with the stub below. Propose only the differences, including `Code Review` and `Reviewer` (`Claude` on v1) where they differ, so the stub's check is `Reviewer / Claude` (`Claude / review` on v1), the name the config uses. The `pick` job's `if:` is compared, in substance, with the template's current filter (the *Self-hosted runners* item below), with the `@<slug>` line exactly when the *Mentions* answer below has the stub pass `review-app-slug`, with the slug it passes, and without it otherwise, and a difference is proposed, so a stub written before the filter changed gets the current one on its next re-check: a pick job still on the old filter `github.event_name == 'pull_request' || github.event.issue.pull_request`, which spent a runner job on every PR event and every comment on a PR, is one such difference. If the user declines the change, the existing job's `name:` (or its job ID when it has none) replaces the job part of every check name in the config's Review section (`<job name> / Claude`, `/ Codex`, `/ request` and `/ checks` on v2; `<job name> / review`, `/ request` and `/ checks` on v1), and its workflow `name:` replaces `Code Review` in the UI name. Likewise, a picker job under another name keeps that name in place of `Runner`.
- **It's a full review workflow of its own**, with its own steps rather than one job calling the review workflow: say the stub will replace it, and ask before going on. Its commands for installing, formatting, linting and type checks are a good source for the `checks` snippet.
- **Another workflow already calls the review workflow** (`grep -lE 'uses: *[^ ]*/\.github/workflows/(claude-)?code-review\.yml' .github/workflows/*`, leaving out the stub itself): say so and ask. A second caller would review every PR twice. If you keep that caller, its job's `name:`, or its job ID when it has none, sets the check names, so that name replaces the job part of every check name in the config's Review section, as for a declined change above, and its workflow `name:` replaces `Code Review` in the UI name. Check its `secrets:` as the `secrets:` item below says, but only report what differs: that file isn't one this skill writes.

Then work out the ref, the options and the inputs. They must match the workflow's interface. At v2: inputs `package-manager` (default `npm`), `node-version-file` (default `.nvmrc`), `exclude-paths` (default none), `checks` (required), `review-app-client-id` (required), `review-app-slug` (default empty), `runs-on` (default the JSON string `"ubuntu-latest"`: a JSON label or array of labels for the workflow's checks and review jobs), `default-reviewer` (`claude` or `codex`, default `claude`), `default-claude-model` (default `opus`), `default-claude-effort` (default `xhigh`), `default-claude-credential` (`auto`, `sub` or `api`, default `auto`), `default-codex-model` (default `gpt-6.1-sol`), `default-codex-effort` (default `xhigh`), `default-codex-credential` (`auto`, `sub` or `api`, default `auto`) and `approve` (default `false`); secrets `CLAUDE_CODE_OAUTH_TOKEN` and `ANTHROPIC_API_KEY` (Claude's `sub` and `api`), `CODEX_AUTH_JSON` and `OPENAI_API_KEY` (Codex's `sub` and `api`), each optional to GitHub, though the `default-reviewer` engine needs at least one of its two or every run fails early, and `REVIEW_APP_PRIVATE_KEY` (required). At v1: the same, except that Claude is the only engine, so there's no `default-reviewer`, no Codex input and no Codex secret, and the Claude inputs are `default-model`, `default-effort` and `default-credential` (`auto`, `oauth` or `api-key`, and only in releases that offer the credential choice, below).

- **The ref:** `v2` for a new stub, or one moving from v1; don't ask. `v2` moves to each compatible 2.x release, so the stub gets fixes and features without changing. `v1` is frozen at 1.2.1, the last release of the workflow as `claude-code-review.yml`, and keeps working for a stub the user keeps on v1. The workflow's `main` renamed that file to `code-review.yml`, so a stub calling `claude-code-review.yml@main` fails at startup on every PR, with GitHub's "This run likely failed because of a workflow file issue". The config's Review section records the ref the stub calls once saved (Step 2). On a re-check:
  - **A v2 stub at `v2`:** keep it. At another ref, such as `main` or a full tag like `v2.0.0`, show it and ask whether to keep it or move to `v2`: `main` gets every merged change before it's released, and a full tag none until it's moved by hand.
  - **A v1 stub at `main`, or at another ref where `claude-code-review.yml` is gone** (`gh api 'repos/it-works-on-local-dev/code-review/contents/.github/workflows/claude-code-review.yml?ref=<ref>' --jq .name` fails with a 404): it can't run, so propose moving it to v2 (above), or, if the user keeps v1, `v1` in its place, saying why.
  - **A v1 stub at any other ref**, such as `v1` or a commit SHA pinning a 1.x release, that the user keeps on v1: keep it. If it's not `v1`, show it and ask whether to keep it.
- **Self-hosted runners:** ask whether the review's jobs should run on the repository's own self-hosted runners, defaulting to no. With no, the stub has no `pick` job and passes no `runs-on`, so the workflow's checks and review jobs run on GitHub-hosted runners. With yes, the stub gains the optional picker block in *Review stub* below: a `pick` job named `Runner`, with empty permissions, that `uses:` a reusable runner-picker workflow and receives `RUNNER_STATUS_TOKEN` as its only secret, and whose `if:` is a copy of the review workflow's `request` job's filter, so the picker starts only when a review might follow: on `pull_request` events, a non-draft PR whose head is in the repository itself and whose author isn't a bot; on `issue_comment` events, a comment on a PR (not a plain issue) from an owner, member or collaborator whose body contains `/review`, or `@<slug>` when the stub passes `review-app-slug` (the slug is written literally: a caller can't read the workflow's `inputs`); and the `review` job gains `needs: pick` and passes `runs-on: ${{ needs.pick.outputs.linux }}` (the picker's output is already JSON). The filter matters because a hosted job is billed a whole minute however short, and the workflow's own `request` job, which decides which events get a review, can't gate a job that runs before it: without the filter the picker would spend a billed minute on every PR event and every comment on a PR, the bot's own summary comments included. When the picker is skipped, the `review` job, which needs it, is skipped too. The filter must change when the workflow's request filter changes; the README of `it-works-on-local-dev/code-review` shows the current one under *Self-hosted example*. The picker the template's `pick` job calls is an example; a user with their own picker substitutes its path, and its output, if it names it differently. With this shape the picker's checks show as `Code Review / Runner / …`, and a red `Runner / …` check, with the review check (`Reviewer / Claude` on v2, `Claude / review` on v1) then shown as skipped, means the automatic review failed. A requested review whose pick fails is quieter: the called workflow never starts, so the request gets no 👀 and no failure reply, and the failed run's checks don't show on the PR, since GitHub attaches checks from `issue_comment` runs to the default branch's head commit. So the picker block also gives the workflow a `run-name:` that names each comment's run `Code Review (comment <id>)`: `gh run list` returns every comment run in the repository with nothing else saying which comment it belongs to, so without it another PR's failed review, or a comment that only mentions `/review` in prose, could pass for a failed request. The config's Review section then says both, and how to find the request's own run by its comment's id (Step 2). `RUNNER_STATUS_TOKEN` goes to the `pick` job only, never to the `review` job. On a re-check, a stub with a picker job means yes.
- **Mentions:** ask whether the bot should respond to mentions of its login in PR comments, so a thank-you to `@<slug>` earns a 🎉 reaction; a `/review` line is what asks for a review, with or without the slug. Default no, so without it mentions are ignored. With yes, the stub passes `review-app-slug` with the bot's login without `[bot]`: read it from a recent PR comment by the review app's `<app>[bot]` login (Step 2's lookup) where the repository has one, else ask for it. With no, leave the input out. On a re-check, a stub that passes `review-app-slug` means yes.
- **`package-manager`:** `pnpm` when `package.json`'s `packageManager` field starts with `pnpm@`, otherwise `npm`. If the field names another package manager, or there's a `pnpm-lock.yaml` without the field, tell the user and ask: the workflow sets up only npm and pnpm.
- **`node-version-file`:** leave it out when `.nvmrc` exists, since that's the default. Otherwise ask which file holds the Node version, such as `.node-version`. The workflow sets Node up from it, so the file must exist.
- **`exclude-paths`:** ask which paths to leave out of the diffs the reviewer reads, one per line, such as a generated output directory like `dist`. Offer the generated directories from the config's Generated files as candidates. Leave the input out when there are none.
- **`checks`:** propose a snippet from the repository's own scripts, or the old review workflow's commands, and ask the user to confirm or change it. Each line is `run_step <setup|check> <name> <command…>`, with `<name>` a single word, since it names the step's output file. A name may use only letters, digits, `.`, `_` and `-` (no `:`, `/` or spaces, so `format-check` rather than `format:check`). Each `<name>` must be unique across the snippet and must not be `summary`: the workflow reports a bad name, a repeated name, or `summary`, as a FAIL line and skips the step.
  - Each `run_step` line wraps exactly one simple command, because the workflow runs `"$@" > "$CHECKS_DIR/<name>.txt"`. Use no `&&`, pipes, redirects or `VAR=value` prefixes. `run_step check full actionlint && claude plugin validate .` wraps only `actionlint`: the second command runs outside `run_step`, gets no summary line, and if it fails the step's `bash -e` stops the checks there: `full` still reads pass, and the steps after it show as unavailable. `run_step setup gen FOO=1 npm run gen` runs `FOO=1` as the command and fails. Split a compound command into separate steps, wrap it as `bash -c '…'`, and `export` variables at the top of the snippet.
  - `setup` steps prepare the checks: installing (`npm ci`, or `pnpm install --frozen-lockfile`) and generating code the checks need. They add a summary line only when they fail.
  - `check` steps are what the reviewer reads instead of guessing at formatting, lint or type errors: usually the format check, lint and typecheck, as CI runs them. Each always adds a pass or FAIL line. Leave tests to CI unless the user wants them here.
  - If a step needs environment variables, such as a dummy database URL for code generation, `export` them at the top of the snippet. The checks job has no secrets, so never put a real one there.
- **`review-app-client-id`:** always `${{ vars.REVIEW_APP_CLIENT_ID }}`.
- **Engines** (v2 only): ask which engines the repository's reviews may use: Claude, Codex, or both. The answer sets which engines' secrets the stub passes. With both, also ask which one reviews when a `/review` line names neither an engine nor a model, the automatic first review included, and write `default-reviewer: codex` when it's Codex; leave the input out for Claude, its default. With Codex alone, write `default-reviewer: codex`: the default, `claude`, would have no credential, and every run would fail before the checks start. On a re-check, the engines whose secrets the stub passes, and its `default-reviewer`, are the answer.
- **Claude credentials** (when Claude is one of the engines, and always on v1): the workflow accepts either credential and needs at least one when Claude is the default reviewer.
  - **The credential choice** (v1 only: every v2 release offers it): check whether the workflow at the stub's ref declares the `default-credential` input: `gh api -H 'Accept: application/vnd.github.raw' 'repos/it-works-on-local-dev/code-review/contents/.github/workflows/claude-code-review.yml?ref=<ref>' | grep -E '^on:|^ +default-credential:'`. It prints `on:` for any workflow it could read, then a `default-credential:` line when the workflow declares the input (1.2.1, which `v1` names, does). A workflow that declares it lets a `/review` line name `oauth` or `api-key`, and hands the review only the credential it runs on. GitHub refuses to start a called workflow that is passed an input it doesn't declare, so a stub that passes `default-credential` to an older one stops every review.
    - `on:` alone: the choice isn't offered. Write no `default-credential`, ask no default below, and leave `[oauth|api-key]` out of the stub's `issue_comment` comment and the config's **Requesting a review** line (Step 2). Say so in the report, and that running `/iwol:setup` again once the workflow at that ref declares the input adds them.
    - No `on:`: the lookup failed, for example a refused call or a ref that doesn't exist, which tells you nothing about the input. Say so and ask the user whether the workflow at that ref declares `default-credential`. Until they answer, keep an existing stub's `default-credential` and the config's credential wording as they are, and propose neither adding nor removing them.
  - **Which credentials:** ask which Claude credential the repository's reviews will use: the Claude Code OAuth token (`CLAUDE_CODE_OAUTH_TOKEN`, for example from `claude setup-token`), an Anthropic API key (`ANTHROPIC_API_KEY`), or both. The question says what both means: with the credential choice offered, each `/review` line can pick one (`sub` or `api` on v2, `oauth` or `api-key` on v1), and a default is asked next; without it, every review runs on the API key. Propose the answer from the stub's `secrets:` on a re-check, otherwise from which of the two Step 5's name-only lists show set, both when both are; with neither set, ask without a proposal. A stub on `secrets: inherit` names neither and passes every credential that's set, so there the proposal comes from Step 5's lists, and an answer that leaves out a credential that's set takes effect only with the named block below: say so if the user keeps `inherit`. The answer sets the stub's `secrets:`, the secrets Step 5 reports missing, and the config's **Requesting a review** line.
  - **The default Claude credential:** only when the credential choice is offered, and either the user chose both credentials or the stub stays on `secrets: inherit` while both are set. Ask which one a Claude review uses when its `/review` line names neither, and write `default-claude-credential: sub` or `api` (v2), or `default-credential: oauth` or `api-key` (v1), under `with:`. The question says why the line matters: the input's default, `auto`, takes the API key whenever both are passed, so without the line every Claude review that names no credential, the automatic first review included, runs on the API key. With one credential chosen, and under `inherit` only that one set, leave the input out, since `auto` takes that one. On a re-check, the stub's value is the answer, a v1 stub's `oauth` or `api-key` becoming `sub` or `api` as it moves to v2, and where this item applies a missing or `auto` value is asked about. Propose a change where it names a credential the stub doesn't pass, since every review that names no credential then fails, and on v1 propose removing it where the lookup above printed `on:` alone.
- **Codex credentials** (v2, when Codex is one of the engines): `OPENAI_API_KEY` (`api`), the recommended route, and `CODEX_AUTH_JSON` (`sub`), the `auth.json` of a stored ChatGPT login. Check the repository's visibility first (`gh repo view --json visibility --jq .visibility`): OpenAI allows the stored login only in private repositories, so where it prints `PUBLIC`, pass only `OPENAI_API_KEY`, ask nothing about `sub`, and on a re-check propose removing a `CODEX_AUTH_JSON` the stub passes. Elsewhere, ask which to pass, proposing `OPENAI_API_KEY`, and say that a stored login goes stale after about 8 days and must then be stored again by hand, and that one login mustn't serve concurrent jobs. With both, ask which one a Codex review uses when its line names neither, and write `default-codex-credential: sub` or `api`; the default, `auto`, takes the API key whenever both are passed. With one, leave the input out. Point the user at the code-review README's *Codex credentials* section for storing either one.
- **`secrets:`:** the `review` job's block in *Review stub* below, which passes secrets by name: `REVIEW_APP_PRIVATE_KEY` always, plus the engine credentials the user chose (above). Never pass `RUNNER_STATUS_TOKEN` to the `review` job: the workflow doesn't declare it, and only the optional `pick` job takes it. Never write `secrets: inherit`: the stub calls the workflow at a tag its owners can move, and `inherit` would hand it every repository and organisation secret, not just the ones it needs. On a re-check:
  - `secrets: inherit`, the form stubs had before: it still works, so it isn't a failure. Propose the block, saying it keeps the repository's other secrets away from a workflow called at a moving ref. If the user declines, leave it and say so in the report.
  - A block that leaves out `REVIEW_APP_PRIVATE_KEY`, or both credentials of the default reviewer's engine (Claude's on v1): the review can't run, so propose replacing the block.
  - A block that passes `RUNNER_STATUS_TOKEN`, or any other secret the workflow doesn't declare, to the `review` job: GitHub refuses to start a called workflow that is passed a secret it doesn't declare, so propose removing it (to the `pick` job, if the user turns self-hosted runners on).
  - A block that reads a secret from another name, such as `CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_TOKEN }}`: keep that mapping, and check that name in Step 5.
- **Default models and efforts:** leave them out, so the workflow's own apply (Claude `opus`, an alias for the newest Opus model the workflow's Claude Code knows, at `xhigh`; Codex `gpt-6.1-sol` at `xhigh`), unless the user wants others: `default-claude-model`, `default-claude-effort`, `default-codex-model` and `default-codex-effort` on v2, `default-model` and `default-effort` on v1. The workflow checks the Codex values on every run, Claude reviews included, so a Codex model must be a `gpt-` slug and its effort one of `low`, `medium`, `high`, `xhigh`, `max` and `ultra`; a Claude effort can't be `ultra`.
- **`approve`:** ask whether a clean review should also approve the PR, defaulting to no. The question's own text must say that the approval is only a visible signal, on any plan: whoever writes a PR can obtain it, so it must never count toward required reviews (a private repository on GitHub's Free plan can't require approvals anyway). Keep that in the question itself, not only in text around it, so it still shows when the questions are asked together in one form. With `approve: true`, a clean review submits a GitHub approval on the head it reviewed, and a later review that isn't clean dismisses it; a push leaves the approval in place until the next review. Write `approve: true` only when the user turns it on, and leave the input out otherwise. On a re-check, compare the stub's `approve` with the answer like any other input.
- **The `pull_request` trigger's branch** is the config's base branch.

## Step 4 — Check the tools

- **`bd`:** installed and set up in this repository. Run `bd info`, and tell the two failures apart:
  - *Command not found:* `bd` isn't installed. Point the user to the beads (`bd`) installation instructions; `bd init` can't help until it's installed.
  - *Installed, but no database here:* tell the user to run `bd init` in the repository.

  Then check that it has the commands and flags the skills use, since an older `bd` can pass `bd info` and still fail later:
  - `bd create --help` lists `--no-inherit-labels`;
  - `bd update --help` lists `--external-ref`;
  - `bd reopen --help` lists `--reason`;
  - `bd comments --help` lists `--json`;
  - `bd children` and `bd dep add` exist.

  If anything's missing, tell the user to upgrade `bd`.
- **`gh`:** signed in with access to this repository (`gh auth status`, `gh repo view`). The skills fetch and push through the `origin` remote but call GitHub through `gh`'s default repository, so those must match. Compare `gh repo view --json nameWithOwner` with `git remote get-url origin` and `git remote get-url --push origin`, and stop if they point to different repositories.
- **Worktrees:** `.claude/worktrees/` must be in the tracked `.gitignore` (`git check-ignore --no-index -v .claude/worktrees/probe`). If it isn't, offer to add it.

## Step 5 — Check the review secrets and app

Run this step only when Step 1's answer is the public review workflow; with another review bot or none, check no review secrets and say so. Skip it too when there's no GitHub repository, and say so.

- **Read only names, never values.** Repository secrets: `gh secret list --json name --jq '.[].name'`. Repository variables: `gh variable list --json name --jq '.[].name'` (a plain `gh variable list` prints values too). Organisation secrets and variables shared with this repository: `gh api repos/{owner}/{repo}/actions/organization-secrets --jq '.secrets[].name'` and `gh api repos/{owner}/{repo}/actions/organization-variables --jq '.variables[].name'`. If a call is refused, say which list you couldn't read instead of calling its names missing.
- **Report which of these are set and which are missing:**
  - the engine credential secrets the user chose (Step 3), or, in the review workflow's own repository, where Step 3 is skipped, the ones its local stub passes, of `CLAUDE_CODE_OAUTH_TOKEN`, `ANTHROPIC_API_KEY`, `OPENAI_API_KEY` and `CODEX_AUTH_JSON`: name each one that isn't set yet. At least one of the default reviewer's engine's two must be set, or every run fails; say so when neither is. Another engine with neither set fails only the requests that name it. Where the user chose both of an engine's credentials but only one is set, a review on the other fails: a `/review` line naming it gets a failure reply, and when that engine's default credential input (`default-claude-credential` or `default-codex-credential`, or v1's `default-credential`) names it, so does every review of that engine that names no credential, the automatic one included;
  - secret `REVIEW_APP_PRIVATE_KEY`, required;
  - variable `REVIEW_APP_CLIENT_ID`, required;
  - secret `RUNNER_STATUS_TOKEN`, only when the stub has a picker job: required for the picker to pick; without it the picker falls back to GitHub-hosted runners.

  Where the stub's `secrets:` reads one of the secrets from another name (Step 3), check that name instead. For each missing one, give the user the commands to run themselves (`gh secret set <name>`, which prompts for the value, or `gh variable set REVIEW_APP_CLIENT_ID --body <the review app's client ID>`), and point them at the README of `it-works-on-local-dev/code-review` for the Codex secrets (its *Codex credentials* section, which stores `CODEX_AUTH_JSON` compacted), for creating the review GitHub App (permissions Contents: Read-only, which a private repository needs, and Pull requests and Issues: Read and write), installing it on the repository, and setting the client-ID variable and the private-key secret. Never name a value, never set one yourself, and never ask for a value in the session.
- **The review GitHub App must be installed on the repository.** `gh`'s token can't check that reliably. If a recent PR comment here comes from the app's `<slug>[bot]` login (Step 2's lookup), it's installed; otherwise ask the user to check the app's installation in their GitHub settings, and that its repository access includes this repository. In a repository that isn't public (`gh repo view --json visibility --jq .visibility` prints anything but `PUBLIC`), also ask the user to confirm the app has Contents: Read-only, even when every secret is set: an app created before the README asked for it can lack it, and the review then can't fetch the base branch. If they add it to an existing app, the installation's owner has to accept the updated permissions in their GitHub settings before reviews get it.

## Step 6 — Show, save and hand over

1. Show what you'd write, file by file, marking the values you inferred rather than read directly and anything the user should confirm:
   - the config: in full when it's new, the rename and the new content when migrating, only the differences on a re-check;
   - `.claude/settings.json`, as the whole file would read;
   - the stub, unless Step 3 was skipped;
   - the `.gitignore` line.

   Save each on the user's yes; one yes can cover several if the user says so. When migrating, run the `git mv` first, then write the new content.
2. Report what's ready and what's missing:
   - whether the plugin is loaded, and if not, why and the fix;
   - each file, written or left as it was;
   - `bd`, `gh` and the worktree ignore rule;
   - the reviewer the user chose, and with none, what that means for the other skills (Step 2);
   - the review secrets, the variable and the app, or that they weren't checked because the repository uses another review bot or none, and a stub the user kept on `secrets: inherit`;
   - which version the stub calls: v2, a v1 stub moved to v2 with what changes for the repository's PRs (Step 3), or a stub the user kept on v1; and on v1, when the workflow at the stub's ref doesn't offer the credential choice yet, that the stub and config leave it out until a re-check (Step 3);
   - anything that stops the merge stage working as written: no usable merge method, or a merge queue;
   - the local copies to delete, with the commands (`git rm -r .claude/skills/<name>`, `git rm .claude/agents/<file>`), and that anything pointing at them, such as instructions or tests that read `.claude/skills/` or `.claude/agents/`, needs updating too.
3. If the plugin wasn't loaded, tell the user to reload the plugins or start a new session once it's enabled: agent definitions only load when a session starts. Give the command in a `text` block:

   ```text
   /reload-plugins
   ```

4. Suggest committing the config, the settings, the stub and the `.gitignore` change together, along with deleting the local copies. Don't commit.
5. Give the next steps:
   - `/iwol:grill-me` to settle a plan's open questions, if it has any, then `/iwol:plan-digest` for the go/no-go on the plan;
   - `/iwol:beads-decompose-plan <path to plan>`, once per plan;
   - then for each wave: `/iwol:beads-implement <sub-epic id>`, `/iwol:beads-summary-and-test <sub-epic id>` in a new session, and `/iwol:beads-pr <sub-epic id>` in a new session each time until the wave closes;
   - for work that isn't a wave: `/iwol:start <description>`, from a session started in the main checkout, to do it in its own named worktree, then `/iwol:summary-and-test` to test it and `/iwol:create-pr` to ship it;
   - `/iwol:side-quest <bead id or prompt>` to do one small task in a worktree of its own and leave it committed on a local branch, which `/iwol:start <branch>` and `/iwol:create-pr` pick up later (name it here even though the session doesn't list it, as in Step 1);
   - any other `iwol:` skills this session offers, such as `/iwol:create-release`.

## Review stub

`.github/workflows/code-review.yml`, calling the workflow at v2, in the shape of the stub in the README of `it-works-on-local-dev/code-review`. Fill in the base branch, the ref, the package manager and the `checks` snippet. Pass `REVIEW_APP_PRIVATE_KEY` and the engine credentials the user chose (Step 3). Leave out `exclude-paths` when there are none, and add `node-version-file` or a default model or effort input under `with:` only as Step 3 says. Add `approve: true` under `with:` only when the user turned approvals on. The lines marked `[optional: …]` are written only when their condition holds; drop the marker comments themselves from the file you write:

- the picker block, the `pick` job, plus the `run-name:` line with its comment and the `review` job's `needs: pick` and `runs-on:`, only when the user turned self-hosted runners on, with their own picker's path in place of the example if they have one;
- `review-app-slug`, and the line of the `pick` job's `if:` that matches a comment mentioning `@<slug>` (the same slug), only when the user wants the bot to respond to mentions: without the input the workflow ignores mentions, so the picker needn't start for them. Drop that one line only; the `) &&` line after it stays;
- each engine credential line, for the credentials the user chose, and `CODEX_AUTH_JSON` never in a public repository;
- `default-reviewer`, only when Step 3's *Engines* item says to write it;
- `default-claude-credential` and `default-codex-credential`, only when Step 3's credential items say to write them.

**A stub on v1**, only for a repository the user keeps on v1, keeps its file, usually `.github/workflows/claude-code-review.yml`, and differs from the template below in:

- `uses: it-works-on-local-dev/code-review/.github/workflows/claude-code-review.yml@v1`;
- the `review` job's `name: Claude`, so its checks read `Code Review / Claude / review`, `/ request` and `/ checks`, and the comment above the job says so;
- the `issue_comment` comment's request line, `/review [model] [effort] [oauth|api-key]`, ending at `[effort]` where the workflow at the ref doesn't offer the credential choice;
- only the Claude credential lines, and no `default-reviewer` or Codex input;
- `default-credential: <oauth or api-key>` in place of `default-claude-credential`, and `default-model` and `default-effort` in place of the other `default-claude-*` inputs.

```yaml
# This repository's pull requests are reviewed through the shared review workflow
# in it-works-on-local-dev/code-review. Written by /iwol:setup.
name: Code Review
# [optional: self-hosted runners] run-name, from here...
# Names a comment's run after the comment, so a /review's own run can be found by the comment's
# id; other runs keep the default title
run-name: ${{ github.event_name == 'issue_comment' && format('Code Review (comment {0})', github.event.comment.id) || '' }}
# ...to here

on:
  pull_request:
    types: [opened, ready_for_review]
    branches: [<base branch>]
  # A comment line starting "/review [claude|codex] [model] [effort] [sub|api]" asks for a fresh review
  issue_comment:
    types: [created]

jobs:
  # [optional: self-hosted runners] picker block, from here...
  # Picks the runners the review's jobs run on
  pick:
    name: Runner
    # Copies the review workflow's request job's filter, so the picker starts only
    # when a review might follow; review needs pick, so it's skipped too otherwise
    # [optional: mentions] the '@<slug>' line, with the slug review-app-slug passes
    if: >-
      (github.event_name == 'pull_request' &&
        github.event.pull_request.draft == false &&
        github.event.pull_request.head.repo.full_name == github.repository &&
        github.event.pull_request.user.type != 'Bot') ||
      (github.event_name == 'issue_comment' &&
        github.event.issue.pull_request &&
        (contains(github.event.comment.body, '/review')
          || contains(github.event.comment.body, '@<slug>')
        ) &&
        contains(fromJSON('["OWNER", "MEMBER", "COLLABORATOR"]'), github.event.comment.author_association))
    permissions: {}
    uses: it-works-on-local-dev/runners/.github/workflows/pick-runner.yml@main
    secrets:
      RUNNER_STATUS_TOKEN: ${{ secrets.RUNNER_STATUS_TOKEN }}
  # ...to here

  review:
    # GitHub shows the called workflow's checks as "Code Review / Reviewer / <job>", the review job
    # named after its engine, as in "Code Review / Reviewer / Claude"
    name: Reviewer
    needs: pick  # [optional: self-hosted runners]
    # The workflow posts through the review app's own token
    permissions:
      contents: read
    uses: it-works-on-local-dev/code-review/.github/workflows/code-review.yml@v2
    # Only the secrets the workflow declares, not every repository and organisation secret
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}  # [optional: Claude's sub credential]
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}  # [optional: Claude's api credential]
      OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}  # [optional: Codex's api credential]
      # [optional: Codex's sub credential, private repositories only] this comment and the line below
      # The stored ChatGPT login, which OpenAI allows only in private repositories
      CODEX_AUTH_JSON: ${{ secrets.CODEX_AUTH_JSON }}
      REVIEW_APP_PRIVATE_KEY: ${{ secrets.REVIEW_APP_PRIVATE_KEY }}
    with:
      package-manager: <npm or pnpm>
      review-app-client-id: ${{ vars.REVIEW_APP_CLIENT_ID }}
      default-reviewer: codex  # [optional: Codex as the default engine]
      default-claude-credential: <sub or api>  # [optional: both Claude credentials]
      default-codex-credential: <sub or api>  # [optional: both Codex credentials]
      runs-on: ${{ needs.pick.outputs.linux }}  # [optional: self-hosted runners]
      review-app-slug: <the bot's login without [bot]>  # [optional: mentions]
      exclude-paths: |
        <path>
      checks: |
        run_step setup install <install command>
        run_step check <name> <command>
```

## Non-goals

- Don't decompose plans, create beads, open PRs or run waves.
- Don't install tools, change repository or GitHub settings, set secrets or variables, or add any workflow other than the review stub.
- Don't write a stub when the repository uses another review bot or none, or in a repository whose `.github/workflows/code-review.yml` or `.github/workflows/claude-code-review.yml` is triggered by `on: workflow_call`.
- Don't put skills or agents into the repository: the plugin ships them. Local copies are listed for the user to delete, not deleted.
- Don't guess the CI, Releases, format or refresh lines: read them from the repository, mark them as inferred, and ask for what it doesn't show.
- Don't edit the plugin's other skills; anything specific to the repository belongs in the config.
- Don't save anything without the user's yes, and don't commit.
