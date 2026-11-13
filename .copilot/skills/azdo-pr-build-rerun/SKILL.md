---
name: azdo-pr-build-rerun
description: Use when a GitHub PR needs a new Azure DevOps (Azure Pipelines) run, including after a base-branch fix, changed inputs, or stale CI results. Prefer queueing a fresh PR validation build through a valid, logged-in Azure CLI with the azure-devops extension; use GitHub `/azp run` comments only when the CLI cannot perform that operation. Also applies to requests phrased as rerun or retry CI.
---

# New Azure DevOps pipeline runs for GitHub PRs

Use this skill whenever a GitHub PR needs a new Azure Pipelines run. Examples include a fix merged into the base branch that should make tests pass, changed pipeline inputs, stale checks, and requests phrased as "rerun the failed builds". GitHub Actions are out of scope.

**Default to a fresh build (a new build ID), not retrying jobs in an old build.** An existing-build retry preserves the old source revision and cannot pick up a base-branch fix. A fresh run must still be checked for the correct revision; neither the CLI nor `/azp run` guarantees that GitHub has refreshed a stale PR merge ref.

The CLI-first preference applies to fresh runs too. Do not skip the CLI merely because the prior build succeeded, its revision is stale, or a job retry would not pick up the fix.

Only start pipelines within the user's requested scope. Do not update PR branches, cancel running builds, change pipeline definitions, install extensions, or run login commands without authorization. Keep the skill directory/name `azdo-pr-build-rerun` so existing installations can be updated in place.

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

URL-decode the project segment. Several checks can share one build ID; deduplicate them. Then resolve the pipeline definition from that build and queue at most once per organization/project/definition/PR. Do not assume each job check is a separate pipeline.

Select pipelines by the user's requested outcome, not just the old build result. For "rerun failed builds", select failed or canceled pipelines. For "validate with the fix now in main", select the affected pipelines even if their old runs succeeded. Before queueing, check for an already queued/running build of the same pipeline and desired revision; do not create duplicates.

## 2. Establish the source revision the new run needs

Read the current PR and its merge ref from the PR's base repository, including for fork PRs:

```powershell
gh api "repos/<owner>/<repo>/pulls/<pr>"
gh api "repos/<owner>/<repo>/git/ref/pull/<pr>/merge" --jq '.object.sha'
```

For normal GitHub PR validation, use `refs/pull/<pr>/merge` and its current SHA, not the old build's `sourceVersion`, the PR head SHA alone, or `refs/heads/main`. Confirm the merge commit's parents match the current PR head and base revisions. If the user identifies a merged fix, verify that the desired merge SHA contains it:

```powershell
gh api "repos/<owner>/<repo>/commits/<mergeSha>" --jq '{sha,parents:[.parents[].sha]}'
gh api "repos/<owner>/<repo>/compare/<fixSha>...<mergeSha>" --jq '{status,ahead_by,behind_by}'
```

`ahead` or `identical` confirms fix ancestry; `behind` or `diverged` does not. If the merge ref is missing, conflicted, stale, or lacks the required fix, report the source-revision blocker. A branch update or merge-ref refresh may be needed; perform a branch update only if authorized. Re-read the refs after any update.

Do not queue the known-old revision and claim it tests the fix. Switching to a command comment does not solve a stale merge ref. If the user explicitly wants a run of that old revision anyway, report that limitation.

## 3. Validate the Azure CLI

Check without prompting:

1. `az` is installed: `Get-Command az -ErrorAction SilentlyContinue`.
2. The extension is installed: `az extension show --name azure-devops --query version -o tsv`.
3. An existing Azure DevOps login, `AZURE_DEVOPS_EXT_PAT`, or Azure login is usable for the organization. For Azure login, check token acquisition without printing the token: `az account get-access-token --resource 499b84ac-1321-427f-aa17-267ca6975798 --query expiresOn -o tsv`. Do not require an Azure subscription or this token check when using DevOps PAT authentication.
4. Read the full original build to obtain its definition, repository, parameters, and PR trigger metadata:

   ```powershell
   az pipelines build show --id <buildId> --org <orgUrl> --project <project> -o json
   ```

Never print credentials or use `--debug` to inspect authentication. Anonymous read access to a public build is not proof of a valid login or queue permission. Check exit codes before consuming CLI output. Validate per organization; one organization failing does not invalidate another.

Missing CLI/extension, invalid authentication, or denied access permits step 5. If reads succeed, attempt the CLI queue operation below; lack of queue permission must be established from an actual rejection, not inferred merely because PR validation is involved.

## 4. Queue a fresh PR validation build through the CLI (preferred)

Use the Build Queue REST API through `az devops invoke`. The convenience command `az pipelines build queue` cannot express all the PR metadata below; that is a reason to use `invoke`, not a reason to abandon the CLI.

Construct a minimal queue request from the original PR validation build and the current PR:

- Preserve `definition.id`, the GitHub `repository` identity, applicable `parameters`, `templateParameters`, and `triggerInfo`. Confirm the original build belongs to this PR and uses `reason: pullRequest`. Do not replay the entire build object, including its old ID, status, result, timestamps, or orchestration plans.
- Set `reason` to `pullRequest`, `sourceBranch` to the verified PR merge ref, and `sourceVersion` to the verified current merge SHA.
- Refresh existing PR metadata against the current PR, particularly `system.pullRequest.sourceCommitId` in the JSON-encoded `parameters` and `pr.sourceSha` in `triggerInfo` (both use the current PR **head** SHA, not the merge SHA). Also refresh PR IDs, number, head/base branch names, fork identity, and any revision-bearing metadata present. Preserve required pipeline inputs unless the user requested changes.
- If the original build's PR context cannot be established or safely reconstructed, explain what is missing and use step 5. Do not invent metadata or mislabel a generic manual build as PR validation.

Illustrative queue request, after resolving and validating `$original`, `$mergeRef`, `$mergeSha`, and refreshing `$parameters`/`$triggerInfo` as above:

```powershell
$request = @{
    definition = @{ id = $original.definition.id }
    repository = $original.repository
    reason = 'pullRequest'
    sourceBranch = $mergeRef
    sourceVersion = $mergeSha
    triggerInfo = $triggerInfo
}
if ($null -ne $parameters) {
    $request.parameters = ($parameters | ConvertTo-Json -Depth 20 -Compress)
}
if ($null -ne $original.templateParameters) {
    $request.templateParameters = $original.templateParameters
}
$body = Join-Path $env:TEMP "azdo-queue-$([guid]::NewGuid()).json"
try {
    $json = $request | ConvertTo-Json -Depth 30
    [System.IO.File]::WriteAllText($body, $json, [System.Text.UTF8Encoding]::new($false))
    az devops invoke --org $orgUrl --area build --resource builds --route-parameters "project=$project" --http-method POST --api-version 7.1 --in-file $body --encoding utf-8 -o json
    $queueExitCode = $LASTEXITCODE
} finally {
    Remove-Item -LiteralPath $body
}
```

This is a POST creating a new build, not `PATCH ...?retry=true`. Only retry an existing build or stage when the user specifically wants failed jobs retried on the **same** revision; do not substitute a retry for a fresh-run requirement.

Capture the queue response and its new build ID. Check the exit code, validation errors, `reason`, `sourceBranch`, and `sourceVersion`, then read the new build with `az pipelines build show`. Confirm the new ID differs from the original and the source matches the verified revision. A fast build may already be completed, so pending/running is not the only valid state. Check GitHub for the corresponding pipeline check; do not assume setting `reason` guarantees check publication for every integration.

If the service definitively rejects the request, report the specific error and proceed to step 5. On a timeout or other uncertain write outcome, look for a newly queued matching build before trying again or commenting; otherwise a duplicate run may be created. An accepted build that has not yet published a GitHub check is not grounds for immediately starting another run.

## 5. Fall back to GitHub command comments

Use comments only when the CLI is unavailable/invalid, rejects the fresh-run request, or cannot preserve the required PR context. A fresh run being necessary is **not** by itself a fallback condition. Explain the actual CLI limitation.

Resolve the pipeline's actual definition name from the build or repository configuration; do not blindly strip parentheses from check names. Deduplicate pipeline names, exclude builds already queued successfully through the CLI, and combine remaining pipelines in one comment:

```powershell
gh pr comment <pr> --repo <owner>/<repo> --body "/azp run <pipeline-name>[, <pipeline-name>...]"
```

Use `/azp run` with no pipeline names only when the user wants every Azure Pipelines pipeline on the PR started.

The comment is a command: post it without attribution or extra prose. The commenter needs permission to trigger builds. Posting a comment is not proof that a build started. Check for a new build/check and verify its source revision using the same requirements as step 4; report if it has not started or still uses an old revision.

## 6. Report

For each pipeline, report the method (CLI fresh run, explicitly requested same-revision retry, or command fallback), new build link and status, and whether the required fix/revision is included. Distinguish requested, accepted, and verified runs. Include any source-revision blocker or specific reason for fallback; never claim the fix was tested just because a new build ID exists.

Reference: [Azure DevOps Build Queue API](https://learn.microsoft.com/en-us/rest/api/azure/devops/build/builds/queue?view=azure-devops-rest-7.1).
