# copilot-app-skills
Reusable skills for the GitHub Copilot app.

## Personal skills

[copilot-attribution](.copilot/skills/copilot-attribution/SKILL.md) omits attribution from command messages so bot and CI comment triggers still work. Commit messages use the inline prefix ` 🤖Copilot🤖:  ` (one leading space and two spaces after the colon), followed by the subject on the same line so GitHub displays both together. Other external interactions on your behalf, including PR descriptions, non-command comments, and review replies, start with the exact unformatted line `🤖Copilot🤖`, with the message body starting on the next line.

The skill source is in `.copilot\skills\copilot-attribution\SKILL.md`. To use it as a personal skill, copy the `copilot-attribution` directory into your user-level `.copilot\skills` directory (on Windows, `%USERPROFILE%\.copilot\skills`). Adding it to this repository does not install or enable it globally.

[azdo-pr-build-rerun](.copilot/skills/azdo-pr-build-rerun/SKILL.md) reruns Azure Pipelines builds for GitHub PRs. When a valid, logged-in Azure CLI with the `azure-devops` extension can access the build, it retries the existing build through the Azure CLI. If the Azure CLI is unavailable or invalid, or the retry fails, it falls back to `/azp run` GitHub command comments. Install it the same way by copying the `azdo-pr-build-rerun` directory into your user-level `.copilot\skills` directory.