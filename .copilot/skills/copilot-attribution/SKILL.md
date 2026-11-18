---
name: copilot-attribution
description: Use whenever composing, publishing, or editing external interactions on the user's behalf. Omit attribution for titles and commands, prefix commit subjects inline on every host including GitHub and Azure DevOps (AzDO), and require a Copilot attribution line for other messages.
---

# Copilot attribution

Choose the attribution format based on the intended interaction, in this order:

1. **Titles:** Do not add `🤖Copilot🤖` or any other attribution to titles of PRs, issues, work items, discussions, releases, or similar items. Write the title normally; only the item's description or body gets attribution.
2. **Commands:** If the intended message is a command (for example, a GitHub comment that triggers a bot or CI run), do not add `🤖Copilot🤖` or any other attribution. Preserve the command exactly so the trigger can recognize it. This exception is for messages intended to execute commands, not prose that merely mentions or quotes a command.
3. **Commit messages (all hosts):** Use the same inline format for GitHub, Azure DevOps (AzDO), and every other Git host, regardless of whether the commit is created through Git, a CLI, an API, or a tool. Begin the subject with the exact inline prefix ` 🤖Copilot🤖:  ` (one leading space and two spaces after the colon), followed immediately by the nonempty subject text on the same line. Never put a newline between the marker and the subject. Keep any commit body and required trailers below the subject. A commit subject is not exempt under the titles rule.
   - Classify by what the text creates, not the API field name: Azure DevOps Git commit `comment` fields contain commit messages, not discussion comments, and must use this inline format.
4. **Other external interactions:** Begin with the exact, unformatted first line:

```text
🤖Copilot🤖
```

- Place a newline immediately after the marker; the actual message body starts on the next line.
- Apply this standalone line to PR and issue descriptions, non-command comments, inline review comments and replies, and analogous external communications.
- Do not add spaces, Markdown formatting, or other text before or on the marker line.
- Preserve the message's substance, formatting, links, references, and required commit trailers.
- When editing an existing interaction, use the appropriate format above exactly once rather than adding a duplicate prefix. Remove an existing attribution header from commands, and replace a standalone commit header with the inline prefix.
- If another target cannot accept a newline, do not silently publish a noncompliant alternative. Explain the limitation and ask the user how to proceed. Titles, commands, and commit subjects use their explicit rules above.
- Check the final outgoing text before publishing or saving an edit, including text assembled by a tool.
  For commits, check that the first line contains both the exact inline prefix and the actual subject; a first line containing only the marker is invalid on every host.

Example PR comment:

```text
🤖Copilot🤖
Updated the retry logic to preserve the original timeout behavior.
```

Example PR title (no attribution):

```text
Fix retry timeout handling
```

Example command comment (no attribution):

```text
/azp run dotnet-sdk-public-ci
```

Example commit message (GitHub, Azure DevOps, or any other host):

```text
 🤖Copilot🤖:  Fix retry timeout handling

Preserve the original timeout behavior.

Co-authored-by: Copilot App <223556219+Copilot@users.noreply.github.com>
```

Example Azure DevOps Git commit API field (same inline subject, not a standalone comment header):

```json
{
  "comment": " 🤖Copilot🤖:  Fix retry timeout handling\n\nPreserve the original timeout behavior.\n\nCo-authored-by: Copilot App <223556219+Copilot@users.noreply.github.com>"
}
```
