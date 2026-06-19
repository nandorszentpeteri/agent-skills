# git-worktree

Manages the full lifecycle of git worktrees for ticket-based development: creation, implementation, and cleanup.

## What It Does

1. **Triggers** when a Jira ticket is mentioned — asks if you want a worktree
2. **Creates** a worktree branched from the remote's default branch (auto-detected), copies `node_modules` and `.husky`
3. **Cleans up** the worktree, local branch, and folder when done

## Naming Convention

```
my-project/            # main repo
my-project-PROJ-3541/   # worktree
```

## How to Invoke

Mention a Jira ticket and the skill will prompt you, or explicitly ask:

```
"Set up a worktree for PROJ-3541"
"Clean up the PROJ-3541 worktree"
```
