---
description: Start a Fantasia workflow from a Linear or Jira ticket - fetches details and begins planning
argument-hint: "<ticket-url-or-id> [--jira|--linear] [--worktree] [--yolo]"
allowed-tools: ["Read", "Bash", "Glob", "Grep", "Write", "Task", "AskUserQuestion", "WebFetch", "mcp__*"]
---

You are starting a Fantasia workflow from a ticket. Your job is to fetch the ticket details and use them to kick off the planning process.

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

Parse `$ARGUMENTS` for:
- Ticket URL or ID (required)
- `--jira` flag (optional) - explicitly use Jira
- `--linear` flag (optional) - explicitly use Linear
- `--worktree` flag (optional) - create isolated git worktree
- `--yolo` flag (optional) - run full workflow without confirmations

```bash
ARGS="$ARGUMENTS"
WORKTREE_MODE=false
YOLO_MODE=false
SOURCE_OVERRIDE=""

if echo "$ARGS" | grep -q "\-\-worktree"; then
  WORKTREE_MODE=true
  ARGS=$(echo "$ARGS" | sed 's/--worktree//')
fi

if echo "$ARGS" | grep -q "\-\-yolo"; then
  YOLO_MODE=true
  ARGS=$(echo "$ARGS" | sed 's/--yolo//')
fi

if echo "$ARGS" | grep -q "\-\-jira"; then
  SOURCE_OVERRIDE="jira"
  ARGS=$(echo "$ARGS" | sed 's/--jira//')
fi

if echo "$ARGS" | grep -q "\-\-linear"; then
  SOURCE_OVERRIDE="linear"
  ARGS=$(echo "$ARGS" | sed 's/--linear//')
fi

TICKET_INPUT=$(echo "$ARGS" | xargs)  # trim whitespace
```

## Load Organization Context

Check for organization-wide context to get integration details:

```bash
ORG_CONTEXT_FILE="$HOME/.claude/fantasia/org-context.md"
if [ -f "$ORG_CONTEXT_FILE" ]; then
  echo "ORG_CONTEXT_EXISTS=true"
fi
```

If org context exists, extract `integrations` section:
- `jira.domain` - Use for Jira URL construction (e.g., `yourcompany.atlassian.net`)
- `jira.project_keys` - Known project keys to help with detection
- `linear.teams` - Known Linear team identifiers

This helps with:
1. **Better detection**: If ticket ID matches a known Jira project key, prefer Jira
2. **URL construction**: Use the configured Jira domain instead of asking user
3. **Validation**: Warn if ticket ID doesn't match known project keys

Example usage:
```
# If org context has jira.domain = "acme.atlassian.net"
# and ticket is "PROJ-123"
# and PROJ is in jira.project_keys
# → Automatically use Jira with full URL: https://acme.atlassian.net/browse/PROJ-123
```

## Argument Required

The user must provide a ticket URL or ID. If ticket input is empty, ask:
"Please provide a ticket URL or ID. Examples:
- Linear: `https://linear.app/team/issue/TEAM-123` or `TEAM-123`
- Jira: `https://yourcompany.atlassian.net/browse/PROJ-456` or `PROJ-456`

Options:
- `--jira` - Explicitly use Jira (for ambiguous IDs)
- `--linear` - Explicitly use Linear (for ambiguous IDs)
- `--worktree` - Create isolated git worktree for this ticket
- `--yolo` - Run full workflow (map→plan→build→review) without confirmations"

Ticket: `$ARGUMENTS`

## Detect Ticket Source

### If Source Explicitly Specified
If `SOURCE_OVERRIDE` is set, use that source directly:
- `--jira` → Use Jira
- `--linear` → Use Linear

### Otherwise, Auto-Detect

Parse the input to determine the source:

**Linear Detection**:
- URL contains `linear.app`
- ID format: `TEAM-123` (letters, hyphen, numbers) - but only if not overridden

**Jira Detection**:
- URL contains `atlassian.net` or `jira`
- ID format: `PROJ-123` (letters, hyphen, numbers) - but only if not overridden
- If org context has `jira.project_keys` and the ID prefix matches a known key → use Jira
- If org context has `jira.domain`, use it for URL construction

**If ambiguous** (ID format matches both and no flag provided):
- First check if ID prefix matches org context's `jira.project_keys` or `linear.teams`
- If still ambiguous, ask the user:
"Is this a Linear or Jira ticket? Use `--jira` or `--linear` to specify."

## Fetch Ticket Details

### For Linear Tickets

Use WebFetch to get ticket details from Linear's URL, or if an MCP server for Linear is available, use that.

Extract:
- Title
- Description
- Status
- Priority
- Labels/Tags
- Acceptance criteria (if in description)
- Related issues

### For Jira Tickets

Use the Atlassian MCP tools if available:
- Search for the ticket using `mcp__atlassian__*` tools
- Or use WebFetch with the Jira URL

Extract:
- Summary (title)
- Description
- Status
- Priority
- Labels
- Acceptance criteria
- Story points
- Epic link
- Subtasks

## Parse Ticket Content

Look for structured information in the description:

### Common Patterns
```
## Acceptance Criteria
- [ ] Criterion 1
- [ ] Criterion 2

## Technical Notes
...

## Design
...
```

### Extract Key Sections
- **What**: The main ask/feature
- **Why**: Business context/motivation
- **Acceptance Criteria**: Success conditions
- **Technical Notes**: Implementation hints
- **Out of Scope**: What NOT to build

## Present Ticket Summary

```
🎫 Ticket: <ID>
Source: Linear | Jira
Status: <status>
Priority: <priority>

## Summary
<ticket title>

## Description
<cleaned up description>

## Acceptance Criteria
<extracted criteria or "Not specified">

## Technical Context
<any technical notes from ticket>

---

Ready to plan this ticket? (y/n)
```

## Start Planning

If user confirms (or in yolo mode), transition to planning phase:

### Step 1: Create Worktree (if --worktree flag)

If `WORKTREE_MODE=true`:

```bash
TICKET_ID="<extracted-ticket-id>"
TASK_SLUG=$(echo "$TICKET_ID" | tr '[:upper:]' '[:lower:]')
MAIN_REPO=$(pwd)
WORKTREE_PATH="../fantasia-$TASK_SLUG"
BRANCH_NAME="feat/$TASK_SLUG"

# Create worktree with new branch
git worktree add "$WORKTREE_PATH" -b "$BRANCH_NAME"

# Worktree uses same FANTASIA_DIR (home directory, not repo-specific)
# So maps and plans are automatically shared across worktrees
echo "✓ Worktree created - shares Fantasia directory at $FANTASIA_DIR"

# Enable direnv in the new worktree if .envrc exists
if [ -f "$WORKTREE_PATH/.envrc" ]; then
  echo "Enabling direnv in worktree..."
  cd "$WORKTREE_PATH" && direnv allow
  cd "$MAIN_REPO"
  echo "✓ direnv enabled for worktree"
fi

# Create task-specific plan directory
mkdir -p "$FANTASIA_DIR/plans/$TASK_SLUG"

# Check if codebase maps exist
if [ -d "$FANTASIA_DIR/codebase" ]; then
  echo "✓ Codebase maps available in $FANTASIA_DIR/codebase/"
else
  echo "⚠ No codebase maps found - run /fantasia:map if needed"
fi

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

echo "WORKTREE_CREATED=$WORKTREE_PATH"
```

Present to user:
```
🌳 Created worktree for: <ticket-id>

Location: <worktree-path>
Branch: feat/<task-slug>
Fantasia Dir: $FANTASIA_DIR (shared)

To continue, open a terminal in the worktree:
  cd <worktree-path>

Or with Claude Code:
  claude --cd <worktree-path>
```

If NOT in worktree mode, continue in current directory:

### Step 2: Create task directory

```bash
TASK_SLUG=$(echo "$TICKET_ID" | tr '[:upper:]' '[:lower:]')
mkdir -p "$FANTASIA_DIR/plans/$TASK_SLUG"
```

### Step 3: Save ticket context to `$FANTASIA_DIR/plans/<task-slug>/TICKET.md`:
   ```markdown
   # Ticket: <ID>

   **Source**: Linear | Jira
   **URL**: <original URL>
   **Fetched**: <timestamp>

   ## Original Content
   <full ticket description>

   ## Extracted Requirements
   - <requirement 1>
   - <requirement 2>

   ## Acceptance Criteria
   - [ ] <criterion 1>
   - [ ] <criterion 2>
   ```

3. Continue with planning workflow (Phase 1: Discovery from plan.md)
   - Use ticket content as the task description
   - Reference TICKET.md for requirements
   - Acceptance criteria become plan success criteria

## Integration with Existing Workflow

The ticket becomes the source of truth (all in `$FANTASIA_DIR/plans/<task>/`):
- `TICKET.md` - Original requirements (don't modify)
- `PLAN.md` - How we'll implement it
- `REVIEW.md` - Verification against acceptance criteria

When review completes, check acceptance criteria:
```
## Acceptance Criteria Check
- [x] Criterion 1 - Implemented in <file>
- [x] Criterion 2 - Tested in <test file>
- [ ] Criterion 3 - Not yet complete
```

## Error Handling

### Ticket Not Found
```
Could not find ticket <ID>. Please check:
- The ticket ID is correct
- You have access to this ticket
- The ticket exists in Linear/Jira
```

### Access Denied
```
Unable to access ticket <ID>. This might be:
- A private ticket you don't have access to
- An authentication issue

Would you like to manually paste the ticket details instead?
```

### Manual Entry Fallback
If fetching fails, offer manual entry:
```
I couldn't fetch the ticket automatically. You can:
1. Paste the ticket details here
2. Provide a different ticket URL
3. Start planning manually with /fantasia:plan <description>
```

## Save Checkpoint

After ticket is fetched and planning begins:

```bash
TASK_SLUG="<ticket-id-lowercase>"
WORKTREE_INFO=""
if [ "$WORKTREE_MODE" = "true" ]; then
  WORKTREE_INFO="**Worktree**: $WORKTREE_PATH
**Branch**: $BRANCH_NAME"
fi

cat > $FANTASIA_DIR/fantasia-state.md << EOF
# Fantasia Checkpoint

**Saved**: $(date -Iseconds)
**Phase**: planning
**Last Action**: ticket fetched
**Plan**: $TASK_SLUG
**Ticket**: <original-url-or-id>
**Mode**: $([ "$YOLO_MODE" = "true" ] && echo "yolo" || echo "interactive")
$WORKTREE_INFO

## Context
- Maps: $([ -d "$FANTASIA_DIR/codebase" ] && echo "available" || echo "not mapped")
- Ticket: $FANTASIA_DIR/plans/$TASK_SLUG/TICKET.md
- Plan: in progress
- Build: not started
- Review: not started

## Ticket Summary
<brief ticket description>

## Next Step
Continue planning phase - discovery and design.

## To Resume
Run \`/fantasia:resume\` to restore context and continue.
EOF
```

## YOLO Mode Continuation

If `YOLO_MODE=true`, automatically continue through the entire workflow without pausing for confirmations:

### YOLO Workflow
```
🚀 YOLO MODE ACTIVATED
======================

Running full workflow for: <ticket-id>

1. ✓ Ticket fetched
2. → Mapping codebase (if not already mapped)
3. → Planning implementation
4. → Building with parallel agents
5. → Reviewing changes

Hold onto your hat...
```

### Execute YOLO Sequence

1. **Check/Run Map** (if `$FANTASIA_DIR/codebase/` doesn't exist):
   - Spawn mapper agents automatically
   - Wait for completion
   - Continue immediately

2. **Auto-Plan**:
   - Run discovery phase
   - Skip "verify understanding" confirmation
   - Generate PLAN.md automatically
   - Skip "ready to build?" confirmation

3. **Auto-Build**:
   - Execute all build phases
   - Skip per-phase confirmations
   - Continue on completion

4. **Auto-Review**:
   - Spawn all 3 review agents
   - Compile REVIEW.md
   - Present final summary

### YOLO Completion

```
🎉 YOLO COMPLETE: <ticket-id>
=============================

## Summary
- Ticket: <id> - <title>
- Time: <duration>
- Files created: X
- Files modified: Y

## Review Verdict
<from REVIEW.md>

## Acceptance Criteria
<status of each criterion>

## What's Next
- If clean: Ready to commit! Run: git add -A && git commit
- If issues: Review findings in $FANTASIA_DIR/plans/<task>/REVIEW.md

<If worktree mode>
When ready to merge:
  /fantasia:worktree finish
</If>
```

### YOLO Error Handling

If any phase fails in YOLO mode:
```
⚠️ YOLO paused at: <phase>

Error: <description>

Options:
1. Fix the issue and run /fantasia:resume --yolo to continue
2. Run the remaining phases manually
3. Abandon with /fantasia:worktree cleanup (if using worktree)
```

Save checkpoint with yolo state so resume can continue in yolo mode.
