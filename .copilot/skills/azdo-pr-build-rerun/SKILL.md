---
name: azdo-pr-build-rerun
description: Use when a GitHub pull request's Azure DevOps (Azure Pipelines) build needs to be rerun or retried. Prefer retrying through a valid, logged-in Azure CLI with the azure-devops extension; fall back to GitHub `/azp run` command comments only when the Azure CLI is unavailable, invalid, or cannot perform the rerun.
---

# Rerun Azure DevOps builds for GitHub PRs

Use this skill when the user asks to rerun, retry, or restart CI for a GitHub pull request and the failing or stale check comes from Azure Pipelines. GitHub Actions checks are out of scope; rerun those with `gh run rerun`.

Only rerun builds the user asked to rerun. Do not cancel running builds, change pipeline definitions, or run `az login` unless the user asks.

## 1. Identify the Azure DevOps builds

List the PR's checks and keep the ones whose link points to Azure DevOps:

```powershell
gh pr checks <pr> --repo <owner>/<repo> --json name,state,bucket,link
```

Azure Pipelines check links look like either of these:

- `https://dev.azure.com/<org>/<project>/_build/results?buildId=<buildId>...`
- `https://<org>.visualstudio.com/<project>/_build/results?buildId=<buildId>...`

From each link, extract:

- **Organization URL:** `https://dev.azure.com/<org>` (also for `<org>.visualstudio.com` links).
- **Project:** the path segment before `/_build`. It may be a name or a GUID; both work.
- **Build ID:** the `buildId` query parameter.

Several checks can share one build ID (for example, per-job checks). Rerun each distinct build once. Select only the builds the user asked about; if the user just said "rerun the failed build", select the failed ones (`bucket` is `fail` or `cancel`).

## 2. Validate the Azure CLI

The Azure CLI path is valid only if all of these succeed without prompting:

1. `az` is installed: `Get-Command az -ErrorAction SilentlyContinue`.
2. The extension is installed: `az extension show --name azure-devops --query version -o tsv`.
3. The CLI is authenticated for the build's organization and can read the build:

   ```powershell
   az pipelines build show --id <buildId> --org <orgUrl> --project <project> --query "{status:status,result:result,definition:definition.name,sourceVersion:sourceVersion,reason:reason}" -o json
   ```

   Authentication can come from `az login` or the `AZURE_DEVOPS_EXT_PAT` environment variable. Do not print or log tokens.

If any step fails (missing command, missing extension, login required, 401/403, organization not found), the Azure CLI path is invalid for that build: go to step 4. Validate per organization; one organization failing does not invalidate the CLI for another.

## 3. Rerun through the Azure CLI (preferred)

Check the build state from step 2 first:

- `status` is `inProgress`, `notStarted`, or `cancelling`: do not rerun. Tell the user the build is still running.
- `status` is `completed` and `result` is `failed`, `partiallySucceeded`, or `canceled`: retry the existing build.
- `status` is `completed` and `result` is `succeeded`: there is nothing to retry; if the user still wants a fresh run, go to step 4.

Retry the failed jobs of the existing build. This keeps the original PR build reason and source commit, so the result is reported back to the same GitHub check:

```powershell
$body = Join-Path $env:TEMP "azdo-retry-$([guid]::NewGuid()).json"
Set-Content -Path $body -Value '{}' -Encoding utf8
az devops invoke --org <orgUrl> --area build --resource builds --route-parameters project=<project> buildId=<buildId> --query-parameters retry=true --http-method PATCH --api-version 7.1 --in-file $body -o json
Remove-Item $body
```

If the user asks to retry only one stage, use the stage endpoint instead, with body `{"state":"retry","forceRetryAllJobs":false}` in the input file:

```powershell
az devops invoke --org <orgUrl> --area build --resource stages --route-parameters project=<project> buildId=<buildId> stageRefName=<stageName> --http-method PATCH --api-version 7.1 --in-file $body -o json
```

Do not use `az pipelines build queue` for PR validation: a manually queued build has a different build reason and may not report to the PR's checks.

Verify the retry by rerunning the `az pipelines build show` command from step 2 and confirming `status` is `notStarted` or `inProgress`. If the retry request fails (for example, 401/403 for missing "Queue builds" permission, or the service rejects the retry), report the error and go to step 4.

## 4. Fall back to GitHub command comments

Use a GitHub comment only when the Azure CLI path is unavailable or invalid, the retry failed, or a fresh build is needed (for example, the previous build succeeded). Tell the user why the fallback was used.

Use the Azure Pipelines check name as the pipeline name. Strip any job suffix in parentheses, such as `dotnet-sdk-public-ci (Build Linux)` -> `dotnet-sdk-public-ci`. Combine multiple pipelines in one comment, separated by commas:

```powershell
gh pr comment <pr> --repo <owner>/<repo> --body "/azp run <pipeline-name>[, <pipeline-name>...]"
```

Use `/azp run` with no pipeline names only when the user wants every Azure Pipelines check on the PR rerun.

The comment is a command: post it exactly as shown, with no attribution header or extra text, or the trigger will not be recognized. `/azp run` reruns the whole pipeline, not only failed jobs, and the commenter needs permission to trigger builds on the repository. After posting, check `gh pr checks` again and confirm the pipeline is pending, or report that it has not started yet.

## 5. Report

Tell the user, for each pipeline, which method was used (Azure CLI retry or `/azp run` comment), the build link, and the current status. If the fallback was used, include the reason the Azure CLI path did not apply.
