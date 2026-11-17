---
name: draft-pr-creation
description: Use whenever creating or opening a GitHub pull request on the user's behalf, through any tool (app PR tools, GitHub CLI, MCP, or API). Always create new PRs as drafts so the user can review and iterate with Copilot before marking them ready for broad team review.
---

# Create pull requests as drafts

Every new pull request created on the user's behalf must start in the **Draft** state. Drafts let the user review the change and iterate with Copilot first. The user alone decides when a PR is ready; moving it out of Draft signals to their team that it should be broadly reviewed for approval.

## Creating the PR

Request the draft state at creation time, whichever tool is used:

- **App PR tool** (`create_pull_request`): pass `draft: true`.
- **GitHub CLI**: `gh pr create --draft ...`.
- **REST/GraphQL/MCP**: set `"draft": true` (REST) or `draft: true` (GraphQL `createPullRequest`).

Do not create a ready PR and convert it afterward; that briefly notifies reviewers and may trigger ready-only automation.

Draft creation is a hard requirement, not an inferred preference. If the selected tool or repository cannot create a draft (for example, draft PRs are unavailable for the repository's plan), do not fall back to a ready PR. Report the limitation and ask the user how to proceed.

After creation, verify the PR is a draft (for example, `gh pr view <pr> --repo <host>/<owner>/<repo> --json isDraft`). If it is not, report the mismatch; convert it with `gh pr ready <pr> --undo` unless that conflicts with the user's instructions.

## Leaving the draft state

Never mark a PR ready for review (`gh pr ready`, `update_pull_request` with `draft: false`, or equivalent) unless the user explicitly asks for that specific PR in the current request. Finishing the work, passing CI, addressing review feedback, or automated merge workflows are not authorization.

## Scope

- Applies to newly created PRs only. Do not change the draft state of existing PRs, including ones linked to the session instead of created.
- An explicit user request to create a specific PR as ready for review overrides this default for that PR only.
- Other PR content conventions (titles, descriptions, attribution) still apply.
