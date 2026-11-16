---
name: azdo-pr-build-rerun
description: Use when a GitHub PR needs new Azure Pipelines validation, including after a base-branch fix, changed inputs, or stale CI results, and for requests phrased as rerun or retry builds. Use an installed, authenticated GitHub CLI to re-request the provider's checks with an account permitted to rerun them; fall back to supported pipeline command comments only when direct re-requests cannot perform the operation.
---

# Request PR pipeline runs through GitHub CLI

Use GitHub CLI (`gh`), not Azure CLI, to request Azure Pipelines builds from their GitHub PR checks. No Azure CLI, Azure DevOps extension, or Azure credentials are required. Keep the existing skill directory/name `azdo-pr-build-rerun` for in-place upgrades.

The goal is new validation of the intended source and inputs, not merely resetting a check's status. Re-requesting a check asks its owning integration to run it; it does not let the caller choose a new commit, refresh a stale merge ref, or change pipeline inputs.

Only act on pipelines within the user's requested scope. Do not update PR branches, cancel builds, modify workflows, install tools, or run login commands without authorization.

## 1. Validate GitHub CLI and access

Use the PR's actual GitHub host, including on GitHub Enterprise:

```powershell
Get-Command gh -ErrorAction SilentlyContinue
gh auth status --hostname <host>
gh api --hostname <host> user --jq '.login'
gh api --hostname <host> "repos/<owner>/<repo>" --jq '{full_name,permissions}'
```

Check exit codes before consuming output. Authentication must work for the target host with the intended user; public read access alone is not proof of authentication or rerun permission. Never print tokens or use `gh auth token` to diagnose access.

The user and token must have permission to rerun the selected checks. Repository write access is a useful preflight signal, not proof: token scopes, SSO, repository policy, and the check-owning integration can impose additional restrictions. Determine support and permission from the actual re-request response; do not assume that being the PR author grants access.

If `gh` is missing, authentication fails, or the account cannot perform any supported trigger, report the specific blocker. Do not silently switch users, install tools, obtain credentials, or revert to Azure CLI. A command-comment fallback also requires a working authenticated `gh` and permission to trigger that pipeline.

## 2. Identify the selected checks

Read the current PR and its checks from the base repository, including for fork PRs:

```powershell
gh api --hostname <host> "repos/<owner>/<repo>/pulls/<pr>"
gh pr checks <pr> --repo <host>/<owner>/<repo> --json name,state,bucket,link
gh api --hostname <host> --paginate "repos/<owner>/<repo>/commits/<headSha>/check-runs?per_page=100"
```

`gh pr checks` can return a nonzero exit code because checks failed or are pending; distinguish that from an authentication or API error.

Match checks using their name, owning `app`, `details_url`, `external_id`, and PR/revision context. Azure Pipelines links normally point to `dev.azure.com` or `<org>.visualstudio.com`. Record each selected check run's `id`, `check_suite.id`, `head_sha`, status, conclusion, and provider build link. If needed, also inspect checks on the verified merge SHA and legacy commit statuses; a status context is not a check run and cannot be re-requested through the Checks API.

For "rerun failed builds", select the failed/canceled pipelines. For "validate with the fix now in main", select the affected pipelines even if old runs succeeded. Do not treat every job or analysis check as an independent pipeline.

GitHub Actions workflow runs are different: use `gh run rerun <run-id> --repo <host>/<owner>/<repo>` (or `--failed` when requested) only for actual Actions runs, never for Azure build IDs or GitHub check IDs. Actions reruns reuse the original event's SHA/ref; they are not a way to pick up a newer base revision.

## 3. Verify the source and avoid duplicate runs

When a base-branch fix or newer source is required, resolve the current PR merge ref:

```powershell
gh api --hostname <host> "repos/<owner>/<repo>/git/ref/pull/<pr>/merge" --jq '.object.sha'
gh api --hostname <host> "repos/<owner>/<repo>/commits/<mergeSha>" --jq '{sha,parents:[.parents[].sha]}'
gh api --hostname <host> "repos/<owner>/<repo>/compare/<fixSha>...<mergeSha>" --jq '{status,ahead_by,behind_by}'
```

Confirm the merge parents match the current PR head and base. `ahead` or `identical` confirms fix ancestry; `behind` or `diverged` does not. A stale, missing, or conflicted merge ref is a source blocker, not a reason to retry an old build or post a command comment. Update a PR branch only if authorized, then re-read the refs.

A check's `head_sha` may be the PR head even when the provider checks out the PR merge commit. Do not equate check association with the actual built revision. Establish the provider's checkout behavior from repository configuration, check output, or linked build evidence. If it will reuse an old revision, do not claim a re-request will test the fix. Changed pipeline inputs must also be confirmed; a re-request has no input-override payload.

Re-read the PR and selected checks immediately before triggering. Skip an equivalent pending/running build for the desired revision/inputs; do not cancel an old active build without authorization. Deduplicate multiple checks that trigger the same pipeline. Save the original check IDs, timestamps, provider links, and available build IDs to distinguish a new run afterward.

## 4. Re-request through GitHub CLI (preferred)

For the specific pipeline's trigger check, use its GitHub check-run ID:

```powershell
gh api --hostname <host> --method POST "repos/<owner>/<repo>/check-runs/<checkRunId>/rerequest"
```

Use the owning integration's supported trigger check, not an arbitrary job/analysis check. If the integration supports suite-level re-requests instead, and every check in that suite is within scope, request the suite once:

```powershell
gh api --hostname <host> --method POST "repos/<owner>/<repo>/check-suites/<checkSuiteId>/rerequest"
```

Never issue both requests for the same pipeline. A suite may include multiple pipelines; do not expand the user's scope just to use a suite request. These are webhook re-requests to the check-owning app, not creation or manual updates of check records. Do not fabricate check runs, mark failures successful, or reconstruct Azure DevOps queue payloads.

Check the exit code and HTTP response. An accepted re-request means GitHub accepted the request, not that the provider started a build. Permission or integration restrictions may reject this operation even when repository reads succeed.

On a definitive rejection or a provider that does not support the applicable re-request webhook, consider step 5. On a timeout or uncertain write result, inspect checks and provider links before retrying or commenting. If an accepted request has not produced a new build, allow for asynchronous processing and report it as unverified; do not immediately send another trigger.

## 5. Use supported command comments only as a fallback

For Azure Pipelines, use `/azp run` through `gh` only when direct re-requesting is unsupported, denied, or demonstrably cannot meet the required operation. Explain the specific limitation. The commenter still needs the provider's trigger permission; a REST denial does not prove a comment is permitted.

Resolve the exact pipeline name from repository configuration or provider evidence; do not blindly strip suffixes from check names. Deduplicate names and exclude pipelines already triggered successfully:

```powershell
gh pr comment <pr> --repo <host>/<owner>/<repo> --body "/azp run <pipeline-name>"
```

Use a bare `/azp run` only when the user requested all Azure Pipelines pipelines on the PR. Post command comments without attribution or extra prose so the trigger remains intact. Do not treat a successful comment post as successful build creation. Comments cannot fix stale merge refs or guarantee refreshed inputs either.

## 6. Verify and report

Re-read the selected checks using `gh pr checks` and the paginated Checks API. Look for updated attempt timestamps, changed provider build links/IDs, or new check records. A check re-request can reuse GitHub check/suite IDs, and a provider may create a new build ID; neither ID behavior alone proves the correct source was built. A reset to `queued` alone is not proof the provider acted.

Confirm the provider run and its actual revision/inputs from available check output and linked build evidence. A fast run may already be completed. If the required fix is absent, report the source mismatch instead of claiming success. Do not bypass inaccessible provider evidence with assumptions.

Report the pipeline, method (check/suite re-request, Actions rerun, or command fallback), run/check link, current status, and whether the required revision is confirmed. Distinguish requested, accepted, provider-started, and verified. State any permission, source, or verification blocker plainly.

References: [Re-request a check run](https://docs.github.com/en/rest/checks/runs#rerequest-a-check-run), [re-request a check suite](https://docs.github.com/en/rest/checks/suites#rerequest-a-check-suite), and [`gh run rerun`](https://cli.github.com/manual/gh_run_rerun).
