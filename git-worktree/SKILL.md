---
name: git-worktree
description: >-
  Manage git worktrees for ticket-based development: create, switch, and clean
  up worktrees with the correct naming convention. Proactively ask the user if
  they want a worktree when a Jira ticket is mentioned and implementation is
  about to begin.
---

# Git Worktree Workflow

This skill manages the full lifecycle of git worktrees for isolated, ticket-based development.

## When to Trigger

Proactively ask **"Do you want to use a worktree for this?"** whenever:
- A Jira ticket ID is mentioned (e.g., `PROJ-3541`, `TEAM-1234`)
- A Jira ticket URL is shared
- The user says they want to start working on a ticket

Do NOT ask if the user is already inside a worktree for that ticket.

## Naming Convention

Worktrees are created as siblings of the main repo:

```
<parent-dir>/
  <repo>/                  # main repo (e.g., my-project)
  <repo>-<BRANCH>/         # worktree (e.g., my-project-PROJ-3541)
```

The branch name matches the Jira ticket ID (e.g., `PROJ-3541`).

## Detecting the Base Branch

Do NOT hardcode the base branch. It could be `develop`, `main`, `master`, or something else. Always detect it dynamically:

```bash
git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|refs/remotes/origin/||'
```

If that fails (e.g., `origin/HEAD` is not set), fall back to the network call:

```bash
git remote show origin | grep 'HEAD branch' | awk '{print $NF}'
```

Use the result as `<BASE_BRANCH>` in all commands below.

## Phase 1: Creation

When the user confirms they want a worktree:

1. **Prune stale worktrees** to avoid conflicts from manually deleted directories:
   ```bash
   git worktree prune
   ```
2. **Fetch latest** from the main repo
3. **Detect the base branch** (see above)
4. **Create the worktree and branch**:
   ```bash
   cd <main-repo>
   git fetch origin
   git worktree add -b <BRANCH> ../<repo>-<BRANCH> origin/<BASE_BRANCH>
   ```
   If the branch already exists **locally**, use `git worktree add ../<repo>-<BRANCH> <BRANCH>` instead.
   If the branch already exists **remotely** but not locally, use `git worktree add ../<repo>-<BRANCH> origin/<BRANCH>`.
   If the worktree **directory** already exists, inform the user and ask how to proceed.
5. **Copy shared artifacts** that aren't in git but are needed to work. For JS/Node projects:
   ```bash
   cp -a <main-repo>/node_modules ../<repo>-<BRANCH>/
   cp -a <main-repo>/.husky ../<repo>-<BRANCH>/
   ```
   Also copy `.env*` files if they exist (they often contain local dev config). For non-JS projects, skip this step and run the project's dependency install command instead.
6. **Switch working directory** to the new worktree using the `EnterWorktree` tool. If `EnterWorktree` is not available, use `cd` in Bash (note: `cd` does not persist between Bash calls — use absolute paths for all subsequent commands).

## Phase 2: Implementation

Work normally in the worktree. Before pushing, rebase onto the latest base branch:

```bash
git fetch origin <BASE_BRANCH>
git rebase origin/<BASE_BRANCH>
```

If the rebase has **no conflicts**, push normally. If there **are conflicts**, do not guess resolutions — surface the conflict files and ask the user. After a successful rebase, use `git push --force-with-lease` if the branch was already pushed (this is safe — it only overwrites your own commits).

## Phase 3: Cleanup

After the MR is merged (or when the user asks to clean up), remove all traces:

1. **Remove the worktree** (from the main repo):
   ```bash
   cd <main-repo>
   git worktree remove ../<repo>-<BRANCH>
   ```
2. **Delete the local branch** (safe delete first):
   ```bash
   git branch -d <BRANCH>
   ```
   If this fails because the branch is not fully merged, inform the user and only use `git branch -D` after they confirm.
3. **Verify cleanup**:
   ```bash
   git worktree list
   ```

If the worktree folder still exists after removal (e.g., due to untracked files), ask the user before force-removing it.

## Guidelines

- Always confirm with the user before creating or cleaning up a worktree
- If `node_modules` or `.husky` don't exist in the main repo, skip copying them and run `npm install` instead
- Keep the main repo on its current branch — never switch its branch as part of worktree operations
- Use `ExitWorktree` (if available) when returning to the main repo after cleanup
