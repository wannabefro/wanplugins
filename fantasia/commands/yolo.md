---
description: Run the full Fantasia workflow without confirmations - map, plan, build, and review in one shot
argument-hint: "<task-or-ticket> [--jira|--linear] [--worktree]"
allowed-tools: ["Read", "Bash", "Glob", "Grep", "Write", "Edit", "Task", "WebFetch", "mcp__*"]
---

You are running Fantasia in YOLO mode - full automated workflow without pausing for confirmations. This is for when you trust the process and want to go from task description to reviewed code as fast as possible.

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

## Parse Arguments

```bash
ARGS="$ARGUMENTS"
WORKTREE_MODE=false
SOURCE_OVERRIDE=""

if echo "$ARGS" | grep -q "\-\-worktree"; then
  WORKTREE_MODE=true
  ARGS=$(echo "$ARGS" | sed 's/--worktree//')
fi

if echo "$ARGS" | grep -q "\-\-jira"; then
  SOURCE_OVERRIDE="jira"
  ARGS=$(echo "$ARGS" | sed 's/--jira//')
fi

if echo "$ARGS" | grep -q "\-\-linear"; then
  SOURCE_OVERRIDE="linear"
  ARGS=$(echo "$ARGS" | sed 's/--linear//')
fi

TASK_INPUT=$(echo "$ARGS" | xargs)
```

## Argument Required

If `$ARGUMENTS` is empty (after removing flags), ask:
"What would you like to build? Provide either:
- A task description: `add user authentication`
- A ticket ID/URL: `PROJ-123` or `https://linear.app/...`

Options:
- `--jira` - Explicitly use Jira for ticket ID
- `--linear` - Explicitly use Linear for ticket ID
- `--worktree` - Create isolated git worktree"

## Detect Input Type

Determine if input is a ticket or task description:

**If source explicitly specified** (`--jira` or `--linear`):
- Treat input as ticket ID from that source

**Ticket patterns** (auto-detect):
- Contains `linear.app` or `atlassian.net` → ticket URL
- Matches `LETTERS-NUMBERS` format (e.g., `PROJ-123`) → ticket ID (will ask source if ambiguous)

**Otherwise**: Treat as task description

## YOLO Startup

```
🚀 YOLO MODE
============

Task: <task or ticket>
Worktree: <yes/no>

This will run the full workflow without stopping:
  Map → Plan → Build → Review

Starting in 3... 2... 1...
```

## Phase 1: Setup

### If Ticket
Fetch ticket details (same as /fantasia:ticket):
- Detect source (Linear/Jira)
- Fetch and parse ticket
- Extract requirements and acceptance criteria
- Save to TICKET.md

### If Task Description
Create task slug and directory:
```bash
TASK_SLUG=$(echo "$TASK_INPUT" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/-/g' | sed 's/--*/-/g' | sed 's/^-//' | sed 's/-$//' | cut -c1-50)
mkdir -p "$FANTASIA_DIR/plans/$TASK_SLUG"
```

### If Worktree Mode
Create worktree before proceeding:
```bash
MAIN_REPO=$(pwd)
WORKTREE_PATH="../fantasia-$TASK_SLUG"
BRANCH_NAME="feat/$TASK_SLUG"

git worktree add "$WORKTREE_PATH" -b "$BRANCH_NAME"

# Enable direnv in the new worktree if .envrc exists
if [ -f "$WORKTREE_PATH/.envrc" ]; then
  echo "Enabling direnv in worktree..."
  cd "$WORKTREE_PATH" && direnv allow
  cd "$MAIN_REPO"
  echo "✓ direnv enabled for worktree"
fi

# Worktree uses same FANTASIA_DIR (home directory)
mkdir -p "$FANTASIA_DIR/plans/$TASK_SLUG"

# Check if codebase maps exist
[ -d "$FANTASIA_DIR/codebase" ] && echo "✓ Codebase maps available"

# Register worktree
mkdir -p "$FANTASIA_DIR"

# Initialize registry with header if it doesn't exist
if [ ! -f "$FANTASIA_DIR/worktrees.md" ]; then
  cat > "$FANTASIA_DIR/worktrees.md" << 'HEADER'
# Fantasia Worktrees

| Task | Path | Branch | Created | Status |
|------|------|--------|---------|--------|
HEADER
fi

# Append worktree entry
echo "| $TASK_SLUG | $WORKTREE_PATH | $BRANCH_NAME | $(date -I) | active |" >> "$FANTASIA_DIR/worktrees.md"

cd "$WORKTREE_PATH"
```

Report:
```
✓ Phase 0: Setup complete
  - Task: <slug>
  - Worktree: <path or "no">
  - Plan directory: $FANTASIA_DIR/plans/<slug>/
```

## Phase 2: Map (if needed)

Check if codebase maps exist:
```bash
[ -d "$FANTASIA_DIR/codebase" ] && echo "maps_exist=true"
```

**If maps exist**: Skip to Phase 3
**If no maps**: Run mapping

### Auto-Map
Spawn all 4 mapper agents in parallel (same as /fantasia:map):
- tech-mapper
- arch-mapper
- quality-mapper
- concerns-mapper

Wait for all to complete.

Report:
```
✓ Phase 1: Mapping complete
  - Created: STACK.md, ARCHITECTURE.md, STRUCTURE.md, CONVENTIONS.md, TESTING.md, CONCERNS.md
```

## Phase 3: Plan

Run planning workflow without confirmations:

### Discovery (auto)
- Read relevant maps
- Explore task-relevant code
- DO NOT ask clarifying questions (make reasonable assumptions)

### Design (auto)
- Create work breakdown
- Assign to specialists
- Write PLAN.md

Report:
```
✓ Phase 2: Planning complete
  - Plan: $FANTASIA_DIR/plans/<slug>/PLAN.md
  - Phases: <count>
  - Files to create: <count>
  - Files to modify: <count>
```

## Phase 4: Build

Execute plan with parallel agents:

### Sequential execution
Follow the plan's execution order:
1. Architect phase (if needed)
2. Implementer phase
3. Integrator phase (if needed)

Each agent:
- Receives relevant context
- Writes code directly to files
- Returns brief confirmation

Report after each:
```
✓ Phase 3.1: Architect complete - created <files>
✓ Phase 3.2: Implementer complete - created/modified <files>
✓ Phase 3.3: Integrator complete - wired up <components>
```

## Phase 5: Review

Spawn all 3 review agents in parallel:
- code-reviewer
- bug-hunter
- test-reviewer

Compile REVIEW.md with all findings.

Report:
```
✓ Phase 4: Review complete
  - Code Quality: X issues
  - Potential Bugs: Y issues
  - Test Coverage: <assessment>
```

## YOLO Complete

```
🎉 YOLO COMPLETE
================

Task: <name>
Duration: <time>

## Files Changed
- Created: <list>
- Modified: <list>

## Review Summary
| Category | Issues | Severity |
|----------|--------|----------|
| Code Quality | X | Y high |
| Bugs | X | Y critical |
| Tests | <assessment> | - |

## Acceptance Criteria (if ticket)
- ✅ <criterion 1>
- ✅ <criterion 2>
- ❌ <criterion 3>

## Verdict: <Ready/Minor Issues/Needs Fixes>

## Next Steps
<If ready>
Commit your changes:
  git add -A && git commit -m "feat: <task summary>"

<If worktree>
When ready to merge:
  /fantasia:worktree finish
</If>
</If>

<If issues>
Review detailed findings:
  $FANTASIA_DIR/plans/<slug>/REVIEW.md

Fix issues and run:
  /fantasia:review
</If>
```

## Save Final Checkpoint

```bash
cat > $FANTASIA_DIR/fantasia-state.md << EOF
# Fantasia Checkpoint

**Saved**: $(date -Iseconds)
**Phase**: idle
**Last Action**: yolo complete
**Plan**: $TASK_SLUG
**Mode**: yolo
$([ "$WORKTREE_MODE" = "true" ] && echo "**Worktree**: $WORKTREE_PATH")

## Context
- Maps: available in $FANTASIA_DIR/codebase/
- Plan: $FANTASIA_DIR/plans/$TASK_SLUG/PLAN.md
- Build: complete
- Review: complete ($FANTASIA_DIR/plans/$TASK_SLUG/REVIEW.md)

## YOLO Summary
<verdict and key stats>

## Next Step
Address any review findings, then commit.

## Workflow Complete
YOLO run finished. Review the changes and commit when ready.
EOF
```

## Error Handling

If any phase fails:

```
⚠️ YOLO HALTED
==============

Failed at: <phase>
Error: <description>

## What Completed
- ✓ Setup
- ✓ Map (if run)
- ✓ Plan
- ✗ Build - <error>
- ○ Review (skipped)

## Recovery Options
1. Fix the error and resume:
   /fantasia:resume

2. Retry just the failed phase:
   /fantasia:build

3. Abandon (if worktree):
   cd .. && git worktree remove <path>

State saved to $FANTASIA_DIR/fantasia-state.md
```

Save checkpoint with:
- Which phases completed
- Where it failed
- Error details
- Resume instructions
