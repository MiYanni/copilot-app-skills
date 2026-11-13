# copilot-app-skills
Reusable skills for the GitHub Copilot app.

## Personal skills

[copilot-attribution](.copilot/skills/copilot-attribution/SKILL.md) omits attribution from command messages so bot and CI comment triggers still work. Commit messages use the inline prefix ` 🤖Copilot🤖:  ` (one leading space and two spaces after the colon), followed by the subject on the same line so GitHub displays both together. Other external interactions on your behalf, including PR descriptions, non-command comments, and review replies, start with the exact unformatted line `🤖Copilot🤖`, with the message body starting on the next line.

The skill source is in `.copilot\skills\copilot-attribution\SKILL.md`. To use it as a personal skill, copy the `copilot-attribution` directory into your user-level `.copilot\skills` directory (on Windows, `%USERPROFILE%\.copilot\skills`). Adding it to this repository does not install or enable it globally.

[azdo-pr-build-rerun](.copilot/skills/azdo-pr-build-rerun/SKILL.md) starts new Azure Pipelines runs for GitHub PRs, including when a base-branch fix or changed inputs require fresh validation. It prefers queueing a new PR validation build through a valid, logged-in Azure CLI with the `azure-devops` extension, preserving PR metadata and verifying the source revision contains the required fix. Existing-build retries are reserved for explicit same-revision requests. `/azp run` comments are a fallback only when the CLI cannot perform the operation, not simply because a fresh run is needed. A stale PR merge ref is reported as a blocker, not bypassed with a comment. The skill keeps its original directory/name for in-place upgrades. Install or update it by copying the `azdo-pr-build-rerun` directory into your user-level `.copilot\skills` directory.