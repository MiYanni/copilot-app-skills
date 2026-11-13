---
name: copilot-attribution
description: Use whenever composing, publishing, or editing external interactions on the user's behalf. Omit attribution for commands, prefix commit subjects inline, and require a Copilot attribution line for other messages.
---

# Copilot attribution

Choose the attribution format based on the intended interaction, in this order:

1. **Commands:** If the intended message is a command (for example, a GitHub comment that triggers a bot or CI run), do not add `🤖Copilot🤖` or any other attribution. Preserve the command exactly so the trigger can recognize it. This exception is for messages intended to execute commands, not prose that merely mentions or quotes a command.
2. **Commit messages:** Begin the subject with the exact inline prefix ` 🤖Copilot🤖:  ` (one leading space and two spaces after the colon), followed immediately by the subject text on the same line. Do not put the marker on its own line. Keep any commit body and required trailers below the subject.
3. **Other external interactions:** Begin with the exact, unformatted first line:

```text
🤖Copilot🤖
```

- Place a newline immediately after the marker; the actual message body starts on the next line.
- Apply this standalone line to PR and issue descriptions, non-command comments, inline review comments and replies, and analogous external communications.
- Do not add spaces, Markdown formatting, or other text before or on the marker line.
- Preserve the message's substance, formatting, links, references, and required commit trailers.
- When editing an existing interaction, use the appropriate format above exactly once rather than adding a duplicate prefix. Remove an existing attribution header from commands, and replace a standalone commit header with the inline prefix.
- If another target cannot accept a newline (for example, a single-line PR title), do not silently publish a noncompliant alternative. Explain the limitation and ask the user how to proceed. Commands and commit subjects use their explicit formats above.
- Check the final outgoing text before publishing or saving an edit, including text assembled by a tool.

Example PR comment:

```text
🤖Copilot🤖
Updated the retry logic to preserve the original timeout behavior.
```

Example command comment (no attribution):

```text
/azp run dotnet-sdk-public-ci
```

Example commit message:

```text
 🤖Copilot🤖:  Fix retry timeout handling

Preserve the original timeout behavior.

Co-authored-by: Copilot App <223556219+Copilot@users.noreply.github.com>
```
