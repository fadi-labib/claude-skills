---
name: update-permissions
description: Review the current session for permission prompts and denied tool calls, then classify and add them to settings
---

# Update Permissions

Review this conversation for any tool calls that were denied or required permission prompts. For each one:

1. Read the current `.claude/settings.local.json`
2. Identify commands/tools that were prompted or denied
3. Use the `manage-permissions` skill to classify each one
4. Present a summary table to the user:

| Command | Classification | Reason |
|---------|---------------|--------|
| ... | Allow / Deny / Ask | ... |

5. Ask the user to confirm before applying changes
6. Update `.claude/settings.local.json` with the approved changes
7. Show the user what was added

If no permission issues were found in this session, tell the user and suggest they can run this command anytime after encountering permission prompts.
