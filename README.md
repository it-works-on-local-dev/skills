# skills

The `iwol` plugin for [Claude Code](https://claude.com/claude-code), and the marketplace that serves it.

## What it is

- **The `iwol` plugin** (`plugins/iwol/`) gives Claude Code a way of working from plan to release:
  - the beads workflow, which turns a plan into work items and runs them through parallel sub-agents, manual testing and one pull request per item: `/iwol:beads-decompose-plan`, `/iwol:beads-implement`, `/iwol:beads-summary-and-test` and `/iwol:beads-pr`;
  - `/iwol:grill-me`, which stress-tests a plan or design before you build it;
  - the PR and release flows for everything else: `/iwol:start`, `/iwol:create-pr`, `/iwol:summary-and-test` and `/iwol:create-release`;
  - `/iwol:setup`, which adopts the plugin in a repository;
  - eight bead worker agents, the sub-agents `/iwol:beads-implement` hands each work item to.
- **The `iwol` marketplace** (`.claude-plugin/marketplace.json`), which lists that one plugin, so Claude Code can install it straight from this repository.

The name `iwol` abbreviates it-works-on-local-dev, the organisation the plugin was written in. It stays `iwol` so that nothing that refers to it has to change: the command prefix `/iwol:`, the agent prefix `iwol:`, and the key `iwol@iwol` that turns the plugin on in a repository's settings.

The beads skills keep their work items in [Beads](https://github.com/steveyegge/beads), so they need its `bd` CLI. The PR and release flows use the GitHub CLI, `gh`. `/iwol:setup` reports either one when it's missing.

## Install

Register the marketplace on your computer:

```sh
claude plugin marketplace add it-works-on-local-dev/skills
```

The repository is public, so this needs no GitHub credentials.

Then turn on auto-update, so your computer receives new releases: in Claude Code, open `/plugin`, go to Marketplaces, select `iwol` and turn auto-update on. Registering the marketplace doesn't turn the plugin on anywhere; a repository does that when it adopts the plugin (next section).

### Developing the plugin

To work on the plugin itself, register the marketplace from a local clone of this repository instead of from GitHub:

```sh
claude plugin marketplace add <path to the clone>
```

The plugin then loads in place: an edit to a skill or agent in the clone is live in every project at its next session, with no copy, install or version bump. In a session that's already running, `/reload-plugins` picks it up.

## Adopt it in a repository

1. In the repository, run:

   ```sh
   claude plugin install iwol@iwol --scope project
   ```

   This writes `.claude/settings.json`, which turns the plugin on for everyone who clones the repository and has the marketplace registered. Commit it. Keep `--scope project`: without it, the plugin is turned on in every project on your computer instead. Never add `extraKnownMarketplaces` to that file: the marketplace's source differs from computer to computer (GitHub on most, a local clone for someone developing the plugin).

2. In Claude Code, in the repository, run `/iwol:setup`. It:
   - asks which reviewer the repository's pull requests use: the public Claude review workflow (see [The review workflow](#the-review-workflow)), another review bot, or none;
   - writes `.claude/iwol-config.md`, the file every iwol skill reads for the repository's checks, reviewer, CI, merge method and releases;
   - writes the review workflow stub, when you chose the public Claude review workflow;
   - reports what's still missing, such as `bd`, `gh`, the ignore rule for `.claude/worktrees/`, and the review workflow's secrets;
   - lists local copies of the plugin's skills and agents that the repository should delete, since the plugin now provides them.

   Run it again whenever the repository's checks, CI or reviewer change, to re-check the setup.

A repository that has adopted the plugin holds only a settings file and a config file, plus a workflow stub when it uses the public Claude review workflow. Everything else comes from the plugin.

## What the plugin ships

Naming rule:

- A skill folder `plugins/iwol/skills/<name>/` is the command `/iwol:<name>`.
- An agent file `plugins/iwol/agents/<name>.md` is the agent `iwol:<name>`.

### Skills

| Command | What it does |
| --- | --- |
| `/iwol:beads-decompose-plan` | Turns an implementation plan, spec or design doc into a graph of beads: one epic, sub-epics as strictly sequential waves whose beads all run in parallel, and each bead labelled with agent fit, effort and task kind. |
| `/iwol:beads-implement` | Runs a wave: fans its beads out to parallel worker agents, then verifies all the results centrally. Also runs fix rounds on beads with open findings, usually for `/iwol:beads-pr`. |
| `/iwol:beads-summary-and-test` | Summarises a wave, checks it against the code, and walks you through manual testing one step at a time, splitting a long test into groups run one per session. Failures are recorded on the beads for `/iwol:beads-pr`. |
| `/iwol:beads-pr` | Takes a tested wave to merged pull requests, one per bead: splits the wave branch into bead branches, opens the PRs, triages review comments and CI failures, sends fixes through `/iwol:beads-implement`, resolves every thread, merges, and closes the wave. |
| `/iwol:grill-me` | Interviews you relentlessly about a plan or design until every branch of the decision tree is resolved. |
| `/iwol:start` | Starts work that isn't a wave in its own named worktree, `.claude/worktrees/<prefix>-<description>` on the branch `<prefix>/<description>`, and moves the session into it. Given the name of a worktree that exists, it moves the session back in. |
| `/iwol:create-pr` | Commits the session's changes and opens a PR against the base branch (draft by default), or pushes follow-up commits to the branch's open PR. Waits for CI, runs the Claude review loop, marks drafts ready, and merges and cleans up the branch the way the repository's config says. |
| `/iwol:summary-and-test` | The same as `/iwol:beads-summary-and-test` for changes that aren't a wave: summarises a branch or PR, checks it against the code, walks you through manual testing step by step, then fixes what failed and re-tests it. |
| `/iwol:create-release` | Cuts a production release by merging the Release PR that release-please keeps open, on your explicit go-ahead, after showing its version and changelog and checking new database migrations for destructive statements. Then watches the tag and the GitHub Release, watches the tag's deploy run when the config names a deploy workflow, and verifies production when the config names a version URL. For repositories whose config describes releases. |
| `/iwol:setup` | Sets up the plugin in a repository, or re-checks one that already uses it: writes `.claude/iwol-config.md`, enables the plugin in `.claude/settings.json`, writes the review workflow stub when the repository uses the public Claude review workflow, and reports what's missing and which local skill or agent copies to delete. |

### Session names

In the Claude desktop app, the bead skills name their session after the bead they run on, and `/iwol:start` after its branch, so the sidebar shows what each session is doing. A CLI session can't be renamed, so it keeps its name.

| Skill | Session name |
| --- | --- |
| `/iwol:beads-decompose-plan` | `[<epic id>] decompose` |
| `/iwol:beads-implement` | `[<id>] implementation`, for the sub-epic or leaf bead it was given; a fix round runs inside a `/iwol:beads-pr` session and keeps that session's name |
| `/iwol:beads-summary-and-test` | `[<id>] summary and test`, or `[<id>] summary and test-<group>` for one group of a split test |
| `/iwol:beads-pr` | `[<sub-epic id>] pr-<n>` for the wave's nth run, counted by the `PR session <n>` comments each run adds to the sub-epic |
| `/iwol:start` | `<prefix>/<description>`, the branch name |

### Worktrees

The Claude desktop app gives each session a worktree with a random name before the session starts. The skills give every piece of work a worktree named after it instead, all under `.claude/worktrees/`, which the repository git-ignores:

| Work | Branch | Worktree | Made by |
| --- | --- | --- | --- |
| A Beads wave | `feature/<sub-epic id>-<title>` | `<sub-epic id>-<title>` | `/iwol:beads-implement` |
| Anything else | `<prefix>/<description>`, with one of `/iwol:create-pr`'s branch prefixes (`feat`, `fix`, `hotfix`, `docs`, `refactor`, `test` or `chore`) | `<prefix>-<description>` | `/iwol:start` |

Start the session for new work in the main checkout (in the desktop app, with the session's worktree option off), then run `/iwol:start <description>` or `/iwol:beads-implement <sub-epic id>`. The skill creates the worktree from the base branch and moves the session into it, so its commands, edits and sub-agents all run there, and the desktop app's diff and file panes follow. The main checkout's branch and files are never touched.

A later session gets back in with `/iwol:start <prefix>/<description>`. `/iwol:beads-implement`, `/iwol:beads-summary-and-test`, `/iwol:beads-pr`, and `/iwol:summary-and-test` given a PR, move into the worktree that holds their branch by themselves; `/iwol:summary-and-test` with no PR tests the folder the session is in. Once the work has merged, `/iwol:create-pr` or `/iwol:beads-pr` checks that nothing would be lost, brings the session back to where it started, and removes the worktree. No skill renames, moves or removes the desktop app's own worktrees.

The shared rules are in [`plugins/iwol/skills/start/worktrees.md`](plugins/iwol/skills/start/worktrees.md).

### Agents

Eight bead workers, `iwol:bead-<model>-<effort>`, for model `opus` or `sonnet` and effort `low`, `medium`, `high` or `xhigh`. `/iwol:beads-implement` dispatches each bead to the worker that matches its `agent:` and `effort:` labels. They're meant only for that: each takes a single bead's text and reports back what it changed.

## Releases and versions

The plugin is released with [release-please](https://github.com/googleapis/release-please). Pull request titles follow [Conventional Commits](https://www.conventionalcommits.org/), and release-please keeps a Release PR open that collects them. Merging the Release PR:

- tags the release `vX.Y.Z`;
- writes the release notes to [`CHANGELOG.md`](CHANGELOG.md);
- bumps `version` in `plugins/iwol/.claude-plugin/plugin.json`.

Claude Code updates an installed plugin when that version changes, and the marketplace entry carries no version of its own, so a computer that registered the marketplace from GitHub receives releases, not every merge to `main`. A computer that registered it from a local clone loads whatever the clone has checked out.

To stay on one release, add the tag to the marketplace source when you register it, for example `claude plugin marketplace add it-works-on-local-dev/skills#<tag>` with a tag such as v1.2.0.

Versions follow semver, and the major number is a signal: a major release, from a PR whose title carries a `!` (such as `feat!: …`), means the repository's config changes shape, and running `/iwol:setup` again migrates it.

## The review workflow

The plugin's PR skills read the reviews on a pull request and act on them. They work with any of three set-ups, which `/iwol:setup` asks about:

- the public Claude review workflow in [`it-works-on-local-dev/code-review`](https://github.com/it-works-on-local-dev/code-review);
- another review bot;
- no reviewer.

The workflow's own README covers calling it, creating the GitHub App it posts as, and its versioning: its v1 tag moves only to compatible releases. You don't need to write the calling workflow yourself: `/iwol:setup` writes the stub for you when you choose this reviewer.

## Checks this repository runs on itself

CI runs, on every pull request and on every push to `main`:

- `actionlint` on the repository's workflows;
- `claude plugin validate .`, for the marketplace;
- `claude plugin validate plugins/iwol`, for the plugin.

## History

The plugin's history before this repository, including its earlier pull requests, is in the private `it-works-on-local-dev/tooling` repository.

## License

[MIT](LICENSE).
