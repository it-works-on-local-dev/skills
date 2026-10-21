---
name: beads-summary-and-test
description: Summarize the work done on a Beads epic, verify it against the actual codebase (committed or not), then walk the user through manual testing one step at a time, splitting a long test into groups run one per session. Use when the user runs /iwol:beads-summary-and-test, or asks to "summarize and test this epic", "walk me through testing this epic", or hands over an epic ID after implementation and wants a brief recap plus guided manual verification. Failures are recorded on the beads and handed to /iwol:beads-pr, which fixes them, opens one PR per bead and closes the wave. Do not use for running/implementing beads — that is beads-implement. For changes that aren't a Beads wave, use /iwol:summary-and-test.
---

# Beads Summary and Test

Recap what an epic actually did, confirm it against the real codebase, then hand the user a guided, one-step-at-a-time manual test script with concrete URLs. Brevity is the point — this is a wrap-up tool, not a report.

It sits between `/iwol:beads-implement` and `/iwol:beads-pr` and is normally run on a wave's sub-epic. It finds problems and records them; it never fixes them. `/iwol:beads-pr` reads what this skill records and takes it from there.

Read `.claude/iwol-config.md` first. It holds everything specific to this repository: check commands, generated files, local services, the PR reviewer and merge rules. If it's missing, stop and ask the user to run `/iwol:setup`, which creates it.

## Step 0 — Require the epic ID (hard stop)

This skill operates on a specific Beads epic ID — normally a wave's sub-epic.

If no epic ID is present in the conversation or command args, **stop immediately**. Do not guess, do not list epics and pick one, do not infer it from recent git history. Ask the user for the epic ID plainly, then end the turn and wait.

If the ID doesn't resolve to a real bead, say so and stop.

The test runs the code in this session's folder, so for a wave's sub-epic whose branch still exists (`git branch --list 'feature/<sub-epic-id>-*'`), move into the worktree that holds it, where `/iwol:beads-implement` checked it out, as *Entering one* in `../start/worktrees.md` says. Do it before reading any git state, unless this session is already there. A wave started before waves had their own worktrees may be in the main checkout; *Entering one* then stops and tells the user to run from there.

## Step 1 — Gather the epic's actual scope

Load the epic and walk its children (recursively through sub-epics) via `bd show`/`bd dep list` or equivalent. For each closed/done child, read its description, acceptance criteria, and any close notes. Ignore anything still open or blocked — this skill summarizes and tests what's *done*, and should flag incomplete children rather than testing them. The exception is a bead that an earlier session of this skill reopened because of an open `Finding [manual test…]` or `Finding [codebase check]` comment. Its work was done, so include it and mention its open findings.

Then read the sub-epic's latest `Manual test plan` comment. If it still has steps marked `not run` that no later `Re-tested` comment on the sub-epic covers, an earlier session split the test into groups or stopped early, and this session carries on with it. Skip Steps 2 and 3 for the beads the plan already covers, and go to *Continuing a split test* in Step 4.

## Step 2 — Verify against the codebase, not against the beads' claims

Beads describe intent; the working tree is ground truth. For every child bead's claimed changes:

- Confirm the files it claims to touch actually exist and contain the described change (`git diff`, `git log`, `Read` the current file — check committed AND uncommitted state, since work may not be committed yet).
- Do not re-run full test suites here unless something looks contradictory — if a prior orchestration pass (e.g. `beads-implement`) already verified this epic's work, trust that unless the working tree has changed since. Spend effort on catching drift between what a bead *says* happened and what is *actually in the repo* — stale claims, half-applied diffs, files that were reverted after being marked done.
- Note anything that doesn't match: claimed but missing, described differently than implemented, or silently reverted.
- Record each mismatch so the next session can see it: add a `Finding [codebase check]` comment to the bead, in the same format as test failures (Step 4) with `Re-test: none` unless a test step covers it, and reopen the bead with the mismatch as the reason. Otherwise `/iwol:beads-pr`, which rebuilds its picture only from `bd` and git, would open a PR for work already known not to match its bead.

Keep this step internal — its output feeds the summary and the test plan, but don't dump raw diffs or file listings on the user.

## Step 3 — Give a brief summary

Output format, kept short:

- One line per completed child bead: what changed, in plain terms (not the bead's title verbatim).
- One line flagging any verification mismatches found in Step 2, if any. If none, say so in a half-line, don't skip it silently.
- Nothing else. No architecture explanations, no restating acceptance criteria, no "here's what we'll do next" preamble beyond a one-line transition into testing.

If the epic has open/blocked children, name them in one line so the user knows the summary is partial.

## Step 4 — Build the test plan, but reveal it one step at a time

Before showing anything, silently plan the full sequence of manual test steps covering every completed child's user-observable behavior. Write, split and run the steps by `test-steps.md` in the `summary-and-test` skill's folder, next to this one (`../summary-and-test/test-steps.md`): how each step reads, groups of about 10 steps with one group per session, and one step per message. Read it before planning. The rest of this step covers what's specific to a wave: naming the session, recording failures, saving the plan, and continuing a split test.

Save the plan before sending the first step (see *Saving the plan*), so the next session can pick it up.

### Naming the session

Before sending step 1, name the session `[<id>] summary and test`, where `<id>` is the ID this skill was run with, normally the sub-epic. In a plan with groups, add the group this session runs: `[<id>] summary and test-<group>`, e.g. `[acme-v1.6] summary and test-2`. Rename it with the Claude desktop app's session-title tool (`set_session_title` with the session `self`; load it with ToolSearch if it's deferred). Where there's no such tool, as in the CLI, skip this without mentioning it. If the app asks the user to approve the new title, because they named the session themselves, leave it to them.

### Recording a failure

Record each failure as a `Finding` comment on the bead whose behaviour the step covers (`bd comments add <bead-id> -f <file>`). If the step covers several beads, pick the one whose files are most likely at fault, or ask. Use this format, where `<step>` is `<k> of <total>`, or `<group>.<step>` for a step the plan numbers that way:

```
Finding [manual test, step <step>]: <one-line summary>
Where: <screen and URL, or file:line if known>
Expected: <what the step said should happen>
Actual: <what the user saw>
Re-test: Step <step> — <title>: start <URL, or device and screen or deep link>; setup <what you did first, or "none">; <action>; <expected result>
Context: manual test session <date>
```

Then reopen the bead, so `bd` doesn't show it as done: `bd reopen <bead-id> --reason "Manual test step <step> failed: <one-line summary>"`. The one line you tell the user names the bead it went on.

If the user wants it fixed right away, explain that fixes run through `/iwol:beads-pr`. That way the fix gets a new sub-agent, central verification and a re-test. Offer to end testing here so they can start it.

### Saving the plan

Save the whole plan as one comment on the sub-epic, starting `Manual test plan (<date>)`. Save it once before the first step, with every step `not run`, and again whenever a session's testing ends (after its group's last step, or when the user stops early), with the results so far. A step that a later `Re-tested` comment on the sub-epic covers counts as `pass (re-tested <date>)`, so a newer plan keeps that result. The latest one is always the current state. Give one line per step: `Step <step> — <title> [<bead-id>, …]: start <URL, or device and screen or deep link>; setup <what you did first, such as the SQL you ran, or "none">; <action>; <expected> — pass | fail (finding <id>) | blocked | not run`. In a plan with groups, put a `Group <g> of <G> — <title>` line before each group's steps. End with a `Beads covered: <bead-id>, …` line listing every completed bead the plan accounts for, whether through steps or checked without one (docs, tests, changes with nothing to see on screen, things you verified yourself). List every bead whose behaviour the step depends on. Each bead gets its own PR, and `/iwol:beads-pr` uses these lines to work out which steps to re-run after fixing a bead, and to write each PR's description.

Then give a one-line close: how many of this session's steps passed, failed or were blocked, and the next step, in a new session. While groups remain, that's `/iwol:beads-summary-and-test <sub-epic id>` for the next group (name it). After the last group, it's `/iwol:beads-pr <sub-epic id>`. Nothing more unless the user asks for detail.

### Continuing a split test

When the latest plan still has steps `not run` that no later `Re-tested` comment covers, carry on with it rather than planning again:

- First look for beads the plan doesn't cover: completed children, as Step 1 gathers them, that its `Beads covered` line doesn't list, such as a bead that was `blocked` when the plan was made and has closed since. If the plan has no `Beads covered` line (it was saved before this format), count the beads its step lines name as covered. Run Step 2 on them, plan steps for those with something to see as a new group at the end, add them all to `Beads covered`, and save the plan again before going on. If the plan's steps have plain numbers (a one-group plan, or one saved before groups), don't renumber them: earlier `Finding`, `Re-test needed` and `Re-tested` comments name steps by those numbers, and `/iwol:beads-pr` looks them up in the latest plan, so a changed number sends a re-test to the wrong step or drops it. Keep the existing steps' plain numbers and number only the new group's steps `<g>.<step>`. Or, if the plan stays at about 10 steps, add the late steps to its one group with the next plain numbers, and say the new total.
- Run the first group that has steps `not run` that no later `Re-tested` comment covers, starting from the first of them. Skip steps a later `Re-tested` comment covers: `/iwol:beads-pr` has already re-tested them. Do each step's recorded setup again before sending it. If the step relies on something the group's earlier steps set up, set that up again too, yourself where you can.
- Before sending a step, check it against the current code: screens, labels and flows may have changed since the plan was saved, for example in a fix round. The plan keeps only one line per step, so write each one out in full in the format `test-steps.md` gives.
- Open with one line: the group this session runs, and what the earlier groups found (passes, failures, and the beads the findings went on).
- Keep the plan's groups and numbering. If the user wants a new plan instead, start again from Step 2.

## Non-goals

- Don't run or implement beads — that's `beads-implement`.
- Don't write or edit code as part of this skill, even to fix something the user finds while testing. Record a finding; `/iwol:beads-pr` runs the fix.
- Don't commit, open a PR, or close beads. That's `/iwol:beads-pr`.
- Don't dump the full test plan at once — one step, wait, next step.
- Don't run more than one group of a split test in a session unless the user asks. The next group starts in a new session, from a small context.
- Don't split a step into a list of actions and a separate list of expected results, and don't shorten steps on the assumption that the user knows the app.
- Don't pad the summary. If a one-liner covers a bead, use one line.
