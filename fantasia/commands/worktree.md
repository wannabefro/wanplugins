---
description: Manage git worktrees for Fantasia tasks - list, switch, finish, or cleanup isolated feature workspaces
argument-hint: "<list|switch|finish|cleanup> [task]"
allowed-tools: ["Read", "Bash", "Write", "Glob", "AskUserQuestion"]
---

You are managing git worktrees for Fantasia workflows. Worktrees provide isolated workspaces for each task/ticket.

## Determine Fantasia Directory

All Fantasia outputs go to `~/.claude/fantasia/<project>/` by default, but this can be overridden with the `FANTASIA_DIR` environment variable.

```bash
# Use environment variable if set, otherwise calculate default
if [ -z "$FANTASIA_DIR" ]; then
  REPO_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
  PROJECT_SLUG=$(basename "$REPO_ROOT" | tr '[:upper:]' '[:lower:]' | tr ' ' '-')
  FANTASIA_DIR="$HOME/.claude/fantasia/$PROJECT_SLUG"
fi

echo "FANTASIA_DIR=$FANTASIA_DIR"
```

## Parse Command

Extract the subcommand from `$ARGUMENTS`:
- `list` - Show all Fantasia-managed worktrees
- `switch <task>` - Switch to a task's worktree
- `finish` - Merge current worktree and cleanup
- `cleanup` - Remove worktrees for completed/merged branches
- (empty) - Show help

## Subcommand: list

Show all worktrees with their Fantasia status:

```bash
# Get all worktrees
git worktree list

# Since all worktrees share the same FANTASIA_DIR, check if it exists
if [ -f "$FANTASIA_DIR/fantasia-state.md" ]; then
  echo "FANTASIA_STATE_EXISTS=true"
fi
```

Present as:
```
🌳 Fantasia Worktrees
=====================

| Worktree | Branch | Task | Status |
|----------|--------|------|--------|
| ../fantasia-proj-123 | feat/proj-123 | PROJ-123 | building |
| ../fantasia-team-456 | feat/team-456 | TEAM-456 | review complete |
| . (main) | main | - | - |

Switch: /fantasia:worktree switch proj-123
Finish: /fantasia:worktree finish (from within worktree)
```

## Subcommand: switch <task>

Switch context to a task's worktree:

```bash
TASK="$1"
TASK_LOWER=$(echo "$TASK" | tr '[:upper:]' '[:lower:]')

# Find worktree for this task
WORKTREE_PATH=$(git worktree list --porcelain | grep -B2 "feat/$TASK_LOWER\|feat/$TASK" | grep "^worktree" | cut -d' ' -f2 | head -1)

if [ -z "$WORKTREE_PATH" ]; then
  echo "No worktree found for task: $TASK"
  exit 1
fi

echo "WORKTREE_PATH=$WORKTREE_PATH"
```

### Environment Setup for Worktree

After switching, ensure the environment is set up:

```bash
cd "$WORKTREE_PATH"

# Enable direnv if .envrc exists (critical for Klaviyo repos)
if [ -f .envrc ]; then
  echo "Enabling direnv for worktree..."
  direnv allow
fi

# Activate virtual environment if present
if [ -f .venv/bin/activate ]; then
  echo "Found Python virtual environment"
fi

# Note nvm requirements if present
if [ -f .nvmrc ]; then
  echo "Found .nvmrc - run 'nvm use' to set Node version"
fi
```

Present:
```
🌳 Switching to: <task>

Worktree: <path>
Branch: feat/<task>

## Environment Setup
<If .envrc found>
✅ direnv enabled (run 'direnv allow' if prompted)
</If>
<If .venv found>
📦 Virtual environment available: source .venv/bin/activate
</If>
<If .nvmrc found>
📦 Node version specified: run 'nvm use'
</If>

To continue working, open a new terminal in:
  cd <worktree-path>

Or if using Claude Code:
  claude --cd <worktree-path>

**Important**: If using direnv, run `direnv allow` in the new terminal.

Current Fantasia state:
<read and display $FANTASIA_DIR/fantasia-state.md>
```

## Subcommand: finish

Complete work in current worktree - merge and cleanup:

### Pre-checks
```bash
# Verify we're in a worktree (not main)
CURRENT_WT=$(git rev-parse --show-toplevel)
MAIN_WT=$(git worktree list | head -1 | awk '{print $1}')

if [ "$CURRENT_WT" = "$MAIN_WT" ]; then
  echo "ERROR: Not in a worktree. Run this from a Fantasia worktree."
  exit 1
fi

# Get current branch
BRANCH=$(git branch --show-current)

# Check for uncommitted changes
git status --porcelain
```

### Finish workflow

1. **Check for uncommitted changes**:
   ```
   You have uncommitted changes. Would you like to:
   1. Commit them now
   2. Stash them
   3. Discard them
   4. Cancel finish
   ```

2. **Confirm merge target**:
   ```
   Ready to finish: <branch>

   This will:
   1. Switch to main repo
   2. Merge <branch> into main
   3. Delete the branch
   4. Remove this worktree

   Proceed? (y/n)
   ```

3. **Execute merge**:
   ```bash
   BRANCH=$(git branch --show-current)
   MAIN_REPO=$(git worktree list | head -1 | awk '{print $1}')

   # Go to main repo
   cd "$MAIN_REPO"

   # Merge
   git merge "$BRANCH" --no-ff -m "Merge $BRANCH"

   # Remove worktree and branch
   git worktree remove "$CURRENT_WT"
   git branch -d "$BRANCH"
   ```

4. **Update Registry**:
   ```bash
   # Remove entry from worktrees.md
   TASK_SLUG=$(basename "$CURRENT_WT" | sed 's/fantasia-//')
   if [ -f "$FANTASIA_DIR/worktrees.md" ]; then
     sed -i '' "/| $TASK_SLUG |/d" "$FANTASIA_DIR/worktrees.md"
   fi
   ```

5. **Report**:
   ```
   ✅ Finished: <task>

   - Merged: <branch> → main
   - Removed worktree: <path>
   - Deleted branch: <branch>
   - Registry updated

   You're now in: <main-repo-path>
   ```

### Handle merge conflicts
```
⚠️ Merge conflict detected

Files with conflicts:
<list>

Options:
1. Resolve conflicts manually, then run /fantasia:worktree finish again
2. Abort merge and stay in worktree
```

## Subcommand: cleanup

Remove worktrees for branches that have been merged or deleted:

### Step 1: Find Stale Worktrees

```bash
# Get main repo path
MAIN_REPO=$(git worktree list | head -1 | awk '{print $1}')
MAIN_BRANCH=$(cd "$MAIN_REPO" && git rev-parse --abbrev-ref HEAD)

# List all worktrees and check their status
STALE_WORKTREES=""

for worktree_path in $(git worktree list --porcelain | grep "^worktree " | cut -d' ' -f2); do
  # Skip main worktree
  if [ "$worktree_path" = "$MAIN_REPO" ]; then
    continue
  fi

  # Only check fantasia worktrees
  if ! echo "$worktree_path" | grep -q "fantasia-"; then
    continue
  fi

  # Get the branch for this worktree
  BRANCH=$(git worktree list --porcelain | grep -A2 "^worktree $worktree_path$" | grep "^branch " | cut -d' ' -f2 | sed 's|refs/heads/||')

  # Check if branch has been merged to main
  if [ -n "$BRANCH" ]; then
    MERGED=$(cd "$MAIN_REPO" && git branch --merged "$MAIN_BRANCH" | grep -w "$BRANCH")
    if [ -n "$MERGED" ]; then
      STALE_WORKTREES="$STALE_WORKTREES\n- $worktree_path (branch '$BRANCH' merged)"
    fi
  fi

  # Check if branch was deleted
  if [ -z "$BRANCH" ]; then
    STALE_WORKTREES="$STALE_WORKTREES\n- $worktree_path (branch deleted)"
  fi
done

# Also prune any worktrees whose directories no longer exist
git worktree prune --dry-run 2>&1 | grep "Removing" && NEEDS_PRUNE=true
```

### Step 2: Present Findings

If no stale worktrees found:
```
🧹 Worktree Cleanup
===================

No stale worktrees found. All Fantasia worktrees are active.

Current worktrees:
<list active worktrees>
```

If stale worktrees found:
```
🧹 Worktree Cleanup
===================

Found stale worktrees:
<STALE_WORKTREES list>

These worktrees have branches that are either:
- Already merged to main
- Deleted from the repository

Remove these? (y/n)
```

### Step 3: Execute Cleanup

If user confirms:

```bash
for worktree_path in $STALE_PATHS; do
  echo "Removing: $worktree_path"

  # Get branch name before removing
  BRANCH=$(git worktree list --porcelain | grep -A2 "^worktree $worktree_path$" | grep "^branch " | cut -d' ' -f2 | sed 's|refs/heads/||')

  # Remove the worktree
  git worktree remove "$worktree_path" --force

  # Delete the branch if it still exists
  if [ -n "$BRANCH" ]; then
    git branch -d "$BRANCH" 2>/dev/null || git branch -D "$BRANCH" 2>/dev/null
  fi
done

# Prune any remaining references
git worktree prune

# Update registry
# Remove cleaned entries from $FANTASIA_DIR/worktrees.md
```

### Step 4: Update Registry

After cleanup, update `$FANTASIA_DIR/worktrees.md` to remove cleaned entries:

```bash
if [ -f "$FANTASIA_DIR/worktrees.md" ]; then
  for task in $CLEANED_TASKS; do
    # Remove line containing this task from registry
    sed -i '' "/| $task |/d" "$FANTASIA_DIR/worktrees.md"
  done
fi
```

### Step 5: Report

```
✅ Cleanup complete

Removed:
- ../fantasia-old-task (branch: feat/old-task)
- ../fantasia-abandoned (branch: feat/abandoned)

Registry updated: $FANTASIA_DIR/worktrees.md
```

Present:
```
🧹 Worktree Cleanup
===================

Found stale worktrees:
- ../fantasia-old-task (branch merged)
- ../fantasia-abandoned (branch deleted)

Remove these? (y/n)
```

## Help (no arguments)

```
🌳 Fantasia Worktree Management
===============================

Commands:
  /fantasia:worktree list          Show all Fantasia worktrees
  /fantasia:worktree switch <task> Switch to a task's worktree
  /fantasia:worktree finish        Merge current worktree, cleanup
  /fantasia:worktree cleanup       Remove stale worktrees

Worktrees are created automatically with:
  /fantasia:ticket PROJ-123 --worktree

All worktrees share the Fantasia directory at ~/.claude/fantasia/<project>/:
- Shared codebase/ maps
- Shared plans/ for all tasks
- Single fantasia-state.md for session state
- Single worktrees.md registry
```

## Registry Management

Maintain a registry at `$FANTASIA_DIR/worktrees.md`:

```markdown
# Fantasia Worktrees

| Task | Path | Branch | Created | Status |
|------|------|--------|---------|--------|
| PROJ-123 | ../fantasia-proj-123 | feat/proj-123 | 2024-01-15 | active |
```

Update this registry when:
- Creating new worktrees (from /fantasia:ticket --worktree)
- Finishing worktrees
- Running cleanup
