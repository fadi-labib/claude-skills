# Permission Manager — Claude Code Plugin

Automatically classifies and manages Claude Code permissions following a safe-by-default security policy.

## What it does

Instead of manually editing `settings.local.json` every time Claude encounters a new command, this plugin:

1. **Collects** permission prompts and denials silently during your session
2. **Classifies** each command as safe (allow), destructive (deny), or needs-judgment (ask first)
3. **Proposes** a batch update at the end of the session
4. **Updates** your settings after you confirm

## Components

| Component | Purpose |
|---|---|
| `skills/manage-permissions/SKILL.md` | Classification rules and collection behavior |
| `commands/update-permissions.md` | `/update-permissions` — manually trigger batch review |
| `hooks/hooks.json` | SessionStart (prime policy) + Stop (remind about collected permissions) |

## Classification Rules

| Category | Action | Examples |
|---|---|---|
| **Safe/local/reversible** | Auto-allow | `git status`, `pnpm test`, `ls`, `tree` |
| **Destructive/escape-hatch** | Auto-deny | `rm -rf`, `git push --force`, `node -e` |
| **Shared state/judgment** | Keep as ask-first | `git push`, `gh pr create`, `curl` |

## Usage

- **Automatic**: Permission prompts are collected silently during the session
- **Manual**: Run `/update-permissions` to review and apply
- **End of session**: Stop hook reminds you if there are unapplied permissions
