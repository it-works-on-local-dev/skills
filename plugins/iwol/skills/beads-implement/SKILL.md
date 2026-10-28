---
name: beads-implement
description: Orchestrate the execution of a sub-epic of beads (work items) by fanning its child beads out to parallel sub-agents, then verifying all the results centrally. Use this skill whenever the user wants to build, run, execute, or implement beads — including phrases like "run phase 1", "implement this sub-epic", "execute these beads", "build out the beads", "kick off the sub-agents for this", or when they hand over a bead ID and ask for the work to be done. Also runs fix rounds (`fix <bead-id> …`) on beads with open findings, usually called by beads-pr. Always use this skill for bead execution — do not dispatch sub-agents or start implementing beads without it.
---

# Beads Implement

Orchestrate one sub-epic of a bead graph. Launch its child beads across parallel sub-agents — each seeing only its own bead — then verify every result yourself.

Two things define this role. First, **you write no implementation code.** Your job is dispatch, sequencing, and verification. Second, **all verification is yours alone.** Sub-agents produce artifacts; they do not get to grade their own work. An agent that just wrote a change is the worst possible judge of whether it broke something, and an agent running tests in its own sandbox tells you nothing about the state of the repo after every wave lands together.

This skill stops at verified, uncommitted work. Committing, the pull requests (one per bead), review rounds, merging and closing the wave belong to `/iwol:beads-pr`.

Read `.claude/iwol-config.md` first. It holds everything specific to this repository: check commands, generated files, local services, the PR reviewer and merge rules. If it's missing, stop and ask the user to run `/iwol:setup`, which creates it.

In these steps, `main` means the base branch the config names, and `origin/main` means that branch on `origin`.

## Step 0 — Require the bead ID (hard stop)

This skill operates on a specific bead ID.

If no bead ID is present in the conversation, **stop immediately**. Do not guess, do not list beads and pick a plausible-looking one, do not start from the top of the graph, and do not begin dispatching anything. Tell the user plainly that a bead ID is required and ask for it. Then end the turn and wait.

If the ID is given but does not resolve to a real bead, say so and stop. Executing the wrong branch of the graph is far more expensive to undo than asking.

If the args start with `fix`, skip to **Fix mode** below.

## Step 1 — Resolve and select

Load the bead and check what kind of node it is:

- **Sub-epic** — run its children. This is the normal case.
- **Epic** — stop. List its sub-epics and ask the user which one to run. An epic is a whole plan; running it wholesale defeats the phase-by-phase model the decomposition was built for.
- **Leaf bead** — run just that one.

Waves run strictly in order, and a wave is finished only when every bead's PR has merged and its sub-epic is closed. If an earlier sub-epic of the same epic isn't closed yet, stop and say which one: its PRs probably haven't all merged.

Then name the session `[<id>] implementation`, where `<id>` is the ID this run was given: the sub-epic, or a leaf bead. Rename it with the Claude desktop app's session-title tool (`set_session_title` with the session `self`; load it with ToolSearch if it's deferred). Where there's no such tool, as in the CLI, skip the rename without mentioning it. If the app asks the user to approve the new title, because they named the session themselves, leave it to them. Fix mode doesn't rename: it runs inside a `/iwol:beads-pr` session, which keeps its own name.

Every wave runs in its own worktree, named as in *Branch and worktree names* below, wherever the session started; the main checkout's branch is never switched. Create, enter and prepare it as `../start/worktrees.md` says (read it first), before dispatching anything. For a leaf bead, use its sub-epic's branch and worktree.

- **The wave's branch already exists** (`git branch --list 'feature/<sub-epic-id>-*'`): this is a re-run (after a blocked bead, a usage limit, or in a new session). Enter the wave's worktree as *Entering one* says, which also covers a worktree left on a bead branch. If the branch isn't checked out anywhere and the wave's worktree doesn't exist, create it on the branch (*Creating one*, without `-b`) and enter that. A wave started before waves had their own worktrees may be checked out in the main checkout: then run there, and if this session isn't there, stop and tell the user to run from the main checkout. Carry on with the beads that aren't closed, except those open only because of an open finding (a `Finding [` comment with no later `Resolved`, `Deferred` or `Declined` comment). Testing or review reopened those, and they're fixed through `/iwol:beads-pr`'s fix round, not implemented again. A `blocked` bead the user has since sorted out is run again here.
- **Otherwise,** create the wave's worktree on a new branch from `origin/main` (*Creating one*): `git worktree add "<main checkout>/.claude/worktrees/<sub-epic-id>-<title>" -b feature/<sub-epic-id>-<title> origin/main`. Enter it, and prepare it as the config's `Worktree setup:` line says (nothing when it's `none`). Uncommitted changes in the folder the session started in stay there, and aren't part of the wave.

Entering the worktree is what makes the wave run there: sub-agents start in the session's folder, so once the session is in the wave's worktree, every bead's edits land there. Git lets a branch be checked out in only one worktree, so every later session for this wave (`/iwol:beads-summary-and-test`, `/iwol:beads-pr`) enters the same worktree. The wave branch stays local. `/iwol:beads-pr` later splits it into one branch and PR per bead, using the file lists recorded in Step 4, and removes the wave's worktree once every PR has merged.

### Branch and worktree names

- **Branches:** `feature/<bead-id>-<title>`. For a wave's branch, `<bead-id>` is the sub-epic's ID.
- **Worktrees:** `<bead-id>-<title>`, under `.claude/worktrees/` (git-ignored in this repo), for any worktree made for a wave or a bead. Work that isn't a wave gets `<prefix>-<description>` from `/iwol:start` (see `../start/worktrees.md`). Worktrees the desktop app makes for its own sessions keep the app's names.
- **`<title>`** is the bead's title in lowercase, with every run of characters other than a–z and 0–9 turned into one hyphen, no hyphen at either end, and cut at a hyphen to 40 characters or fewer. For a sub-epic, leave out the leading `Wave <n> —`.
- **Examples:**
  - wave `acme-v1.6`, "Wave 6 — Group data and balances": branch `feature/acme-v1.6-group-data-and-balances`, worktree `.claude/worktrees/acme-v1.6-group-data-and-balances`;
  - bead `acme-v1.6.2`, "Build the web link session layer: server loading": branch `feature/acme-v1.6.2-build-the-web-link-session-layer-server`.
- **Find branches by ID**, never by working out the title again: a title can change after its branch exists. The wave branch is `git branch --list 'feature/<sub-epic-id>-*'`, which doesn't match its beads' branches; a bead branch is its PR's `headRefName`.

## Step 2 — Build the wave

A bead is runnable only when all of its dependencies are `done`. Group the currently runnable beads into a wave.

Then check the wave for collisions: **never put two beads that edit the same files into one wave.** Docs count too. The only exceptions are the generated files the config lists, which tooling rebuilds. Parallel agents writing the same surface produce work that looks fine individually and is broken together.

Where you find a likely collision, stop before dispatching. Tell the user which beads would share which file, and suggest moving one of them to the next wave (`bd update <id> --parent <next sub-epic>`, plus `bd dep add` on the other bead). Don't run the pair one after the other inside the wave. Each bead becomes its own PR, and the second bead's changes would sit on top of the first's, so its PR couldn't stand alone on `main`.

## Step 3 — Dispatch, one sub-agent per bead

When the wave starts, set its sub-epic to `in_progress` (`bd update <sub-epic> --status in_progress`). It stays that way until `/iwol:beads-pr` closes it once every bead's PR has merged.

Before dispatching a bead, set its status to `in_progress` (e.g. `bd update <id> --status in_progress`). Do this for every bead in the wave as you pick it up, right before launching its sub-agent — it marks the bead as claimed for the duration of the run, not just planned.

Hand each sub-agent **only its own bead's text** — its description, design, acceptance criteria, notes, and all of its comments (`bd comments <id>`). Comments carry owner decisions and findings deferred from earlier reviews, so they are part of the bead. No plan, no sibling beads, no access to this conversation. The beads were written to stand alone; treat that as a contract and do not paper over a weak bead by smuggling context in alongside it. If a bead genuinely cannot be executed on its own, that is a finding to report, not a gap to fill in silently.

Pick the model from the bead's `agent:` label — `sonnet`, or `opus` — and set the effort from its `effort:` label. Do not substitute either. The labels encode a judgement made with full sight of the plan; overriding them from inside a single bead's context is guesswork.

This is a hard constraint on the tool call itself, not a suggestion to weigh.
- Dispatch every bead with its own `Agent` call, the whole wave in one message so the calls run concurrently. The `Agent` tool is the only dispatch mode: there's no `Workflow` mode, and `- use a workflow` on the invocation isn't something this skill does (see *Why only the Agent tool* below).
- Dispatch every bead to the worker definition for its two labels: `subagent_type: iwol:bead-<agent>-<effort>`, such as `iwol:bead-opus-xhigh`. The definitions come from the `iwol` plugin, not from `.claude/agents/`. Those definitions set the model and the effort. The `Agent` tool takes a `model` but has no effort parameter, so a definition is the only way to set effort.
- On every dispatch, also pass `model` set to the bead's `agent:` label.
- If the definition for a bead's label pair doesn't exist, first check that the pair is one the plugin ships: `opus` or `sonnet`, each at `low`, `medium`, `high` or `xhigh`. If it isn't (`effort:max`, `agent:haiku`, or a typo), that is a defect in the bead. Stop and flag it to the user, as for a missing label below. Only for a shipped pair with no definition, stop because the `iwol` plugin isn't loaded in this session. Ask the user to run `/reload-plugins` or start a new session; don't create a definition file.
- Never omit the model or the effort to let the tool auto-route or inherit a default. Never round an `effort:xhigh` bead down to `high` because a model doesn't nominally support it, and never swap `opus` for `sonnet` (or vice versa) because the task looks simpler or costlier than the label implies.
- If a bead is missing either label, that is a defect in the bead. Stop and flag it to the user rather than guessing a model or effort to fill the gap.

The same applies in the other direction: if a bead carries **more than one** value from the same label set (e.g. both `agent:opus` and `agent:sonnet`, or both `effort:low` and `effort:xhigh`), that is a conflicting bead, not a bead you can resolve by picking the "obviously right" one. This commonly happens when a bead was created with `bd create --parent <epic>` without `--no-inherit-labels` — the parent epic's own labels silently merge with whatever `--labels` was passed. Stop and ask the user which value to use, exactly as you would for a missing label; do not guess, and do not default to the parent epic's label on the theory that it's more authoritative.

Honour the task-kind label:

- `code-change` — writes files
- `review-only` — produces findings, changes nothing
- `research` — produces notes
- `docs` — produces documentation
- `test-only` — touches tests only

### What to tell every sub-agent

- Meet the acceptance criteria by producing the required artifacts — code, docs, notes, review findings, or tests.
- **Do not run any tests, linters, type checks, e2e suites, builds, or other verification commands.**
- **Do not perform validation beyond making sure your own changes are internally consistent** — obvious syntax errors you can see at a glance, nothing more.
- Report in observable terms: what you did, which files you created, modified, or deleted, and any manual testing instructions you propose for the orchestrator to run.
- Leave all verification to the orchestrator.

### Why only the Agent tool

The skill this one came from offered a second dispatch mode, the `Workflow` tool with one `agent()` call per bead in a script, on request (`- use a workflow`). It's gone from this skill. It existed there to set each bead's effort, which the worker definitions do here, and everything after dispatch (the scope check on each diff, the once-per-wave format and refresh, central verification, re-dispatch, the shared-cause reporting) is judgement work that this session does as results land, not something a script could run ahead of it. An option nobody runs would still have to stay correct in every one of those steps. If a wave ever grows too large for its results to fit in this session's context, the thing to add back is a workflow for the dispatch step alone, returning each bead's report, with everything after it still done here.

## Step 4 — Run the wave and verify centrally

Launch the whole wave concurrently. Wait for all of it to finish before judging any of it — a bead that looks green in isolation can still be broken by what its sibling landed.

Then work through the wave in two parts. The first runs once for the whole wave, and again after each round of re-dispatches (see below); don't start the second until every item of the first is finished.

**For the whole wave**, in this order:

1. For each completed bead, collect its reported changes and its proposed manual testing instructions. If it said it needed something in code its bead leaves alone, record that on the bead, so later sessions can see it: `Needs outside this bead: <file:line> — <the change that code needs>`.
2. **Rebuild generated files yourself** before verifying; sub-agents don't. For each generated file in the config whose inputs changed, run its rebuild command.
3. **Format and refresh the wave** once, after every bead of the wave has landed and before running any check: run the config's `Format:` command, then its `Format check:` command, then its `Post-wave refresh:` command, skipping each one the config gives as `none`. Files they change are accounted for in the beads' `Files:` comments like the rest of the wave's changes: a reformatted file belongs to the bead that changed it, and refreshed output follows the config's generated-files rules.
4. **Run all verification yourself**: the config's checks, plus anything else the repo has (tests, e2e, lint, type checks, build).

**Then, for each completed bead:**

5. Record the bead's files in a comment: `Files: <path>, <path> (deleted), <path> (generated), …`. List every file the bead created, modified or deleted, taken from the sub-agent's report and checked against `git status` and `git diff`. Add the generated files it caused, marked `(generated)`: each one in the config whose inputs this bead changed. `/iwol:beads-pr` uses these lists to give each bead its own PR. If a non-generated file shows up in two beads' lists, say so in your report.
6. Mark the bead `done` only when both hold: every acceptance criterion is met in observable terms, *and* every verification command you ran passes. If you accept a failure, document explicitly what failed and why you accepted it — never mark something done past a silent failure.

On verification failure, keep the bead `in_progress` while you deal with it. Record what failed — the commands, the output, the files involved — and either re-dispatch with that context attached or flag it for a human if it looks like the bead itself is wrong. Wait for every re-dispatched agent in flight to finish, then run the whole-wave part again (items 1–4) once, before each re-dispatched bead's part (items 5–6), so their changes are rebuilt, formatted, refreshed and verified like the rest of the wave's. When you flag it, or it still fails after re-dispatching, set it to `blocked` (`bd update <id> --status blocked`). Add a comment saying what failed and what it needs, so it doesn't look like an agent is still working on it.

### Re-dispatching: start new agents, don't resume old ones

Resuming a sub-agent with `SendMessage` replays its whole transcript, and by the end of a wave that is 150k–650k tokens. Once the prompt cache has expired, every resume rewrites all of it before any work starts. Resumes like this have made up over 90% of a wave's sub-agent cache writes during PR review.

- To re-dispatch, start a **new** agent of the bead's type (same `agent:` and `effort:` labels). Brief it with the bead's text and comments, the files the first agent changed, and the exact failure: commands, output and file:line.
- Resume the same agent only for a quick follow-up sent within a few minutes of its last turn, such as "also regenerate the types".
- If a usage limit cuts a sub-agent off, wait for the reset, then start a new agent with the bead's text and the files already changed (from `git status`). Don't resume the old one.

## Step 5 — Report

Beads in a sub-epic never depend on each other, so there's no second round inside it. Once every bead in the wave is `done` or `blocked`, the run is over.

Leave the sub-epic `in_progress`, and don't commit. `/iwol:beads-pr` commits the work, opens one PR per bead, and closes the sub-epic when they have all merged.

Then report:

- What completed.
- What is blocked, and why.
- Anything a sub-agent said it needed in code its bead leaves alone (shared code, or another bead's files), with the change that code needs.
- The full set of verification and manual testing steps you ran across the whole sub-epic, with their results.
- The next step: `/iwol:beads-summary-and-test <sub-epic id>`, in a new session.

## Fix mode

Called as `fix <bead-id> [<bead-id> …]`, normally by `/iwol:beads-pr` after a review or manual test turned up findings. Each bead is a leaf bead with at least one open finding: a comment starting `Finding [` with no later comment starting `Resolved <its id>`, `Deferred <its id>` or `Declined <its id>`. If a bead has no open findings, say so and leave it alone.

1. Reopen each closed bead (`bd reopen <id> --reason "Fix round: <k> open findings"`), then set every bead `in_progress`. A repair bead that `/iwol:beads-pr` just created is already open.
2. **Keep each fix inside its own bead's files.** A bead's fix may change only the files in its `Files:` comments, plus new files it creates, which then become its own. If a finding looks like it needs another bead's file, or two beads' fixes would touch the same file, don't dispatch those beads. Report it to the caller instead; `/iwol:beads-pr` asks the owner. Running such fixes one after the other would put one bead's change into another bead's PR, so it couldn't stand alone on `main`.
3. Dispatch a **new** sub-agent per bead, never a resumed one, with the bead's `agent:` and `effort:` labels as in Step 3. Its brief is the bead's full text and comments, headed by a list of the open findings it must fix. Its job is to fix those findings without breaking the bead's acceptance criteria. List the files it may change (the bead's own), and tell it to report instead of editing anything else. Tell it to **change only what each finding needs**: no refactoring, renaming or tidying of other code. Every extra change is new code for the next review to find problems in, and that's how review rounds multiply. The same rules about not running verification apply.

   Tell it too that **a cause outside its bead's files isn't its to fix.** When a finding's cause is in a file outside the bead's files (shared code, or another bead's file), it must not work around it in its own files: no height estimates from the window size, no timers to wait out a race, no copies of shared hooks or helpers. It reports the cause instead, with file:line evidence and the change that file needs, and leaves that finding open. A workaround hides the symptom on one screen, leaves it on every other screen that uses the shared code, and is more code to take out once the cause is fixed. In one fix round, five agents worked around two shared causes this way, with window-size height estimates, a copied hook and a settle timer, and all of it had to be removed again.
4. Check each sub-agent's changes against its bead's files (`git status`, `git diff`). An edit to another bead's file counts as a failed verification (step 6), even if every check passes. So does a workaround or a copy of shared logic: read each diff for sizes estimated from the window or screen, timers or delays that wait out a race, holds that hide a flash, and copies of a hook, helper or constant that shared code already has. When you re-dispatch, brief the new agent to take it out and report the cause instead. Then rebuild generated files as in Step 4. After that, and before any check, run the config's `Format:` command, then its `Format check:` command, then its `Post-wave refresh:` command, skipping each one the config gives as `none`. Then verify centrally as in Step 4: the config's checks for everything the findings touch, plus each finding's own check where it has one.
5. For each finding that is fixed and verified, add a comment `Resolved <finding id>: <what changed>. Verified with <commands>.` If the fix created, deleted or regenerated a file that isn't in the bead's `Files:` comments yet, add a new `Files:` comment listing them. Files that step 4's format and refresh changed go in by the same rules as Step 4, item 3: a reformatted file belongs to the bead that changed it, and refreshed output follows the config's generated-files rules. Then close the bead with a reason once none of its findings is open. A bead with a finding left open for a cause outside its files stays `in_progress` (step 7).
6. If verification fails, re-dispatch once to a new agent with the failure attached. Then wait for every re-dispatched agent in flight to finish, and run step 4 once for them (the scope check, the rebuild, `Format:`, `Format check:`, `Post-wave refresh:` and verification), before judging any of them. Judging earlier lets the formatter rewrite files another agent is still editing, counts a sibling's half-finished work as this bead's second failure, or fails `Format check` on unformatted output. If it fails again, set the bead to `blocked`, add a comment saying what failed, and report it.
7. **Report causes outside the beads' files.** Check each cause a sub-agent reported against the code before passing it on. If it's in the bead's own files after all, re-dispatch as in step 6. Otherwise record it on the bead whose finding it explains, so a later run doesn't have to work it out again: `Cause for <finding id>: <file:line> — <the change that file needs>`.
   - **In another bead's file:** report it to the caller, as in step 2.
   - **In shared code no bead owns** (a file in no `Files:` comment of a bead whose PR hasn't merged, usually because it's already on `main`): when findings on two or more beads share the cause, stop work on those findings. Don't re-dispatch them, and don't accept a fix for them in any bead's files. Report the cause to the caller and propose one repair bead under the wave's sub-epic that owns the shared file and fixes the cause once, with those findings deferred to it. Give the cause with file:line evidence, the change the file needs, and each finding it explains (bead and finding id). Report a shared cause behind a single bead's finding the same way. `/iwol:beads-pr` asks the owner and creates the repair bead; fix mode creates none.

Don't commit, and don't touch the sub-epic's status. Report back what was fixed, what is still open (with the causes from step 7), and the files changed. The caller commits.

## Non-goals

- Don't start beads outside this sub-epic.
- Don't edit a bead's description to make it pass. If a bead is wrong, say so.
- Don't collapse multiple beads into a single sub-agent, even when they look adjacent.
- Don't delegate test execution, linting, type checking, e2e, or builds to sub-agents under any framing. Verification is yours.
- Don't commit, push, open or update PRs, or close the sub-epic. That's `/iwol:beads-pr`.
- Don't resume a sub-agent after a long gap or a usage-limit cutoff. Start a new one.
- Don't accept a fix that works around a cause outside its bead's files or copies shared logic. The cause gets reported, and fixed once where it lives.
