# A queued CI job on self-hosted runners

How to tell why a gating CI job is still `queued` **under `self-hosted`** (the config's `Runs on:` line), and how to recover a run that no runner will take. `/iwol:create-pr` follows it from its *CI check* section once its wait for a run ends with a gating job still queued. `/iwol:beads-pr` follows it once from *A queued gating job* under its *Step 1 — Read the state*.

The two skills differ in one way that matters here: create-pr waits and polls, while beads-pr reads state once per run and never waits. Where the steps below differ for them, they say what **a skill that waits** and **a skill that reads once** do.

**Under `hosted`**, none of this applies: a queued job is only waiting for GitHub's hosted runners, and re-running would only queue it again.

## The job's labels

A gating job still `queued` may be waiting for one of the repository's own runners; the config's `Runs on:` line says how to tell. The run itself can show `in_progress`, because other jobs, such as a job that picks a runner, already ran. Take the job's labels from the queued job itself, where GitHub has already resolved its `runs-on` (the workflow file may hold only an expression, such as one reading a runner-picking job's output):

```
gh api --paginate repos/{owner}/{repo}/actions/runs/<run-id>/jobs --jq '.jobs[] | select(.status == "queued") | "\(.name) \(.labels | join(","))"'
```

## The runners that could take it

A runner can take the job only when it carries all of the job's labels, compared without regard to case, as GitHub does. Check the runners registered to the repository itself, and, when the repository's owner (`<owner>`, from `gh repo view --json owner --jq .owner.login`) is an organisation (`gh api repos/{owner}/{repo} --jq .owner.type` prints `Organization`), the organisation's runners too; a personal account (`User`) has no organisation runners, so check only the repository's. In each command, `<labels>` is the job's labels as a JSON array, such as `["self-hosted","linux"]`. Listing an organisation's runners needs a token that can read its self-hosted runners: an organisation admin's classic token with the `admin:org` scope (a default `gh auth login` doesn't grant it), or a fine-grained token with "Self-hosted runners: Read".

```
gh api --paginate repos/{owner}/{repo}/actions/runners --jq '.runners[] | select((<labels> | map(ascii_downcase)) - [.labels[].name | ascii_downcase] == []) | "\(.name) \(.status) busy=\(.busy)"'
gh api --paginate orgs/<owner>/actions/runners --jq '.runners[] | select((<labels> | map(ascii_downcase)) - [.labels[].name | ascii_downcase] == []) | "\(.name) \(.status) busy=\(.busy)"'
```

## What the listings show

- **Any matching runner `online`** in either list, even if the other listing errored: the job is queued behind other work, since a busy runner still counts as online, and re-running would only queue it again. A skill that waits waits again, and tells the user if it's still queued after a second wait. A skill that reads once reports "CI queued behind other work" and doesn't wait for it: its next run reads the job again.
- **None `online`**, and every listing that applies ran without an error: the job waits until a runner is back, or until GitHub cancels it after 24 hours, so don't wait it out. When every matching runner listed is `offline`, recover the run (*Recovering the run* below). When no matching runner is listed at all, one listing doesn't settle it, since an ephemeral runner is briefly unregistered between jobs:
  - **A skill that waits** lists the runners again a minute later. If none is listed again, recover the run; otherwise go by what the second listing shows.
  - **A skill that reads once** can't list them again later, so it asks the user whether the runner's machine is up, as for a listing that errors (next bullet).
- **A listing command errors** and no other list shows a matching runner `online` (a 403 when the token can't read runners): you can't tell a busy runner from an offline one, so don't guess. Tell the user the job is queued and ask whether the runner's machine is up. If it is, the job is queued behind other work: a skill that waits waits again, and a skill that reads once reports the job as queued and doesn't wait for it. If not, recover the run. Never suggest widening the `gh` token's scopes, since `admin:org` grants full organisation admin to read one list.

## Recovering the run

Recover it per the documentation the config's `Recovering a stuck run:` line points to (if it says `none`, the steps here are all there is): cancel the run, then re-run all its jobs (`gh run rerun <run-id>`, not `--failed`) so a job that picks a runner, where the repository has one, runs again and can fall back to hosted runners. `--failed` would keep the runner it picked the first time, and the job would wait for it again.

- **A skill that waits** recovers the run as soon as the listings call for it, and waits for the re-run. If it's still stuck after that, it tells the user.
- **A skill that reads once** recovers the run only with the user's OK, and reads the re-run on its next run.
