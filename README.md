# Agent Skills

This repository contains reusable AI agent skills for use with various coding assistants.

## What are Skills?

Skills are markdown-based instruction sets that teach AI coding assistants how to perform specific tasks. They provide domain knowledge, workflows, and conventions that the AI wouldn't otherwise know.

## Supported Tools

Skills in this repository are designed to work with:

- **Cursor** - Place in `.agents/skills/` or `~/.cursor/skills/`
- **Claude Code** - Place in `.claude/` (symlinked to `.agents/`)
- **Codex (OpenAI)** - Referenced via `AGENTS.md`
- **OpenCode** - Referenced via `AGENTS.md`

## Usage

### Global Installation (recommended)

Install skills globally so they're available across all projects:

```bash
# Cursor - symlink to personal skills
mkdir -p ~/.cursor/skills
ln -s ~/mydev/nandorszentpeteri/agents-skills/setup-agents-config ~/.cursor/skills/

# Codex - copy AGENTS.md content or reference skills in ~/.codex/AGENTS.md
mkdir -p ~/.codex

# Claude Code - reference in ~/.claude/CLAUDE.md
```

### Project-Level Installation

For project-specific skills:

```bash
# Copy to project
cp -r setup-agents-config /path/to/project/.agents/skills/

# Or symlink for easier updates
ln -s ~/mydev/nandorszentpeteri/agents-skills/setup-agents-config /path/to/project/.agents/skills/
```

## Structure

Each skill is a folder containing:

```
skill-name/
├── SKILL.md      # Main instructions (required)
├── reference.md  # Additional documentation (optional)
└── scripts/      # Utility scripts (optional)
```

## Creating New Skills

See the `setup-agents-config` skill for guidance on creating new skills and setting up agent configuration in repositories.
