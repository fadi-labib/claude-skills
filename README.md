# Claude Skills — Local Plugin Marketplace

A collection of Claude Code plugins loaded via a single local directory source.

## Installation

Add to `~/.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "local": {
      "source": {
        "source": "directory",
        "path": "/path/to/claude-skills"
      }
    }
  },
  "enabledPlugins": {
    "permission-manager@local": true
  }
}
```

Or test all plugins directly:

```bash
claude --plugin-dir /path/to/claude-skills
```

## Plugins

| Plugin | Description |
|---|---|
| [permission-manager](permission-manager/) | Automatically classifies and manages Claude Code permissions following a safe-by-default security policy |
