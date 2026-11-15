---
name: plan-digest
description: Compress an implementation plan into the few things the user needs to know before approving it — outcome, blast radius, decisions, risks, verification, go/no-go. Normally runs right after /iwol:grill-me, on the plan grilling just produced, and before the plan is built, by /iwol:beads-decompose-plan as beads or by /iwol:start as one PR. Use when the user wants a plan summarised, a TL;DR of a plan, "just tell me what this does", "digest this plan", or "/iwol:plan-digest the plan we just built".
---

# /iwol:plan-digest

Compress the plan into a digest the user can read in under a minute. The user should never have to read the full plan to know what it does, what it touches, where the risk is, and whether to approve it. In this plugin the plan then goes on to `/iwol:beads-decompose-plan` (waves of beads) or `/iwol:start` (one PR), so the digest is the go/no-go before building.

Order by importance, most load-bearing first. Assume the user reads top-to-bottom and stops the moment they are satisfied.

Plans may touch backend, infrastructure, data, cloud, networking, CI/CD, or other areas outside the user's expertise. Use plain terminology; define a specialised term — for example Redis, Azure, migration, feature flag, or invariant — in one sentence on first use. Do not pad: a three-line section stays three lines.

## Find and verify the plan

"The plan we just built" means the plan produced earlier in this session, normally by `/iwol:grill-me`. Look for it in this order, and do not ask the user to paste it:

1. The file `/iwol:grill-me` wrote in this session, `<main checkout>/plans/<slug>.md` (`<main checkout>` as `../start/worktrees.md` defines it), whose absolute path it named. Read and edit it at that path: in a worktree session, a bare `plans/<slug>.md` points into the worktree, not at the file the hand-off builds.
2. A file the user points at, such as `docs/rfc.md`, wherever it is.
3. Failing both, the plan in the conversation.

When the plan is in a file, read the file and verify the digest against it, not against the conversation: the file is what gets built.

If the plan references files, branches, wiki pages, linked docs, configs, or APIs, inspect what is accessible before summarising.

Before writing the digest, re-read the full plan and verify every claim against it. Do not invent files, decisions, alternatives not taken, risks, tests, or rollout steps. If the plan is vague or omits something important, say **"the plan does not specify"** rather than filling the gap.

If genuinely no plan-shaped artefact exists in context, say so in one line and stop.

## Output budget

Target **150–300 words** total. Hard cap: **25 lines excluding headings and the next-command hand-off** (see *Next command*).

If the plan is large, compress harder rather than exceeding the cap. Prioritise the decisions and risks that affect correctness, reversibility, users, data, security, or other systems.

## The digest

Use these sections, in this order.

### 1. In one sentence

What will be different when this is done, in observable user-, system-, or behaviour-level terms. Not implementation steps.

### 2. Blast radius

Files, modules, services, data, integrations, or areas touched, sorted by how much they change. Limit to the top 5–8 areas; if there are more, say "plus minor changes across N files".

Mark shared, critical-path, hard-to-reverse, security-sensitive, or user-facing items with ⚠️.

A one-line change that flips an invariant outranks a 500-line mechanical move — call it out.

### 3. Decisions locked in

The consequential design choices made in the plan (during grilling, if it was grilled). Limit to **3–5 decisions**.

For each:

- **Decision** — state the chosen option first, bold the load-bearing ones.
- **Why** — one line.
- **Instead of** — one line naming the alternatives not taken. Never label this "Rejected": that reads as a verdict on the decision itself.

Use the same nouns in the decision and in "Instead of", so the reader can pair them at a glance. Example:

> **Seed orders as front-end fixtures.** Why: there are no order tables to seed, and the backend seed tests would break. Instead of: seeding them as SQL now, or building the tables ourselves.

Omit trivial choices — naming, formatting, minor refactors — unless they affect behaviour, maintainability, reversibility, or another system.

### 4. Risks, assumptions & scope

Use concise labelled bullets. Do not bury assumptions inside generic risk language.

- **Risk:** what could break, regress, or be difficult to reverse.
- **Assumption:** what the plan depends on being true.
- **Out of scope:** what is deliberately not being done.

A digest that reads all-green reads as unread. If there are no material risks, say why in one line.

### 5. Proof it works

One to three bullets maximum, based only on what the plan specifies or what is verifiable in the repo:

- Tests added or updated, using the repo's existing test approach.
- Manual verification steps, if automated coverage is insufficient or not planned.
- Rollout implications: feature flag, migration, deploy order, backward compatibility, or "not covered by plan".

If the plan has no verification strategy, flag that explicitly. Do not assume a particular test framework or test type.

### 6. Still open

`/iwol:grill-me` ends with two lists: the decisions reached, and the questions it routed to someone else (a stakeholder, an architect) because neither the user nor Claude could answer them. It writes the routed questions to the plan file's Open questions section, so read them from there; when there is no plan file, fall back to grill-me's list in the conversation. List each routed question here as an open item, naming who has to answer it. Do not invent an answer for it: grilling already said nobody in the conversation can. A routed question that affects correctness, reversibility, users, data, security or another system pushes the verdict to "inspect <that question> first".

Apart from those, this section should be **empty** for a plan `/iwol:grill-me` worked on in this session, because grilling should have resolved every material fork. If a fork that neither list covers genuinely remains unresolved, put this warning at the very top of the digest:

> ⚠️ **Grilling missed this:** <question>

Then give your recommended answer and why.

For a plan that wasn't grilled in this session, grilling didn't run, so never say it missed anything. List the plan's unresolved forks here as open items under "The plan leaves open:", each with your recommended answer and why, and let a material one push the verdict to "inspect <that fork> first", as the routed questions do.

In either case, do not force an unresolved question into "Decisions locked in" or disguise it as a risk.

## Formatting rules

- **Bold the conclusions.** Headers plus bold text must be enough to understand the change.
- **No step-by-step implementation.** Include WHAT, WHY, and WATCH-OUTS only; the user opens the full plan for the HOW.
- **Separate fact from inference.** If you infer something the plan does not state, label it as inference.
- **Do not hedge instead of deciding.** If the evidence supports a recommendation, make it; if it does not, name the missing information.
- **Keep every section as short as its content deserves.**

## Verdict

End the digest with exactly one line in this format:

**Size:** trivial / contained / significant / sprawling · **Risk:** low / medium / high · **Recommendation:** approve as-is / inspect <specific thing> first / do not approve yet

Approve as-is only if the outcome, blast radius, risks, and verification approach are clear and acceptable for the size of the change. Otherwise name the single most important thing for the user to inspect before approving.

## Next command

The verdict line is the digest's last line. Only "approve as-is" is followed by a hand-off, outside the 25-line cap: the next command, to run in a new session started in the main checkout, with the plan file's path in it, so the builder works from the approved file rather than from a conversation that also holds every option turned down.

"Inspect <thing> first" and "do not approve yet" end with the verdict line alone: the verdict already names what to look at, and a Copy button under a warning invites pasting past it. When the user later says the flagged item is settled, edit the plan file to match, asking first when it's a file `/iwol:grill-me` didn't write, such as one the user pointed at. Then re-check the plan and give a new verdict line: settling the item the verdict named doesn't make the plan approvable by itself. Give the command as below only if the new verdict is "approve as-is".

**Pick the route.** Use the one the plan's *How it ships* section states. If the plan doesn't state one, pick by your Size rating: trivial or contained goes to `/iwol:start`, significant or sprawling to `/iwol:beads-decompose-plan`. Then say in the lead-in that you picked by size, and name the other route in prose after the block. Give one block, never both.

**Give the command** as the lead-in "Next, in a new session started in the main checkout:" and the command in a `text` block with the real path. For one PR:

```text
/iwol:start plans/<slug>.md
```

For waves of beads:

```text
/iwol:beads-decompose-plan plans/<slug>.md
```

A plan file the user pointed at elsewhere, such as `docs/rfc.md`, is used as it is, with its own path in the command. When this session isn't in the main checkout (the two absolute paths `git rev-parse --path-format=absolute --git-dir --git-common-dir` prints differ), give that file's absolute path instead: the command runs in a new session in the main checkout, where a relative path names a different copy of the file, or none.

**No plan file.** When the approved plan exists only in the conversation, give no block. Say in one line that the plan isn't in a file, and offer to save it to `plans/<slug>.md` in the main checkout, where grill-me writes it, in grill-me's shape: the seven sections its *How to ask* lists, in `../grill-me/SKILL.md`. Never write it without a yes. On a yes, write it, with "the plan does not specify" in any section the plan doesn't fill, then give the command as above.
