# nightfall-required-check-test

Public throwaway repo. It reproduces two problems with a required **"Nightfall DLP"** check when the check is
posted by [`nightfallai/nightfall_dlp_action`](https://github.com/nightfallai/nightfall_dlp_action) from inside a
shared (composite) security-scan action, and it shows the fixes. The workflows here are toys: no product code, no
secrets. `main` requires a check named `Nightfall DLP` from GitHub Actions, like a protected release branch.

## Problem 1: "Nightfall DLP — Expected, waiting for status to be reported", PR blocked

### What happens

1. Someone runs a workflow **by hand** (`workflow_dispatch`, for example a manual "deploy this branch" run) on a commit.
2. Later a PR is opened whose head is that same commit (typical for a release PR from a long-lived branch).
3. The DLP scan runs on the PR and passes. Its "Nightfall DLP" check run is created through the API with
   `GITHUB_TOKEN`. That API call cannot say which workflow run the result belongs to, so GitHub files it under the
   **oldest** GitHub Actions run on the commit: the manual run from step 1.
4. GitHub does not list check runs from manually dispatched runs on a PR. The required check never shows up,
   the PR stays blocked, and it has to be merged with a bypass.

It does not matter whether the manual run succeeded or was cancelled. Earlier runs triggered by a tag push do not
cause it.

### Evidence (one PR per case, each on its own new commit)

| Case | Run on the commit before the PR | Result | PR |
|---|---|---|---|
| A2 | nothing | mergeable | [#5](https://github.com/asepulveda37461sc/nightfall-required-check-test/pull/5) |
| B | tag push, cancelled | Nightfall listed (the cancelled job shows as a failed check) | [#2](https://github.com/asepulveda37461sc/nightfall-required-check-test/pull/2) |
| D | tag push, succeeded | mergeable | [#4](https://github.com/asepulveda37461sc/nightfall-required-check-test/pull/4) |
| C, C2 | **manual run, cancelled** | **blocked, "Nightfall DLP" missing** | [#3](https://github.com/asepulveda37461sc/nightfall-required-check-test/pull/3), [#6](https://github.com/asepulveda37461sc/nightfall-required-check-test/pull/6) |
| E | **manual run, succeeded** | **blocked, "Nightfall DLP" missing** | [#8](https://github.com/asepulveda37461sc/nightfall-required-check-test/pull/8) |
| G | manual run, cancelled + **fix 1** | mergeable | [#9](https://github.com/asepulveda37461sc/nightfall-required-check-test/pull/9) |
| F | manual run, cancelled + **fix 2** | mergeable | [#10](https://github.com/asepulveda37461sc/nightfall-required-check-test/pull/10) |

In every case the check run itself exists and is `success` (`GET /repos/{owner}/{repo}/commits/{sha}/check-runs`).
Only the PR does not count it.

### Fix 1 (in the shared action, no change in the projects that use it)

After the scan, also post the result as a **commit status** with the same name. A commit status is not filed under
any workflow run, so the PR always sees it, and a status created with `GITHUB_TOKEN` satisfies a required check that
is bound to the GitHub Actions app.

```bash
# state = success | failure, same value as the check run's conclusion
curl -sS -X POST \
  -H "Authorization: Bearer $GITHUB_TOKEN" -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/$OWNER/$REPO/statuses/$PR_HEAD_SHA" \
  -d '{"state":"success","context":"Nightfall DLP","description":"0 findings"}'
```

- Use the PR head commit (`github.event.pull_request.head.sha`), not a branch name. Looking up check runs by branch
  name can fail with `422 No commit found for SHA: <branch>`.
- The token needs `statuses: write`. That is already the case wherever another required check is posted as a status.
- The toy version is in [`case-g2/.github/workflows/fake-appsec.yml`](https://github.com/asepulveda37461sc/nightfall-required-check-test/blob/case-g2/.github/workflows/fake-appsec.yml).

### Fix 2 (in each project)

Give the PR workflow a **job** whose name is `Nightfall DLP`. Its check run then always belongs to the PR's own run.
This needs a change in every project and ties the required check to the job's result, so fix 1 is the simpler one.
Toy version: [`case-f`](https://github.com/asepulveda37461sc/nightfall-required-check-test/blob/case-f/.github/workflows/fake-appsec.yml).

### Workaround until it is fixed

Don't run manual workflows on the commit you are about to open the release PR from. Tag-triggered runs are fine. If
one already ran, any new commit on the branch clears it.

## Problem 2: the scan fails with `PATCH .../check-runs/<id>: 404 Not Found`

### What happens

The scanner creates its check run and updates it about one second later with the result. Sometimes GitHub answers
the update with `404 Not Found` for the run it has just created:

```
Got 0 annotations for request #1
##[error]Error updating check run to success
nightfalldlp: failed to create check: PATCH https://api.github.com/repos/<owner>/<repo>/check-runs/<id>: 404 Not Found []
```

The scanner step fails, so the composite action stops: the steps after it are skipped, the job fails, and the
"Nightfall DLP" check run stays `in_progress` forever. The PR is blocked until someone re-runs the job.

This is a short delay on GitHub's side before a new check run can be updated, not a problem with the token or the
scan. Others report the same pattern for an immediate update after creating a check run: Red Hat's Pipelines as Code
([SRVKP-13334](https://redhat.atlassian.net/browse/SRVKP-13334)) and
[community discussion #21590](https://github.com/orgs/community/discussions/21590). It happened once in this repo
too (case A, [#1](https://github.com/asepulveda37461sc/nightfall-required-check-test/pull/1)). The scanner image (`nightfall_code_scanner:v2.2.0`) does not retry.

In one organisation using this setup it went from 0% of scan runs for 13 weeks to 0.5%, 1.2% and then 4.4% of runs
in three consecutive weeks of November 2026, with no change to the action or the scanner version. So it is worth
handling now.

### Fix: retry the scanner step

```yaml
    - name: nightfallDLP action step
      id: nightfall
      continue-on-error: true            # supported for composite action steps
      uses: nightfallai/nightfall_dlp_action@v2.2.0
      env: &nightfall_env                # same env as today
        NIGHTFALL_API_KEY: ${{ ... }}
        GITHUB_TOKEN: ${{ inputs.github_token }}
        EVENT_BEFORE: ${{ github.event.before }}

    - name: nightfallDLP action step (retry after a transient error)
      if: steps.nightfall.outcome == 'failure'
      uses: nightfallai/nightfall_dlp_action@v2.2.0
      env: *nightfall_env                # if anchors are not allowed, repeat the env block
```

- A real scan failure fails again on the retry, so the job still fails. Nothing passes by accident.
- The first attempt leaves an `in_progress` "Nightfall DLP" check run behind. That is harmless: GitHub uses the newest
  check run with that name, and the default listing (`filter=latest`) returns only the new one, so a later step that
  looks up "the" Nightfall DLP run finds the right one. Shown in case H, [#11](https://github.com/asepulveda37461sc/nightfall-required-check-test/pull/11) (mergeable).
- Any own code that updates a check run right after creating it should retry on 404 for a few seconds.

## Files

- `main`: the toy workflows. `fake-deploy.yml` (manual or tag-triggered "deploy" that sleeps), `test-on-pr.yml`,
  `fake-appsec.yml` (posts "Nightfall DLP" through the API like the scanner does, with a retry on the update).
- `case-*` branches: one per case in the tables above.
