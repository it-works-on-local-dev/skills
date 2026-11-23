---
name: grill-me
description: Interview the user relentlessly about a plan or design until reaching shared understanding, resolving each branch of the decision tree. Use when user wants to stress-test a plan, get grilled on their design, or mentions "grill me".
---

Interview me relentlessly about every aspect of this plan until we reach a shared understanding. Walk down each branch of the design tree, resolving dependencies between decisions one-by-one. For each question, provide your recommended answer.

Ask the questions one at a time.

If a question can be answered by exploring the codebase, explore the codebase instead.

## How to ask

- **Show progress on every question.** Number each one against the total, e.g. "Question 3 of ~10". Plan the full list up front so the total is real. If the total changes because an answer opens or closes a branch, say so and give the new total. I always want to know how much questioning is left. When the grilling is about something to build, how it ships is one of the questions: as waves of beads, one PR per bead, through `/iwol:beads-decompose-plan`, or as one PR through `/iwol:start`, with your recommendation.
- **Keep each question under 300 words**, in plain terminology. Define a specialised term in one sentence the first time it appears, especially for infrastructure and cloud topics (Azure, Redis, networking). If I do not understand something, I will ask you to elaborate; do not pre-empt that with long explanations.
- **Be honest and constructive.** I am a front-end developer; many plans touch areas outside my expertise. If an idea is a different kind of thing than I think it is, say so plainly. Name the trade-offs, recommend one option, and explain why. Do not rubber-stamp.
- **Offer options as a short labelled list** (A, B, C) with my recommended option stated explicitly, then ask for agreement.
- **Track two running lists** and hand both over at the end: the decisions we reached, and the questions that must go to someone else (a stakeholder, an architect) because neither of us can answer them. Then write the plan to `plans/<slug>.md` in the main checkout, the slug being the plan's title in lowercase words joined by hyphens. The main checkout is `<main checkout>` as `../start/worktrees.md` defines it, not this session's folder when that is a worktree: the plan is built from a new session started in the main checkout, and an untracked file in a worktree doesn't exist there. Leave it uncommitted. Give it these seven sections, in this order:
  - **Context:** what the grilling found and settled by reading the repository, and why the change is wanted.
  - **How it ships:** the route, as waves of beads through `/iwol:beads-decompose-plan` or one PR through `/iwol:start`, with the PR title and the files it touches.
  - **Acceptance criteria:** the checks and observable behaviour that mean it's done.
  - **Decisions:** the decisions we reached, each with why and what it was chosen instead of.
  - **Changes:** what changes, file by file. When the plan ships as more than one PR, one section per phase instead.
  - **Out of scope:** what is deliberately not being done.
  - **Open questions:** the questions that must go to someone else, each naming who must answer it. "None" when nothing was routed.

  If a file of that name already holds another plan, add a word to the slug; if it holds this plan from an earlier grilling, update it. Name the file's absolute path, then offer the next step, in this same session, as a lead-in line and the command in a `text` block:

  ```text
  /iwol:digest-plan
  ```

## Before the first question

Research first. Explore the codebase, related branches, wiki pages and any links I provide, and check the current state of any framework or vendor feature involved. Open with a short summary of what that research settles, so questions are only about what is genuinely open.
