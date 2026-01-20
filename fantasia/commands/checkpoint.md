---
description: Save current Fantasia workflow state as a checkpoint for later resumption
allowed-tools: ["Read", "Bash", "Write"]
---

Save the current Fantasia workflow state to a checkpoint file.

**TOKEN EFFICIENCY**: Checkpoint files should be minimal - just enough metadata to resume. All detailed content lives in dedicated files (PLAN.md, INVESTIGATION.md, REVIEW.md, etc.). Checkpoints POINT TO files, they don't DUPLICATE content.

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

## Gather Current State

1. Check what maps exist:
```bash
ls $FANTASIA_DIR/codebase/ 2>/dev/null | tr '\n' ', '
```

2. Find current plan:
```bash
ls -t $FANTASIA_DIR/plans/ 2>/dev/null | head -1
```

3. Check plan status:
```bash
PLAN=$(ls -t $FANTASIA_DIR/plans/ 2>/dev/null | head -1)
[ -f "$FANTASIA_DIR/plans/$PLAN/PLAN.md" ] && echo "has_plan=true"
[ -f "$FANTASIA_DIR/plans/$PLAN/REVIEW.md" ] && echo "has_review=true"
```

## Determine Current Phase

Based on artifacts and what we're currently doing:

- `mapping` - Currently running /fantasia:map
- `planning` - Currently running /fantasia:plan
- `building` - Currently running /fantasia:build
- `reviewing` - Currently running /fantasia:review
- `idle` - Between phases, waiting for user

## Determine Current Mode

Check if we're in YOLO mode by reading existing state:
```bash
CURRENT_MODE=$(sed -n -E 's/^[[:space:]]*([*][*])?Mode([*][*])?:[[:space:]]*//p' $FANTASIA_DIR/fantasia-state.md 2>/dev/null | head -1)
if [ -z "$CURRENT_MODE" ]; then
  CURRENT_MODE="interactive"
fi
```

## Write State File

Write to `$FANTASIA_DIR/fantasia-state.md`:

```markdown
# Fantasia Checkpoint

**Saved**: <timestamp>
**Phase**: <current phase>
**Plan**: <plan name or "none">
**Mode**: <yolo | interactive>

## Context
- Maps: <list of map files>
- Plan status: <not started | in progress | complete>
- Build status: <not started | in progress | complete>
- Review status: <not started | in progress | complete>

## Notes
<Only essential context not captured elsewhere - max 100 words>
<Reference files, don't duplicate content - e.g., "See INVESTIGATION.md for details">

## To Resume
Run `/fantasia:resume` to continue from this checkpoint.
```

## Confirm Checkpoint

```
✓ Checkpoint saved to $FANTASIA_DIR/fantasia-state.md

Current state:
- Phase: <phase>
- Plan: <plan name>
- Progress: <summary>

To resume later: /fantasia:resume
```

## Auto-Checkpoint Integration

Note: The main commands (map, plan, build, review) should call checkpoint automatically at key points:
- After map completes
- After plan is approved
- After each build phase completes
- After review completes

This command is for manual checkpoints mid-workflow.
