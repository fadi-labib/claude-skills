# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A **local Claude Code plugin marketplace** — a collection of prompt-based plugins loaded via a single directory source. Each subdirectory is an independent plugin with its own manifest, hooks, skills, and commands.

## Repository Structure

```
claude-skills/                  ← Marketplace root (directory source path)
  <plugin-name>/                ← One directory per plugin
    .claude-plugin/plugin.json  ← Plugin manifest (name, version, description)
    commands/                   ← Slash commands (.md with YAML frontmatter)
    skills/<skill-name>/SKILL.md ← Auto-activating skills
    hooks/hooks.json            ← Event hooks (prompt-based or command-based)
    agents/                     ← Subagent definitions (if needed)
    .mcp.json                   ← MCP server config (if needed)
```

The marketplace is registered in `~/.claude/settings.json` under `extraKnownMarketplaces.local` pointing to this repo. Individual plugins are enabled via `enabledPlugins` using `<plugin-name>@local`.

## Plugin Conventions

- **Prompt-based**: Plugins use markdown files interpreted by Claude at runtime — no compiled code.
- **Standard layout**: Follow the directory structure above. Auto-discovery finds components in conventional locations.
- **Kebab-case naming**: Plugin directories, command files, and skill directories all use kebab-case.
- **Manifest minimum**: Only `name` is required in `plugin.json`. Add `version` and `description` as good practice.

## Testing

Load all plugins from this marketplace:
```bash
claude --plugin-dir /home/fadi/projects/claude-skills
```

## Current Plugins

### permission-manager

Automatically classifies and manages Claude Code permissions using a safe-by-default security policy. Collects permission prompts/denials silently during a session, then proposes batch updates.

**Data flow**: SessionStart hook primes policy → Claude collects permission events → Stop hook reminds about unapplied permissions → `/update-permissions` triggers batch review.

**Key design**: Three-tier classification (Allow/Deny/Ask-First) driven by an ordered decision framework rather than static lookup tables. Always uses narrow permission patterns over broad wildcards.
