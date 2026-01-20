---
description: Resume Fantasia workflow from last checkpoint - restores context and continues where you left off
argument-hint: "[--yolo]"
allowed-tools: ["Read", "Bash", "Glob", "Grep", "Write", "Edit", "Task", "AskUserQuestion"]
---

Resume the Fantasia workflow from where it was left off.

## Determine Fantasia Directory

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
YOLO_MODE=false

if echo "$ARGUMENTS" | grep -q "\-\-yolo"; then
  YOLO_MODE=true
fi
```

The `--yolo` flag can be used to:
- Force YOLO mode continuation (skip confirmations)
- Resume a previously interrupted YOLO run

## 1. Load State

Read the state file and maps to restore context:

```bash
# Check state
cat $FANTASIA_DIR/fantasia-state.md 2>/dev/null

# Check what maps exist
ls $FANTASIA_DIR/codebase/ 2>/dev/null

# Find most recent plan
ls -t $FANTASIA_DIR/plans/ 2>/dev/null | head -1
```

## 2. Restore Context

Based on what exists, read the relevant files:

### If maps exist:
Read key maps to restore pattern knowledge:
- `$FANTASIA_DIR/codebase/CONVENTIONS.md` (always)
- `$FANTASIA_DIR/codebase/ARCHITECTURE.md` (if planning/building)
- `$FANTASIA_DIR/codebase/STACK.md` (if building)

### If a plan exists:
Read the plan file:
- `$FANTASIA_DIR/plans/<task>/PLAN.md`
- `$FANTASIA_DIR/plans/<task>/EXTERNAL-DEPS.md` (if exists)
- `$FANTASIA_DIR/plans/<task>/REVIEW.md` (if exists)

## 3. Determine Position and Mode

### Check for YOLO Mode
Look for a saved mode in the state file or use the `--yolo` flag:
```bash
CURRENT_MODE=$(sed -n -E 's/^[[:space:]]*([*][*])?Mode([*][*])?:[[:space:]]*//p' $FANTASIA_DIR/fantasia-state.md 2>/dev/null | head -1)
if [ -z "$CURRENT_MODE" ]; then
  CURRENT_MODE="interactive"
fi

if [ "$CURRENT_MODE" = "yolo" ] || [ "$YOLO_MODE" = "true" ]; then
  YOLO_MODE=true
fi
```

### Position Logic

| State | Action |
|-------|--------|
| `phase: mapping` | Mapping was interrupted - offer to restart or continue |
| `phase: planning` | Planning was interrupted - continue discovery/design |
| `phase: building` | Building was interrupted - check progress, continue |
| `phase: reviewing` | Review was interrupted - continue review |
| `phase: fixing` | Fix workflow interrupted - check attempt and feedback state |
| `phase: fixing` + `mode: feedback-loop` | Mid-feedback-loop - continue with decision |
| No state but maps exist | Ready for planning |
| No state but plan exists | Ready for build or review |

### Special Handling: Fix Workflow with Feedback

If state shows `phase: fixing`, check for feedback loop state:

```bash
# Check for feedback loop mode and attempt number
FEEDBACK_MODE=false
if [ "$CURRENT_MODE" = "feedback-loop" ]; then
  FEEDBACK_MODE=true
fi
ATTEMPT=$(grep -i "Attempt:" $FANTASIA_DIR/fantasia-state.md 2>/dev/null | grep -o '[0-9]*')
FEEDBACK=$(grep -A1 "Feedback:" $FANTASIA_DIR/fantasia-state.md 2>/dev/null | tail -1)
DECISION=$(grep -i "Decision:" $FANTASIA_DIR/fantasia-state.md 2>/dev/null)
```

If in feedback loop mode:

```
🔄 Resuming Fix (Attempt $ATTEMPT)
===================================

**Fix**: <fix-slug>
**Previous Feedback**: <feedback from state>
**Decision**: <reinvestigate | iterate>

You were in the middle of addressing feedback on a fix that didn't work.

Ready to continue with <re-investigation | revised fix approach>?
```

If fixing but not in feedback loop:
- Load INVESTIGATION.md and PLAN.md
- Check what phase of fix workflow was active
- Resume from that phase

## 4. Present Resume Summary

```
🎬 Resuming Fantasia
====================

## Restored Context
- Codebase: <repo type if detected>
- Maps: <list of available maps>
- Plan: <current plan name>
- Phase: <where we left off>
- Mode: <yolo | interactive | feedback-loop>

<If feedback-loop mode>
## Feedback Loop State
- Attempt: <N>
- Last Feedback: <observation>
- Decision: <reinvestigate | iterate>
</If>

## Last Activity
<From state file if available>

## Ready to Continue
<What we'll do next>

<If YOLO mode>
🚀 YOLO mode active - continuing without confirmations...
</If>

<If feedback-loop mode>
🔄 Feedback loop active - continuing fix iteration...
</If>

<If interactive mode>
Proceed? (y/n)
</If>
```

## 5. Continue Workflow

Based on the restored state, continue the appropriate phase:
- If mapping: Spawn remaining mapper agents
- If planning: Continue discovery or design phase
- If building: Check what's done, continue with remaining agents
- If reviewing: Continue with remaining reviewers

### YOLO Mode Continuation

If `YOLO_MODE=true`, skip all confirmations and continue through phases automatically:

1. **Don't ask "Proceed?"** - just continue
2. **Auto-approve plans** - skip readiness check confirmation
3. **Auto-continue builds** - don't pause between phases
4. **Complete the full workflow** - map → plan → build → review

After each phase completes in YOLO mode, immediately continue to the next:

| Current Phase | Next Action (YOLO) |
|---------------|-------------------|
| mapping | Auto-start planning |
| planning | Auto-start building |
| building | Auto-start reviewing |
| reviewing | Present final summary |

### Interactive Mode Continuation

If not in YOLO mode, ask for confirmation at each phase transition.

## State Not Found

If no state file exists but artifacts exist (maps, plans), reconstruct state:

```
No checkpoint found, but I found:
- Codebase maps in $FANTASIA_DIR/codebase/
- Plan: <name> in $FANTASIA_DIR/plans/

Would you like to:
1. Continue from where the artifacts suggest (build/review)
2. Start fresh with /fantasia:map
3. Create a new plan with /fantasia:plan
```
