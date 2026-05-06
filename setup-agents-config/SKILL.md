---
name: setup-agents-config
description: >-
  Set up AI agent configuration (project or global): AGENTS.md as source of
  truth, symlinks for Claude/Cursor/Codex/OpenCode, migration from older
  layouts. Only invoke when explicitly asked to set up agents config or install
  skills globally.
---

# Setting Up AI Agent Configuration

This skill sets up AI agent configuration at **project level** or **globally**. It creates the necessary files, folders, and symlinks so all AI coding tools can read your project instructions or share skills/agents across projects.

Ask the user which scope they need (project or global), or infer it from their request.

**Note:** This guide is for Linux and macOS. On Windows, use WSL — native Windows paths are not covered.

## Core Concept

**All tools read from `AGENTS.md` + `.agents/`** — this is the universal standard. The only exception is Claude Code, which requires `CLAUDE.md` + `.claude/`.

You don't need tool-specific folders like `.github/copilot-instructions.md`, `.cursor/`, or `.codex/` in your project.

## Tool Compatibility

| Tool | Project Discovery | Global Location |
|------|-------------------|-----------------|
| Codex (OpenAI) | `AGENTS.md` + `.agents/` | `~/.codex/` |
| Cursor | `AGENTS.md` + `.agents/` | `~/.cursor/` |
| GitHub Copilot | `AGENTS.md` + `.agents/`, `.claude/`, `.github/` | `~/.agents/`, `~/.claude/`, `~/.copilot/` |
| OpenCode | `AGENTS.md` + `.agents/` | `~/.config/opencode/` |
| **Claude Code** | `CLAUDE.md` + `.claude/` | `~/.claude/` |

**Copilot duplicate skills caveat:** Copilot scans `.agents/skills/`, `.claude/skills/`, and `.github/skills/` (project) and `~/.agents/skills/`, `~/.claude/skills/`, `~/.copilot/skills/` (personal). Because our setup symlinks `.claude/skills/` → `.agents/skills/`, Copilot discovers each skill twice. This is a Copilot bug (it should deduplicate by resolved path). The duplicates are cosmetic and don't affect functionality — no action needed.

## Symlink Strategy

Two different strategies depending on scope:

**Project level — whole-folder symlinks.** `.claude/skills` → `.agents/skills` and `.claude/agents` → `.agents/agents`. Only Claude Code needs this — all other tools read `.agents/` directly. Safe because no tool writes to project-level `skills/` or `agents/` directories.

**Global level — per-item symlinks.** Each tool's `skills/` and `agents/` directories are **real directories** containing individual symlinks into `~/.agents/skills/<name>` and `~/.agents/agents/<name>`. This is necessary because tools can have their own managed skills in their global `skills/` directory (e.g. Cursor uses `skills-cursor/` but may also write to `skills/`). A whole-folder symlink would overwrite those.

**Why symlink `CLAUDE.md` instead of using `@AGENTS.md` content?** A symlink is simpler — one file instead of a file containing an import directive. Claude-specific additions (rules, tool config) go in `.claude/rules/` which Claude Code loads automatically.

**Why not symlink the whole `.claude/` folder?** Claude Code stores its own files (like `settings.local.json`) in `.claude/`. Symlinking `.agents` to `.claude` would either destroy those files or pollute `.agents/` with Claude-specific config.

## Project Setup

### File Structure

```
repo-root/
├── AGENTS.md              # Single source of truth (all tools)
├── CLAUDE.md -> AGENTS.md # Symlink (Claude Code only)
├── .agents/               # Configuration directory
│   ├── skills/
│   └── agents/
└── .claude/               # Real directory (Claude Code owns it)
    ├── settings.local.json  # Claude Code-specific config (preserved)
    ├── skills/ -> ../.agents/skills   # Whole-folder symlink
    └── agents/ -> ../.agents/agents   # Whole-folder symlink
```

### Check Existing Setup

Before creating files, check what already exists:

```bash
# Check for existing config files
ls -la AGENTS.md CLAUDE.md .agents .claude 2>/dev/null

# Detect old whole-folder symlink of .claude itself
if [ -L .claude ]; then
  echo "WARNING: .claude is a symlink (old approach) — needs migration"
  readlink .claude
elif [ -d .claude ]; then
  echo ".claude is a real directory (correct)"
  ls -la .claude/skills .claude/agents 2>/dev/null
fi

# Check CLAUDE.md — should be a symlink to AGENTS.md
if [ -L CLAUDE.md ]; then
  echo "CLAUDE.md is a symlink to: $(readlink CLAUDE.md)"
elif [ -f CLAUDE.md ]; then
  echo "CLAUDE.md is a regular file (should be symlink to AGENTS.md)"
fi
```

Report missing or misconfigured items to the user before proceeding.

### Migration

**From `@AGENTS.md` content approach:** If `CLAUDE.md` is a regular file containing `@AGENTS.md`, replace it with a symlink:

```bash
rm CLAUDE.md
ln -sf AGENTS.md CLAUDE.md
```

**From old whole-folder `.claude` symlink:** If `.claude` itself is a symlink to `.agents`:

```bash
# 1. Save any Claude-specific files
cp .claude/settings.local.json /tmp/claude-settings-backup.json 2>/dev/null

# 2. Remove the old symlink and recreate as real directory
rm .claude
mkdir -p .claude

# 3. Restore Claude-specific files
cp /tmp/claude-settings-backup.json .claude/settings.local.json 2>/dev/null

# 4. Symlink skills and agents folders
ln -sf ../.agents/skills .claude/skills
ln -sf ../.agents/agents .claude/agents
```

**From per-item symlinks:** If `.claude/skills/` is a real directory full of individual symlinks:

```bash
# Verify these are directories of symlinks, not real files with content
if find .claude/skills -maxdepth 1 -not -type l -not -type d | grep -q .; then
  echo "WARNING: .claude/skills contains non-symlink files — back up before removing"
else
  rm -rf .claude/skills .claude/agents
  ln -sf ../.agents/skills .claude/skills
  ln -sf ../.agents/agents .claude/agents
fi
```

### Setup Commands

Run these commands in the repository root:

```bash
# 1. Create AGENTS.md if it doesn't exist
#    (tell the user it needs project-specific instructions — an empty file is a no-op)
test -f AGENTS.md || touch AGENTS.md

# 2. Symlink CLAUDE.md to AGENTS.md (Claude Code reads this)
ln -sf AGENTS.md CLAUDE.md

# 3. Create .agents directory with subdirectories
mkdir -p .agents/skills .agents/agents

# 4. Create .claude directory (preserves existing Claude Code config)
mkdir -p .claude

# 5. Symlink skills and agents folders into .claude (whole-folder)
ln -sf ../.agents/skills .claude/skills
ln -sf ../.agents/agents .claude/agents
```

### Verification

```bash
ls -la AGENTS.md CLAUDE.md .agents/ .claude/
readlink CLAUDE.md           # Should show: AGENTS.md
readlink .claude/skills      # Should show: ../.agents/skills
readlink .claude/agents      # Should show: ../.agents/agents
```

## Global Setup

Global setup lets you share skills and agents across all projects by keeping them in a central `~/.agents/` directory and symlinking into each tool's global config.

### Global Directory Structure

```
~/.agents/
├── AGENTS.md              # Global shared rules (single source of truth)
├── skills/
│   └── <skill-name>/
│       └── SKILL.md
└── agents/
    └── <agent-name>/
        └── ...
```

Each tool's global directory gets a symlink to `~/.agents/AGENTS.md`. Under each tool, `skills/` and `agents/` are **real directories** whose **children** are per-item symlinks into `~/.agents/skills/<name>` and `~/.agents/agents/<name>`. This preserves any tool-managed skills already in those directories.

```
~/.claude/
├── CLAUDE.md -> ~/.agents/AGENTS.md   # Symlink
├── rules/                              # Claude-specific rules (optional)
├── skills/                             # Real dir; entries are symlinks to hub
└── agents/                             # Real dir; entries are symlinks to hub

~/.cursor/
├── AGENTS.md -> ~/.agents/AGENTS.md   # Symlink
├── skills/                             # Real dir; may contain tool-managed skills too
└── agents/

~/.codex/
├── AGENTS.md -> ~/.agents/AGENTS.md
├── skills/
└── agents/

~/.config/opencode/
├── AGENTS.md -> ~/.agents/AGENTS.md
├── skills/
└── agents/
```

### Check Existing Setup

```bash
# Check if central directory and AGENTS.md exist
ls -la ~/.agents/AGENTS.md ~/.agents/skills ~/.agents/agents 2>/dev/null

# Check tool-specific global AGENTS.md symlinks
for f in ~/.claude/CLAUDE.md ~/.cursor/AGENTS.md ~/.codex/AGENTS.md ~/.config/opencode/AGENTS.md; do
  if [ -L "$f" ]; then
    echo "$f -> $(readlink "$f")"
  elif [ -f "$f" ]; then
    echo "$f exists but is not a symlink (should be migrated)"
  else
    echo "$f missing"
  fi
done

# Check tool-specific skills/agents dirs exist
ls -la ~/.claude/skills ~/.cursor/skills ~/.codex/skills ~/.config/opencode/skills 2>/dev/null
ls -la ~/.claude/agents ~/.cursor/agents ~/.codex/agents ~/.config/opencode/agents 2>/dev/null
```

Report what already exists to the user before proceeding.

### Setup Commands

```bash
# 1. Create central directories and AGENTS.md
mkdir -p ~/.agents/skills ~/.agents/agents
touch ~/.agents/AGENTS.md

# 2. Copy or clone skills/agents into ~/.agents/
#    (method left to user — git clone, manual copy, etc.)

# 3. Symlink AGENTS.md into each tool's global config
#    Claude Code uses CLAUDE.md filename (symlinked to same file)
mkdir -p ~/.claude
ln -sf ~/.agents/AGENTS.md ~/.claude/CLAUDE.md
mkdir -p ~/.cursor
ln -sf ~/.agents/AGENTS.md ~/.cursor/AGENTS.md
mkdir -p ~/.codex
ln -sf ~/.agents/AGENTS.md ~/.codex/AGENTS.md
mkdir -p ~/.config/opencode
ln -sf ~/.agents/AGENTS.md ~/.config/opencode/AGENTS.md

# 4. Create tool-specific skills/agents parent dirs (real directories, not symlinks)
mkdir -p ~/.claude/skills ~/.claude/agents
mkdir -p ~/.cursor/skills ~/.cursor/agents
mkdir -p ~/.codex/skills ~/.codex/agents
mkdir -p ~/.config/opencode/skills ~/.config/opencode/agents

# 5. Mirror every skill and agent from ~/.agents into each tool (per-item, idempotent)
AG="$HOME/.agents"
mirror_hub_kind() {
  local kind="$1"
  local sub="$AG/$kind"
  mkdir -p "$sub"
  for base in \
    "$HOME/.claude/$kind" \
    "$HOME/.cursor/$kind" \
    "$HOME/.codex/$kind" \
    "$HOME/.config/opencode/$kind"; do
    mkdir -p "$base"
    find "$sub" -mindepth 1 -maxdepth 1 | while IFS= read -r item; do
      name=$(basename "$item")
      ln -sf "$sub/$name" "$base/$name"
    done
  done
}
mirror_hub_kind skills
mirror_hub_kind agents

# If nothing was mirrored, inform the user
if [ -z "$(find "$AG/skills" -mindepth 1 -maxdepth 1 2>/dev/null)" ] && \
   [ -z "$(find "$AG/agents" -mindepth 1 -maxdepth 1 2>/dev/null)" ]; then
  echo "No skills or agents found in ~/.agents/ — add them first, then re-run the mirror step."
fi
```

To add **one** new skill or agent later, link it into the hub first, then re-run step **5** (or run `mirror_hub_kind skills` / `mirror_hub_kind agents` only).

**Key points:**
- GitHub Copilot reads directly from `~/.agents/` — no separate global folder needed (it also reads `~/.claude/` and `~/.copilot/`, but `~/.agents/` is sufficient)
- Claude Code uses `CLAUDE.md` as filename but points to the same `~/.agents/AGENTS.md`
- **Per-item symlinks at global level** preserve any tool-managed skills in those directories
- Claude-specific global rules go in `~/.claude/rules/` (not in AGENTS.md)
- If the user doesn't specify which skills/agents to install, list available ones under `~/.agents/` and ask
- Existing non-symlink files are never overwritten — existing symlinks are refreshed by `ln -sf`

### Global Verification

```bash
# Verify AGENTS.md symlinks
readlink ~/.claude/CLAUDE.md          # Should show: ~/.agents/AGENTS.md
readlink ~/.cursor/AGENTS.md          # Should show: ~/.agents/AGENTS.md
readlink ~/.codex/AGENTS.md           # Should show: ~/.agents/AGENTS.md
readlink ~/.config/opencode/AGENTS.md  # Should show: ~/.agents/AGENTS.md

# Verify skills/agents per-item symlinks resolve correctly
ls -la ~/.claude/skills/ ~/.cursor/skills/
# Check all symlinks resolve:
find ~/.claude/skills -maxdepth 1 -type l -exec sh -c 'echo "{} -> $(readlink "{}")"' \;
```

### Global Troubleshooting

**AGENTS.md not picked up by a tool:**
- Check the symlink exists and resolves: `readlink ~/.cursor/AGENTS.md`
- Verify the central file exists: `ls -la ~/.agents/AGENTS.md`
- For Claude Code, verify the filename is `CLAUDE.md`: `readlink ~/.claude/CLAUDE.md`

**Symlink broken (skill not found by tool):**
- Check the target exists: `ls -la ~/.agents/skills/<skill-name>`
- Recreate the symlink: `ln -sf ~/.agents/skills/<skill-name> ~/.claude/skills/<skill-name>`

**New skill not showing up:**
- Verify it's in the hub: `ls ~/.agents/skills/<skill-name>/`
- Re-run the mirror script (step 5) to create per-item symlinks in all tools

**Tool not picking up global skills:**
- Verify the tool supports the global location (see Tool Compatibility table above)
- Check the symlink is in the correct directory for that tool

## Troubleshooting

**Claude Code not finding instructions:**
- Verify `CLAUDE.md` is a symlink to `AGENTS.md`: `readlink CLAUDE.md`
- If it's a regular file with `@AGENTS.md` content, migrate to symlink (see migration section above)
- Verify `.claude/` is a real directory (not a symlink): `test -L .claude && echo "symlink (needs migration)" || echo "real dir (correct)"`
- Project level: verify `.claude/skills` is a symlink to `.agents/skills`: `readlink .claude/skills`
- Global level: verify per-item symlinks exist: `ls -la ~/.claude/skills/`
- If `.claude` is a symlink to `.agents`, migrate (see Migration section above)

**Codex not loading instructions:**
- Check `AGENTS.md` exists at repo root
- Run `codex --ask-for-approval never "Summarize current instructions"`

**Other tools not reading config:**
- Verify `AGENTS.md` exists at repo root
- Check `.agents/` directory exists

**Global symlinks not working:**
- Check the hub exists: `ls ~/.agents/skills/`
- Verify per-item symlink resolves: `readlink ~/.claude/skills/<skill-name>`
- Ensure central directory was created: `ls ~/.agents/`
