---
name: azdo-pr-build-rerun
description: Use when a GitHub PR needs Azure Pipelines validation, including rerunning failed builds or validating newer source or inputs. Prefer Azure CLI to retry failed jobs of the existing PR build through Azure DevOps REST, preserving PR context. Verify Azure permissions, source revisions, job attempts, and GitHub checks; do not rely on GitHub check re-requests or silently fall back to command comments.
---

# Rerun PR pipelines through Azure CLI

Use Azure CLI (`az`) and Azure DevOps REST to retry failed jobs of an existing Azure Pipelines PR build. Use GitHub CLI (`gh`) to discover the PR and checks, not to trigger Azure builds. Keep the existing skill directory/name `azdo-pr-build-rerun` for in-place upgrades.

Do not call GitHub's check-run or check-suite re-request endpoints (`POST .../check-runs/<id>/rerequest`, `POST .../check-suites/<id>/rerequest`) with `gh`. GitHub documents both as Checks: write endpoints available only to GitHub App tokens: user access tokens (`ghu_`) and installation access tokens (`ghs_`). Check-run re-requests also require the token to belong to the app that owns the run (`azure-pipelines`). `gh` OAuth tokens (`gho_`), classic PATs (`ghp_`), and fine-grained PATs (`github_pat_`) are not accepted, and users effectively never hold the owning app's token. `gh auth status` shows the masked token prefix; never print the token itself.

In the investigation, a user with repository maintain/push permission and `repo`/`workflow` scopes received REST `404 Not Found` (GraphQL: FORBIDDEN) on four `azure-pipelines` check runs, even though GET requests worked and the suites reported `rerequestable: true`. A 404 from these POSTs means the token type is unsupported. It does not mean the check run is missing. Do not retry, add scopes, switch accounts, or switch between the run and suite endpoints; all fail the same way. More scopes, PR authorship, or `rerequestable: true` do not help. Do not send diagnostic re-request POSTs either: if one unexpectedly succeeds, it can queue an unintended build. Send one only after the user approves, or after confirming that a new build for that PR is already queued or running. `gh run rerun` is only for GitHub Actions, never Azure build IDs or GitHub check IDs.

The goal is validation of the intended source and inputs, not merely resetting a check's status. Retrying failed jobs preserves the existing build's revision, inputs, and PR context; it does not refresh a stale merge ref or pick up a newer base-branch fix.

Only act on pipelines within the user's requested scope. Do not update PR branches, cancel builds, modify workflows, install tools, or run login commands without authorization.

## 1. Validate tools and authentication

Use the PR's actual GitHub host, including on GitHub Enterprise. Check exit codes before consuming output:

```powershell
Get-Command gh -ErrorAction SilentlyContinue
Get-Command az -ErrorAction SilentlyContinue
gh auth status --hostname <host>
gh api --hostname <host> user --jq '.login'
az account show --query '{tenantId:tenantId}' --output json
```

The preferred `az rest` path uses Microsoft Entra authentication from an authorized `az login`, with the Azure DevOps resource ID `499b84ac-1321-427f-aa17-267ca6975798`. It is a core Azure CLI command and does not require the `azure-devops` extension. Azure subscription ownership is not proof of Azure DevOps access; the account needs access to the target organization/project and permission to queue/retry the selected pipeline.

GitHub authentication does not authenticate Azure DevOps. Public build reads and GitHub repository write permission do not prove Azure retry permission. If Azure CLI is missing, its session has expired, or access is denied, report the specific blocker. Request authorization before installation or sign-in; do not silently switch accounts or tenants. An authorized Entra login may use `az login --tenant <tenant-id> --allow-no-subscriptions` when the account has no Azure subscriptions.

An existing Azure DevOps PAT is a separate alternative through the `azure-devops` extension's `az devops invoke`, using its supported credential store or `AZURE_DEVOPS_EXT_PAT`. It needs Build (Read & execute), corresponding to `vso.build_execute`, plus pipeline permissions. `az rest` does not automatically consume that PAT. Check `az extension show --name azure-devops` before extension commands, since they can auto-install it. Never print tokens, request secrets in chat, or use `gh auth token` to diagnose access.

## 2. Identify the selected PR builds

Read the current PR and its checks from the base repository, including for fork PRs:

```powershell
gh api --hostname <host> "repos/<owner>/<repo>/pulls/<pr>"
gh pr checks <pr> --repo <host>/<owner>/<repo> --json name,state,bucket,link
gh api --hostname <host> --paginate "repos/<owner>/<repo>/commits/<headSha>/check-runs?per_page=100"
```

`gh pr checks` can return a nonzero exit code because checks failed or are pending; distinguish that from an authentication or API error.

Match checks using their name, owning `app`, `details_url`, `external_id`, and PR/revision context. Azure Pipelines links normally point to `dev.azure.com` or `<org>.visualstudio.com`. Resolve the actual organization, project, Azure build ID, and pipeline definition ID from the linked build; do not confuse check IDs with build IDs or hardcode another repository's organization.

Read the build and timeline using Azure CLI. For a legacy provider URL, resolve its canonical Azure DevOps API endpoint first:

```powershell
az rest --method get --resource 499b84ac-1321-427f-aa17-267ca6975798 --uri "https://dev.azure.com/<org>/<project>/_apis/build/builds/<buildId>" --uri-parameters api-version=7.1 --output json
az rest --method get --resource 499b84ac-1321-427f-aa17-267ca6975798 --uri "https://dev.azure.com/<org>/<project>/_apis/build/builds/<buildId>/timeline" --uri-parameters api-version=7.1 --output json
```

Record the build's `definition.id`, repository, `reason`, `sourceBranch`, `sourceVersion`, PR metadata in `triggerInfo`, parameters/template parameters, status, result, timestamps, and web link. Save timeline job IDs, attempts, states, and results, plus the associated GitHub check IDs, timestamps, and links. Verify this is the intended PR build, not a similarly named branch/manual run.

For "rerun failed builds", select the failed/canceled pipelines. For "validate with the fix now in main", select the affected pipelines even if old runs succeeded. Deduplicate job/analysis checks that belong to one pipeline build.

## 3. Verify the source and avoid duplicate runs

For an unchanged-source infrastructure failure, confirm that the original build still matches the intended PR source and inputs. When a base-branch fix or newer source is required, resolve the current PR merge ref:

```powershell
gh api --hostname <host> "repos/<owner>/<repo>/git/ref/pull/<pr>/merge" --jq '.object.sha'
gh api --hostname <host> "repos/<owner>/<repo>/commits/<mergeSha>" --jq '{sha,parents:[.parents[].sha]}'
gh api --hostname <host> "repos/<owner>/<repo>/compare/<fixSha>...<mergeSha>" --jq '{status,ahead_by,behind_by}'
```

Confirm the merge parents match the current PR head and base. `ahead` or `identical` confirms fix ancestry; `behind` or `diverged` does not. A stale, missing, or conflicted merge ref is a source blocker, not a reason to retry an old build. Update a PR branch only if authorized, then re-read the refs.

A check's `head_sha` may be the PR head even when Azure checks out the PR merge commit. Compare the provider's `sourceVersion` and checkout evidence with the required revision; check association alone is insufficient. **Do not retry an old build to validate a fix absent from its source or changed inputs.** Use the fresh-validation guidance in step 5 instead.

Immediately before triggering, re-read the PR, selected build/timeline, and recent builds for the same definition and PR ref:

```powershell
az rest --method get --resource 499b84ac-1321-427f-aa17-267ca6975798 --uri "https://dev.azure.com/<org>/<project>/_apis/build/builds" --uri-parameters definitions=<definitionId> branchName=refs/pull/<pr>/merge queryOrder=queueTimeDescending api-version=7.1 --output json
```

Use the verified source ref if the pipeline uses a different one. Follow continuation tokens when necessary. Skip an equivalent pending/running build or retry for the desired revision/inputs, and do not cancel an old active build without authorization. This applies to every trigger, including Azure retries, fresh runs, and `/azp run` comments. Check the PR's current head commit for a queued or running build. If you cannot determine the build state, ask the user before triggering.

## 4. Retry failed jobs with Azure CLI (preferred)

For a completed PR build with retryable failed/canceled jobs and the correct source/inputs, use the Azure DevOps **Builds - Update Build** API with `retry=true`. This retries failed jobs within the existing build, rather than queueing an unrelated manual run. It is not a full rerun of already successful jobs. If the request targets only a subset of failed jobs, do not use a build-wide retry that expands that scope.

PowerShell example (replace the endpoint placeholders with the verified build):

```powershell
$bodyFile = New-TemporaryFile
try {
    Set-Content -LiteralPath $bodyFile.FullName -Value '{}' -NoNewline -Encoding ascii
    az rest --method patch --resource 499b84ac-1321-427f-aa17-267ca6975798 --uri "https://dev.azure.com/<org>/<project>/_apis/build/builds/<buildId>" --uri-parameters retry=true api-version=7.1 --headers Content-Type=application/json --body "@$($bodyFile.FullName)" --output json
    if ($LASTEXITCODE -ne 0) { throw "Azure DevOps build retry failed (exit $LASTEXITCODE)." }
}
finally {
    Remove-Item -LiteralPath $bodyFile.FullName
}
```

**Windows:** pass query values with `--uri-parameters`, not `?retry=true&api-version=7.1` in the URL. The `az.cmd` wrapper can split at `&` even when the URL is quoted in PowerShell; the investigation's first request failed with a missing API version for this reason. The body file also avoids native-command JSON quoting issues. Clean up only the temporary file created for this request.

For an already configured PAT-based extension session, the equivalent request is `az devops invoke --organization <org-url> --area build --resource builds --route-parameters project=<project> buildId=<buildId> --query-parameters retry=true --http-method PATCH --api-version 7.1 --in-file <body-file> --output json`, with the same `{}` body, scope checks, and verification.

Check the exit code and response. On a timeout or uncertain write result, inspect the build/timeline and GitHub checks before any retry. An accepted PATCH is not proof that a job started. A definitive authentication, permission, or non-retryable-build failure is a blocker, not a reason to send another trigger automatically.

## 5. Fresh validation and fallback limits

A retry keeps the original build's source and inputs. For newer source, changed parameters, a full rerun, or a build without retryable jobs, establish a supported fresh-run mechanism for that repository before triggering.

Do not blindly substitute `az pipelines run` or `az pipelines build queue --branch refs/pull/<pr>/merge`. They queue a new manual build; a PR-shaped ref alone does not preserve `Build.Reason=PullRequest`, PR variables, pipeline conditions, or reporting on the PR head. A successful manual run is not proof that the required PR check was refreshed. Only use such a run when its semantics satisfy the request and its source/inputs and reporting can be verified; otherwise report that fresh PR validation is blocked. Do not fabricate PR trigger metadata or modify check records.

Use `/azp run <exact-pipeline-name>` only if the user explicitly authorizes a command-comment fallback and the integration supports it. The comment requires repository write access. In the investigation, it successfully queued new builds where check re-requests returned 404. Resolve the pipeline name from repository configuration or provider evidence; the check run name is usually the pipeline name. Never post one when the user requested a non-comment CLI operation. Explain the Azure limitation first, preserve the command without attribution or extra prose, and verify the resulting provider build and source. A GitHub API denial does not prove comment-trigger permission. Comments cannot fix stale merge refs either.

## 6. Verify and report

Re-read the Azure build and timeline from step 2. Compare job attempts, states, and start times against the saved baseline; a retry normally retains the build ID while increasing attempts for retried jobs. A fast retry may already be completed. Confirm that only the intended work was retried and that the actual source/inputs still match the request.

Re-read GitHub checks through `gh pr checks` and the paginated Checks API. Confirm the Azure integration updated or created the expected check on the intended PR head. New check IDs are possible even when the Azure build ID is unchanged. A reset to `queued` alone is not proof a job started, and provider activity alone is not proof PR reporting is correct.

Report the pipeline, method (Azure failed-job retry, separately verified fresh run, or authorized command fallback), build/check link, current status, job attempt evidence, and whether the required revision and PR reporting are confirmed. Distinguish requested, accepted, provider-started, and verified; do not call a running retry successful. State any permission, source, or verification blocker plainly rather than assuming success.

References: [Builds - Update Build (`retry`)](https://learn.microsoft.com/en-us/rest/api/azure/devops/build/builds/update-build?view=azure-devops-rest-7.1), [`az rest`](https://learn.microsoft.com/en-us/cli/azure/reference-index#az-rest), [Azure DevOps CLI authentication](https://learn.microsoft.com/en-us/azure/devops/cli/log-in-via-pat?view=azure-devops), [`az devops invoke`](https://learn.microsoft.com/en-us/cli/azure/devops#az-devops-invoke), [`az pipelines run`](https://learn.microsoft.com/en-us/cli/azure/pipelines#az-pipelines-run), and [GitHub check re-request permissions](https://docs.github.com/en/rest/checks/runs#rerequest-a-check-run) ([token types](https://docs.github.com/en/rest/authentication/permissions-required-for-github-apps#repository-permissions-for-checks)).
