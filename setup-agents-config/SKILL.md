---
name: setup-agents-config
description: >-
  Set up AI agent configuration for repositories or globally: AGENTS.md hub,
  per-skill and per-agent symlinks from ~/.agents into Claude, Cursor, Codex,
  and OpenCode (same harness everywhere — no whole-folder ~/.claude/skills
  → hub). Includes sync script, verification, and migration from older layouts.
  Only invoke when the user explicitly asks to set up agents config, configure
  AGENTS.md, initialize AI harnesses, or install skills/agents globally. Do NOT
  use proactively.
---

# Setting Up AI Agent Configuration

This skill sets up AI agent configuration at **project level** or **globally**. It creates the necessary files, folders, and symlinks so all AI coding tools can read your project instructions or share skills/agents across projects.

Ask the user which scope they need (project or global), or infer it from their request.

**Note:** This guide is for Linux and macOS. Windows paths may differ.

## Core Concept

**All tools read from `AGENTS.md` + `.agents/`** — this is the universal standard. The only exception is Claude Code, which requires `CLAUDE.md` + `.claude/`.

You don't need tool-specific folders like `.github/copilot-instructions.md`, `.cursor/`, or `.codex/` in your project.

## Tool Compatibility

| Tool | Project Discovery | Global Location |
|------|-------------------|-----------------|
| Codex (OpenAI) | `AGENTS.md` + `.agents/` | `~/.codex/` |
| Cursor | `AGENTS.md` + `.agents/` | `~/.cursor/` |
| GitHub Copilot | `AGENTS.md` + `.agents/` | None |
| OpenCode | `AGENTS.md` + `.agents/` | `~/.config/opencode/` |
| **Claude Code** | `CLAUDE.md` + `.claude/` | `~/.claude/` |

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
    ├── skills/            # Real dir: one symlink per item → ../.agents/skills/<name>
    └── agents/            # Real dir: one symlink per item → ../.agents/agents/<name>
```

**Why symlink `CLAUDE.md` instead of using `@AGENTS.md` content?** A symlink is simpler — one file instead of a file containing an import directive. Claude-specific additions (rules, tool config) go in `.claude/rules/` which Claude Code loads automatically.

**Why symlink subdirectories instead of the whole `.claude/` folder?** Claude Code stores its own files (like `settings.local.json`) in `.claude/`. Symlinking the entire `.agents` to `.claude` would either destroy those files or pollute `.agents/` with Claude-specific config. Symlinking only the `skills/` and `agents/` subdirectories keeps both tools happy.

**Per skill and per agent inside `.claude/` (and globally):** use a **real** `.claude/skills` and `.claude/agents` directory with **one symlink per item** pointing at `../.agents/skills/<name>` or `../.agents/agents/<name>`. Do **not** symlink the whole `skills` or `agents` folder into `.claude/` — that matches how Cursor and other tools expose the hub and avoids special-casing Claude.

### Check Existing Setup

Before creating files, check what already exists:

```bash
# Check for existing config files
ls -la AGENTS.md CLAUDE.md .agents .claude 2>/dev/null

# Detect old whole-folder symlink vs real directory
if [ -L .claude ]; then
  echo "WARNING: .claude is a symlink (old approach) — needs migration"
  readlink .claude
elif [ -d .claude ]; then
  echo ".claude is a real directory (correct)"
  ls -la .claude/skills .claude/agents 2>/dev/null
  if [ -L .claude/skills ] && [ "$(readlink -f .claude/skills 2>/dev/null)" = "$(readlink -f .agents/skills 2>/dev/null)" ]; then
    echo "NOTE: .claude/skills still points at the whole skills hub — migrate (see Project migration step 5)"
  fi
fi

# Check CLAUDE.md — should be a symlink to AGENTS.md
if [ -L CLAUDE.md ]; then
  echo "CLAUDE.md is a symlink to: $(readlink CLAUDE.md)"
elif [ -f CLAUDE.md ]; then
  echo "CLAUDE.md is a regular file (should be symlink to AGENTS.md)"
  cat CLAUDE.md
fi
```

Report missing or misconfigured items to the user before proceeding.

**Migrating from old `@AGENTS.md` content approach:** If `CLAUDE.md` is a regular file containing `@AGENTS.md`, replace it with a symlink:

```bash
rm CLAUDE.md
ln -sf AGENTS.md CLAUDE.md
```

**Migrating from old whole-folder symlink:** If `.claude` is a symlink to `.agents`, migrate to subdirectory symlinks:

```bash
# 1. Save any Claude-specific files from the symlinked directory
cp .claude/settings.local.json /tmp/claude-settings-backup.json 2>/dev/null

# 2. Remove the old symlink
rm .claude

# 3. Create .claude as a real directory
mkdir -p .claude

# 4. Restore Claude-specific files
cp /tmp/claude-settings-backup.json .claude/settings.local.json 2>/dev/null

# 5. Create .claude/skills and .claude/agents as real dirs; symlink each hub item
mkdir -p .claude/skills .claude/agents
find .agents/skills -mindepth 1 -maxdepth 1 | while IFS= read -r s; do
  ln -sf "../.agents/skills/$(basename "$s")" ".claude/skills/$(basename "$s")"
done
find .agents/agents -mindepth 1 -maxdepth 1 2>/dev/null | while IFS= read -r a; do
  ln -sf "../.agents/agents/$(basename "$a")" ".claude/agents/$(basename "$a")"
done
```

### Setup Commands

Run these commands in the repository root:

```bash
# 1. Create AGENTS.md (user adds project-specific instructions)
touch AGENTS.md

# 2. Symlink CLAUDE.md to AGENTS.md (Claude Code reads this)
ln -sf AGENTS.md CLAUDE.md

# 3. Create .agents directory with subdirectories
mkdir -p .agents/skills .agents/agents

# 4. Create .claude directory (preserves existing Claude Code config)
mkdir -p .claude

# 5. Symlink each skill and agent into .claude (real directories, per-item links)
mkdir -p .claude/skills .claude/agents
find .agents/skills -mindepth 1 -maxdepth 1 | while IFS= read -r s; do
  ln -sf "../.agents/skills/$(basename "$s")" ".claude/skills/$(basename "$s")"
done
find .agents/agents -mindepth 1 -maxdepth 1 2>/dev/null | while IFS= read -r a; do
  ln -sf "../.agents/agents/$(basename "$a")" ".claude/agents/$(basename "$a")"
done
```

### Verification

After setup, verify the structure:

```bash
ls -la AGENTS.md CLAUDE.md .agents/ .claude/
readlink CLAUDE.md       # Should show: AGENTS.md
# Each entry under .claude/skills and .claude/agents should be a symlink into .agents/
ls -la .claude/skills/ .claude/agents/
readlink .claude/skills/<skill-name>   # Should show: ../.agents/skills/<skill-name>
readlink .claude/agents/<agent-name>   # Should show: ../.agents/agents/<agent-name>
```

## Global Setup

Global setup lets you share skills and agents across all projects by keeping them in a central `~/.agents/` directory and symlinking into each tool's global config. The steps below target **Claude Code, Cursor, Codex, and OpenCode**.

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

Each tool's global directory gets a symlink to `~/.agents/AGENTS.md`. Under each tool, `skills/` and `agents/` are **real directories** whose **children** are symlinks into `~/.agents/skills/<name>` and `~/.agents/agents/<name>` — **including Claude** (same harness as Cursor, Codex, and OpenCode).

```
~/.claude/
├── CLAUDE.md -> ~/.agents/AGENTS.md   # Symlink
├── rules/                              # Claude-specific rules (optional)
├── skills/                             # Real dir; entries are symlinks to hub
└── agents/                             # Real dir; entries are symlinks to hub

~/.cursor/
├── AGENTS.md -> ~/.agents/AGENTS.md   # Symlink
├── skills/
└── agents/

~/.codex/
├── AGENTS.md -> ~/.agents/AGENTS.md   # Symlink
├── skills/
└── agents/

~/.config/opencode/
├── AGENTS.md -> ~/.agents/AGENTS.md   # Symlink
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

# Check tool-specific skills/agents dirs
ls -la ~/.claude/skills ~/.cursor/skills ~/.codex/skills ~/.config/opencode/skills 2>/dev/null
ls -la ~/.claude/agents ~/.cursor/agents ~/.codex/agents ~/.config/opencode/agents 2>/dev/null

# Detect old Claude global layout (~/.claude/skills -> whole ~/.agents/skills)
if [ -L ~/.claude/skills ]; then
  if [ "$(readlink -f ~/.claude/skills 2>/dev/null)" = "$(readlink -f ~/.agents/skills 2>/dev/null)" ]; then
    echo "NOTE: ~/.claude/skills points at the whole hub — migrate to per-skill (see Global Setup step 5–6)"
  fi
fi
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

# 5. Migrate Claude if ~/.claude/skills (or agents) is still a symlink to the whole hub
for kind in skills agents; do
  tgt="$HOME/.claude/$kind"
  if [ -L "$tgt" ] && [ "$(readlink -f "$tgt" 2>/dev/null)" = "$(readlink -f "$HOME/.agents/$kind" 2>/dev/null)" ]; then
    rm "$tgt"
    mkdir -p "$tgt"
  fi
done

# 6. Mirror every skill and agent from ~/.agents into each tool (idempotent)
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
```

To add **one** new skill or agent later, link it into the hub first, then re-run step **6** (or run `mirror_hub_kind skills` / `mirror_hub_kind agents` only).

**Key points:**
- GitHub Copilot is skipped (no global config directory)
- Claude Code uses `CLAUDE.md` as filename but points to the same `~/.agents/AGENTS.md`
- **Same harness for every tool:** real `skills/` and `agents/` dirs; **only child entries** are symlinks into `~/.agents/` — **never** `~/.claude/skills` → `~/.agents/skills` as a single directory symlink (harder to align with Cursor and tool-owned files under those paths)
- Claude-specific global rules go in `~/.claude/rules/` (not in AGENTS.md)
- If the user doesn't specify which skills/agents to install, list available ones under `~/.agents/` and ask
- Existing tool configs are never overwritten — we only add or refresh symlinks under `skills/` and `agents/` subdirs

### Global Verification

```bash
# Verify AGENTS.md symlinks
readlink ~/.claude/CLAUDE.md          # Should show: ~/.agents/AGENTS.md
readlink ~/.cursor/AGENTS.md          # Should show: ~/.agents/AGENTS.md
readlink ~/.codex/AGENTS.md           # Should show: ~/.agents/AGENTS.md
readlink ~/.config/opencode/AGENTS.md  # Should show: ~/.agents/AGENTS.md

# Verify skills/agents symlinks resolve correctly (each tool: real dir, children are links)
ls -la ~/.claude/skills/ ~/.cursor/skills/ ~/.codex/skills/ ~/.config/opencode/skills/
ls -la ~/.claude/agents/ ~/.cursor/agents/ ~/.codex/agents/ ~/.config/opencode/agents/

# ~/.claude/skills should NOT be a symlink to the entire hub directory
test ! -L ~/.claude/skills || test "$(readlink -f ~/.claude/skills)" != "$(readlink -f ~/.agents/skills)"

# Check a specific symlink target
readlink ~/.claude/skills/<skill-name>
# Should show: ~/.agents/skills/<skill-name> (or the expanded absolute path)
```

### Global Troubleshooting

**AGENTS.md not picked up by a tool:**
- Check the symlink exists and resolves: `readlink ~/.cursor/AGENTS.md`
- Verify the central file exists: `ls -la ~/.agents/AGENTS.md`
- For Claude Code, verify the filename is `CLAUDE.md`: `readlink ~/.claude/CLAUDE.md`

**Symlink broken (skill not found by tool):**
- Check the target exists: `ls -la ~/.agents/skills/<skill-name>`
- Recreate the symlink: `ln -sf ~/.agents/skills/<skill-name> ~/.claude/skills/<skill-name>`

**Tool not picking up global skills:**
- Verify the tool supports the global location (see Tool Compatibility table above)
- Check the symlink is in the correct directory for that tool

## Troubleshooting

**Claude Code not finding instructions:**
- Verify `CLAUDE.md` is a symlink to `AGENTS.md`: `readlink CLAUDE.md`
- If it's a regular file with `@AGENTS.md` content, migrate to symlink (see migration section above)
- Verify `.claude/` is a real directory (not a symlink): `test -L .claude && echo "symlink (needs migration)" || echo "real dir (correct)"`
- Verify `.claude/skills` and `.claude/agents` are **directories** (not symlinks to the whole `.agents/skills` or `.agents/agents` tree): `ls -ld .claude/skills .claude/agents`
- If `.claude` is a symlink to `.agents`, migrate to subdirectory symlinks (see "Migrating from old whole-folder symlink" above)

**Codex not loading instructions:**
- Check `AGENTS.md` exists at repo root
- Run `codex --ask-for-approval never "Summarize current instructions"`

**Other tools not reading config:**
- Verify `AGENTS.md` exists at repo root
- Check `.agents/` directory exists

**Global symlinks not working:**
- Check symlink target exists: `ls -la ~/.agents/skills/<skill-name>`
- Verify symlink resolves: `readlink ~/.claude/skills/<skill-name>`
- Ensure central directory was created: `ls ~/.agents/`
