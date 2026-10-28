# Skill frontmatter `model:` / `effort:` — does it enforce a whole run?

Follow-up from PR 34, which added the README's `Model` and `Effort` columns as recommendations only and left enforcement through skill frontmatter to test. Tested 2026-10-28 on Claude Code 2.1.308.

## Answer

No form covers a whole run. `effort:` in a `SKILL.md` applies to **the turn that invokes the skill and nothing after it**; the next prompt runs on the session's effort again. `model:` reaches even less: it applies to that one turn only when you type the slash command, and **not at all when Claude invokes the skill through the Skill tool**, which is how iwol's skills hand off to each other. Where they apply, both replace the session's setting rather than acting as a floor, so they lower a stronger session as readily as they raise a weaker one.

**Recommendation: set nothing.** Keep the README's `Model` and `Effort` columns as the only guidance and leave every `plugins/iwol/skills/*/SKILL.md` without `model:` or `effort:`.

## What the docs say now

- Skills frontmatter reference, `model`: unchanged since 2026-10-26. The override "applies for the rest of the current turn" and the session model resumes on the next prompt; `inherit` keeps the active model.
- Skills frontmatter reference, `effort`: still "when this skill is active. Overrides the session effort level."
- Model config, *Adjust effort level*: frontmatter effort "applies when that skill or subagent is active, overriding the session level but not the environment variable". The same page's *Effort level after a fallback* section describes it as set "for one turn", which is what the experiment shows.

So the `model` text holds only for a typed slash command. When Claude invokes the skill through the Skill tool, the override doesn't apply even for the current turn, and the docs don't say so. The `effort` text ("while active") reads as if it lasted for the whole skill run; in practice it lasts one turn, like `model`.

## How it was tested

Throwaway skills under `.claude/skills/` (removed afterwards). Each skill ran `true` with Bash once and replied in one line. Later turns asked for the same thing without the skill.

- **One process, several turns.** `claude -p --input-format stream-json --output-format stream-json`, with each user message sent only after the previous turn's `result` event. That makes the turns real follow-up prompts in one session, not resumed processes.
- **Model readout:** `message.model` on every streamed assistant message, which is the model the API ran. Also the result's `modelUsage`. The model's own claim about its ID was unreliable: in one run it named Opus on a turn the API ran on Sonnet.
- **Effort readout:** the `effort.level` field that `PreToolUse` and `Stop` hooks receive, logged by a hook added with `--settings`. It reports the level in effect at that point in the turn. The `${CLAUDE_EFFORT}` substitution in the skill body confirmed the level the skill saw when it loaded.
- **Interactive check:** the `model: sonnet` + `effort: low` case was repeated in the TUI, driven by `expect`, with the same hook logger.

The session ran on `--model opus --effort xhigh` unless noted.

## Results

| Skill frontmatter | Invoked by | Turn 1 (skill) | Turn 2 | Turn 3 |
| --- | --- | --- | --- | --- |
| `model: sonnet`, `effort: low` | `/fmtest-both` | Sonnet 5.5, `low` | Opus 5.5, `xhigh` | Opus 5.5, `xhigh` |
| `effort: low` | `/fmtest-effortonly` | Opus 5.5, `low` | Opus 5.5, `xhigh` | Opus 5.5, `xhigh` |
| `model: inherit`, `effort: low` | `/fmtest-inherit` | Opus 5.5, `low` | Opus 5.5, `xhigh` | Opus 5.5, `xhigh` |
| `model: opus`, `effort: xhigh` (session `sonnet`/`low`) | `/fmtest-raise` | Opus 5.5, `xhigh` | Sonnet 5.5, `low` | — |
| `model: opus`, `effort: xhigh` (session `sonnet`/`low`) | Claude, through the Skill tool | **Sonnet 5.5**, `xhigh` | Sonnet 5.5, `low` | Sonnet 5.5, `low` |
| `effort: low` (`CLAUDE_CODE_EFFORT_LEVEL=high`, no `--effort`) | `/fmtest-effortonly` | Opus 5.5, **`high`** | Opus 5.5, `high` | — |
| `model: sonnet`, `effort: low`, interactive TUI | `/fmtest-both` | `low` (reply named Sonnet) | `xhigh` | `xhigh` |

What each form covers:

- **`model:`** lasts one turn, and only when you type the slash command. When Claude invokes the skill through the Skill tool, the model doesn't switch at all, even in the turn that loaded it. Most iwol skills are invoked that way at least some of the time, for example when beads-pr hands off to beads-implement or when you ask in plain words.
- **`effort:`** lasts one turn, by either invocation path. It beats `--effort`, in both directions, but loses to `CLAUDE_CODE_EFFORT_LEVEL`. A saved level (from `/effort` or settings) wasn't tested.
- **`model: inherit`** behaves exactly like leaving `model:` out. With `effort:` set, it gives the same one-turn effort override as `effort:` alone.

Does it downgrade a stronger session?

- **`model:`** does, for that turn when the slash command is typed: an Opus session ran its skill turn on Sonnet. A Fable session would drop to Opus the same way if a skill said `model: opus`, and the frontmatter has no "else" form to express "Fable if you have it, else Opus".
- **`effort:`** does too. An `xhigh` session ran the skill turn at `low`. A skill set to its README floor, say `high`, would lower the first turn of an `xhigh` or `max` session. Frontmatter sets a level; it can't express "at least".

Gap: the TUI check covers effort only. The nested session doesn't persist its transcript, so the TUI turns' API model couldn't be read. The turn-1 reply named Sonnet, the docs state the one-turn `model` behaviour, and the headless runs measured it directly.

## Why "set nothing"

- **Neither form reaches most of a run.** Most iwol skills are multi-turn (grill-me, summary-and-test, beads-pr) or long enough that you reply in the middle (setup, create-pr, create-release). Setting a value would move the first turn only, which is the cheapest part of the run.
- **Both forms can downgrade.** The README's values are floors ("or higher"), and the frontmatter can't express a floor. Setting `effort: high` on create-pr would lower a session you'd deliberately run at `xhigh`, and setting `model: opus` on beads-pr would take a Fable session off Fable.
- **`model:` doesn't even apply when Claude invokes the skill.** That makes it unpredictable across the plugin's own hand-offs.
- **The only form that can't downgrade is a no-op.** `effort: xhigh` on the three `xhigh` skills (beads-decompose-plan, beads-pr, grill-me) can only lower a `max` session, but it still affects only turn 1 and never covers the turns where those skills do their judgement-heavy work. Not worth carrying.

## If enforcement is still wanted later

The `${CLAUDE_EFFORT}` substitution reports the session's actual level when the skill loads; it showed `high` under `CLAUDE_CODE_EFFORT_LEVEL=high`. A skill could compare that to its floor and, when it's lower, tell the user to run `/effort <floor>` before continuing. Changing the level that way lasts the whole session and never downgrades a stronger one. That's a nudge, not enforcement, and it has no model equivalent. It isn't tested beyond the substitution itself, so it needs its own plan.
