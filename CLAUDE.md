# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Claude Code plugin** called `permission-manager`. It automatically classifies and manages Claude Code tool permissions using a safe-by-default security policy. Instead of manually editing `settings.local.json`, it collects permission prompts/denials during a session, classifies them, and proposes batch updates.

## Architecture

This is a prompt-based plugin (no compiled code). All logic lives in markdown files interpreted by Claude at runtime.

```
.claude-plugin/plugin.json   → Plugin manifest (name, version, description)
hooks/hooks.json              → SessionStart + Stop hooks (prompt-based)
skills/manage-permissions/    → Core classification skill with decision framework
commands/update-permissions.md → /update-permissions slash command
```

**Data flow**: SessionStart hook primes the classification policy → Claude silently collects permission events during the session → Stop hook reminds about unapplied permissions → `/update-permissions` command triggers batch review and settings update.

## Key Design Decisions

- **Batch, not immediate**: Permissions are collected silently and applied in a single batch, not one-at-a-time during the session.
- **Three-tier classification**: Commands are classified as Allow (safe/local/reversible), Deny (destructive/escape-hatch), or Ask-First (shared state/judgment required).
- **Decision framework over static lists**: Unknown commands are classified by asking ordered questions (can it destroy data? bypass rules? affect shared state? etc.) rather than relying solely on a lookup table.
- **Narrow patterns**: Always prefer specific permission patterns like `Bash(npx prisma generate:*)` over broad wildcards like `Bash(npx:*)`.

## Testing the Plugin

```bash
claude --plugin-dir /home/fadi/projects/claude-skills
```

## Plugin Manifest Format

The manifest at `.claude-plugin/plugin.json` uses the standard Claude Code plugin schema. Hooks use `"type": "prompt"` (not shell commands) — they inject instructions into Claude's context rather than running external processes.
