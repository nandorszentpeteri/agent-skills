---
name: setup-mcp
description: >-
  Sync MCP server configuration across AI coding tools (Claude, Cursor, OpenCode,
  Copilot) from a single canonical .agents/mcp.json. Only invoke when the user
  asks to set up MCP, sync MCP servers, or add/remove MCP servers.
---

# Setting Up MCP Server Configuration

This skill manages MCP (Model Context Protocol) server configuration across multiple AI coding tools from a single canonical source. It ensures all tools share the same MCP servers without manual per-tool configuration.

## Supported Tools

| Tool | Project Config | Global Config | Top Key | Notes |
|------|---------------|---------------|---------|-------|
| Claude Code | `.mcp.json` | `~/.claude.json` | `mcpServers` | Global embeds in larger config |
| Cursor | `.cursor/mcp.json` | `~/.cursor/mcp.json` | `mcpServers` | |
| Open Code | `opencode.json` | `~/.config/opencode/opencode.json` | `mcp` | `local`/`remote` types, `environment` key |
| GitHub Copilot | `.vscode/mcp.json` | (not supported) | `servers` | Global path is platform-specific |

## Canonical Config Format

The canonical config lives at:
- **Project**: `.agents/mcp.json` (in the repo root)
- **Global**: `~/.agents/mcp.json`

Format matches Claude Code's `.mcp.json` schema:

```json
{
  "mcpServers": {
    "server-name": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "some-mcp-package"],
      "env": { "API_KEY": "value" },
      "disabled": false
    },
    "remote-server": {
      "type": "http",
      "url": "https://mcp.example.com",
      "headers": { "Authorization": "Bearer token" }
    }
  }
}
```

### Fields

- **type**: `stdio` (local process) or `http`/`sse` (remote)
- **command** + **args**: for `stdio` servers
- **url**: for `http`/`sse` servers
- **env**: environment variables (optional)
- **headers**: HTTP headers (optional, for remote servers)
- **disabled**: set to `true` to exclude from sync without removing the entry

## Workflow

### Adding an MCP Server

1. Edit the canonical config (`.agents/mcp.json` or `~/.agents/mcp.json`)
2. Add the server entry under `mcpServers`
3. Run the sync script

### Removing an MCP Server

1. Remove the entry from the canonical config (or set `"disabled": true`)
2. Run the sync script — it will remove the server from all tool configs

### Running the Sync Script

The sync script is located at the skill directory. Run it with:

```bash
# Project scope (auto-detected if .agents/mcp.json exists in cwd)
python3 /path/to/setup-mcp/sync.py

# Explicit project scope with custom root
python3 /path/to/setup-mcp/sync.py --scope project --project-root /path/to/repo

# Global scope
python3 /path/to/setup-mcp/sync.py --scope global

# Non-interactive (skip gum selection, sync to all detected tools)
python3 /path/to/setup-mcp/sync.py --non-interactive

# Specific tools only
python3 /path/to/setup-mcp/sync.py --tools claude,cursor
```

When run interactively (default), the script uses `gum` to let the user select which tools to sync to. All detected tools are pre-selected.

### Flags

| Flag | Description |
|------|-------------|
| `--scope project\|global` | Target scope (default: auto-detect) |
| `--project-root <path>` | Project root directory (default: cwd) |
| `--tools <list>` | Comma-separated tool IDs: `claude`, `cursor`, `opencode`, `copilot` |
| `--non-interactive` | Skip interactive selection, sync to all detected tools |

## How the AI Should Use This Skill

When the user asks to add, remove, or sync MCP servers:

1. **Determine scope** — ask the user if unclear (project or global)
2. **Check if canonical config exists** — if not, create `.agents/mcp.json` (or `~/.agents/mcp.json`)
3. **Edit the canonical config** — add/remove/modify server entries
4. **Run the sync script** — use `--non-interactive` when running programmatically

Example: user asks "add the filesystem MCP server to all my tools"

```bash
# 1. Create/edit canonical config
mkdir -p .agents
cat > .agents/mcp.json << 'EOF'
{
  "mcpServers": {
    "filesystem": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/dir"]
    }
  }
}
EOF

# 2. Run sync
python3 /path/to/setup-mcp/sync.py --scope project --non-interactive
```

## Merge Behavior

The sync script preserves servers that were added directly to a tool's config (not via the canonical file). It tracks which servers it manages using a manifest file (`.agents/.mcp-sync-manifest.json`).

- **Canonical servers**: always synced (added/updated/removed as needed)
- **Tool-native servers**: never touched — servers added directly to `.cursor/mcp.json` etc. are preserved
- **Disabled servers**: excluded from sync but tracked in the manifest

## Format Transformations

The script automatically handles format differences:

- **Open Code**: `stdio` becomes `local`, `http`/`sse` becomes `remote`, `command`+`args` merge into a single array, `env` becomes `environment`
- **GitHub Copilot**: `mcpServers` becomes `servers`, `env` is stripped (not supported)
- **Claude Code / Cursor**: nearly pass-through (strip `disabled` field)
