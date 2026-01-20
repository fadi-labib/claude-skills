---
name: update-permissions
description: Review the current session for permission prompts and denied tool calls, then classify and add them to settings
---

# Update Permissions

Review this conversation for any tool calls that were denied or required permission prompts. Use both your collected notes AND a fresh scan of the conversation (earlier notes may have been lost to context compression).

For each permission found:

1. Read the current `.claude/settings.local.json` (project-level). If it doesn't exist, check `~/.claude/settings.json` (global)
2. Identify all commands/tools that were prompted, denied, or explicitly approved by the user
3. Use the `manage-permissions` skill decision framework to classify each one
4. Present a summary table:

| Command | Pattern | Classification | Reason |
|---------|---------|---------------|--------|
| ... | `Bash(...)` | Allow / Deny / Ask | ... |

5. Ask which settings file to update (project-level or global). Default to project-level if one exists
6. Ask the user to confirm before applying changes
7. Update the settings file with the approved changes
8. Show the user exactly what was added

If no permission issues were found in this session, tell the user their settings are up to date.
