# Manual test steps

How to write, size and run a guided manual test. `/iwol:summary-and-test` and `/iwol:beads-summary-and-test` both follow it. Each of them says what it tests, where a failure is recorded, and where the plan is saved.

## Writing a step

Each step must be self-contained.

Write every step for someone using the app for the first time. The user may never have seen these screens before, so nothing can rely on knowing where things are or what they're called.

- **One numbered list per step, each action with its expected result.** Each item is a single action, followed on the next line by what the user should see after it, starting **Expect:**. Never put the actions in one list and the expected results in another: the user ends up going back and forth between the two lists and repeating actions.
- **A concrete starting point** as the first item. For a web page, a link to it: resolve the actual dev server port from where `.claude/iwol-config.md` says ports are documented, never guess. If a change touches multiple apps, use each app's own port. For a mobile app, the device (simulator or emulator) and how to reach the starting screen from the app's first screen. A screen that can only be opened from a link, such as one a shared link or a dev link opens, starts with the deep-link command that opens it instead, one `bash` block per device, naming the screen it opens.
- **Exact names for everything on screen.** Pages, screens, tabs, panels, sheets, cards, buttons, fields, menus and toggles get the label the user sees, in bold, and where it is when that isn't obvious (top right, at the bottom of the sheet, under **Members**). Take the labels from the code that renders them, not from how a bead, a PR or a commit describes them. Never use internal names in the instructions: components, files, routes, bead IDs. A route inside a deep-link command is fine.
- **No shortcuts.** One action per item: every tap, every field, and the exact text to type, in backticks. Never write "as before", "same as step 3", "repeat for the others", "fill in the form", or "go to Settings" without saying how to get there. If the step starts on a screen an earlier step reached, give the way there again.
- **Setup.** Do the setup you can do yourself (start a server, load data, run SQL) before sending the step, and say it's done. Record it in the plan when the skill saves one, so a later session can do it again. Setup only the user can do, such as signing in or opening a second browser window, comes first in the step as items of its own.
- **Commands** each go in their own fenced code block tagged `bash`, so the user gets Run and Copy buttons: one command per block, no `$` prompt, no output inside. Never give a command the user has to run only in backticks.
- **URLs** the user opens in a browser are always Markdown links, never in backticks. A URL that's part of a command stays in that command's `bash` block.
- **Expected results** are concrete and observable: the exact text, value, screen or change, specific enough that a mismatch is unambiguous. Not a restatement of the acceptance criteria.

A step looks like this:

````
**Step 2 of 6 — <short title>**

1. <Setup only the user can do, if any>:
   ```bash
   <command>
   ```
   **Expect:** <what shows when it's ready>.
2. Open [<url>](<url>) in <browser>.
   **Expect:** the **<page title>** page, showing <what's on it>.
3. Tap **<button label>** <where it is on the screen>.
   **Expect:** <what appears or changes, with its exact text>.
4. In the **<field label>** field, type `<exact text>`.
   **Expect:** <what the field shows>.
5. Tap **<button label>** <where it is>.
   **Expect:** <the result>.
````

## Keep each session's test short

Keep the plan as short as coverage allows. Merge related checks into one step, and check yourself whatever doesn't need the user's eyes (test runs, database queries, logs) instead of making it a step.

If the plan still runs to more than about 10 steps, split it into groups of about 10 steps or fewer, and run **one group per session**. A 33-step test in one sitting wore the user out. Group steps logically: the same feature or screen flow, the same setup or test data. Keep steps that build on each other, where one creates what the next uses, in the same group. Each group runs in a new session, so it has to start from a state that session can set up again. Order the groups the way a user would move through the app.

In a plan with groups, number the steps `<group>.<step>`, such as `2.3`, and label each step with its group too, e.g. "Group 2 of 4 — Review items · Step 3 of 8 — Change an item's price". A plan of one group keeps plain numbers, and its existing steps keep them if it gains a group later.

Before step 1, tell the user in one line how many groups and steps there are, and which group this session runs.

## Running the steps

Do NOT print the whole plan up front. Send only step 1. Wait for the user's reply (pass, fail, or feedback) before sending step 2, and so on, in the same message-per-step rhythm all the way through. This is a hard requirement, not a default — even if you're confident the user wants the full list, hold it back.

Since the full plan is built silently before the first step is sent, its total length is already known — label every step with its position and the total (the group's total, in a plan with groups), plus a short title, e.g. "Step 1 of 6 — Confirm a cart id appears in local storage". Keep the total fixed once testing starts; if a mid-plan finding (e.g. a bug the user reports) leads you to add a step, say so explicitly ("adding a step — now 7 total") rather than silently changing the count.

When the user reports a failure or unexpected result on a step, do not jump to fixing code. Acknowledge it and ask only what's needed to understand the mismatch. Then record it the way the skill says, tell the user in one line where it went, and move on to the next step. If the failure blocks later steps (the screen won't load, the app crashes), mark those steps as blocked instead of sending them.
