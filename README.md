# Agent Skills

Reusable AI agent skills for coding assistants.

## What are Skills?

Skills are markdown-based instruction sets that teach AI coding assistants how to perform specific tasks. They provide domain knowledge, workflows, and conventions that the AI wouldn't otherwise know.

## Skills in This Repository

| Skill | Description |
|-------|-------------|
| [nandors-coding-style](nandors-coding-style/) | Functional coding conventions — factory functions over classes, immutability, pure functions, currying |
| [setup-agents-config](setup-agents-config/) | Set up AI agent configuration for multi-tool support |
| [setup-mcp](setup-mcp/) | Add MCP servers to any AI coding tool — knows the correct path, format, and schema for each harness |
| [write-ticket](write-ticket/) | Write, draft, or extend tickets/issues in Nandor's style for any tracker — Bug, Task, Story, and Investigation types, then file on approval |

## Global Config Locations

Each tool has a global config location where you can install skills:

| Tool | Global Folder |
|------|---------------|
| Cursor | `~/.cursor/` |
| Claude Code | `~/.claude/` |
| Codex (OpenAI) | `~/.codex/` |
| OpenCode | `~/.config/opencode/` |
| GitHub Copilot | `~/.agents/` (reads directly) |

> **Copilot duplicate skills:** Copilot scans `.agents/skills/`, `.claude/skills/`, and `.github/skills/` at project level (plus `~/.agents/skills/`, `~/.claude/skills/`, and `~/.copilot/skills/` at personal level). Since our setup symlinks `.claude/skills/` → `.agents/skills/`, Copilot discovers each skill twice. This appears to be a Copilot bug — the same resolved path should deduplicate. No action needed on your side; the duplicates are cosmetic.

## Skill Structure

Each skill is a folder containing:

```
skill-name/
├── SKILL.md      # Instructions for the AI (required)
└── README.md     # How to use the skill (optional)
```
