---
name: copilot-attribution
description: Use whenever composing, publishing, or editing external interactions on the user's behalf, including pull request descriptions, comments, review replies, commit messages, and analogous communications. Require a Copilot attribution line before the message body.
---

# Copilot attribution

Every external interaction authored on the user's behalf must begin with the exact, unformatted first line:

```text
🤖Copilot🤖
```

- Place a newline immediately after the marker; the actual message body starts on the next line.
- Apply this to PR and issue descriptions, comments, inline review comments and replies, commit messages, and analogous external communications.
- Do not add spaces, Markdown formatting, or other text before or on the marker line.
- Preserve the message's substance, formatting, links, references, and required commit trailers after the attribution line.
- When editing an existing interaction, ensure exactly one attribution line at the beginning rather than adding a duplicate prefix.
- If a target cannot accept a newline (for example, a single-line title), do not silently publish a noncompliant alternative. Explain the limitation and ask the user how to proceed.
- Check the final outgoing text before publishing or saving an edit, including text assembled by a tool.

Example PR comment:

```text
🤖Copilot🤖
Updated the retry logic to preserve the original timeout behavior.
```
