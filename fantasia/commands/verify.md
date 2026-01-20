---
description: Record manual testing results for the current task
argument-hint: "[pass|fail] [notes]"
allowed-tools: ["Read", "Bash", "Write"]
---

Record manual verification results for the current Fantasia task.

## Determine Context

```bash
if [ -z "$FANTASIA_DIR" ]; then
  REPO_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
  PROJECT_SLUG=$(basename "$REPO_ROOT" | tr '[:upper:]' '[:lower:]' | tr ' ' '-')
  FANTASIA_DIR="$HOME/.claude/fantasia/$PROJECT_SLUG"
fi

# Get current task
TASK_SLUG=$(grep -E "^\*\*Plan\*\*:" "$FANTASIA_DIR/fantasia-state.md" 2>/dev/null | sed 's/.*: //')
echo "FANTASIA_DIR=$FANTASIA_DIR"
echo "TASK=$TASK_SLUG"
```

## Parse Arguments

`$ARGUMENTS` can be:
- `pass` or `fail` - Result of manual testing
- Additional text - Notes about what was tested

If no arguments, ask what was tested and the result.

## Record Verification

Append to `$FANTASIA_DIR/plans/<task>/MANUAL-VERIFICATION.md`:

```markdown
## Manual Verification - <timestamp>

**Result**: PASS / FAIL
**Tested by**: Manual verification
**Notes**: <from arguments or user input>

### What Was Tested
- <description>

### Evidence
<any relevant observations>
```

## Update Review Status

If this task has a REVIEW.md, append verification status:

```markdown
## Manual Verification Status
- Result: <pass/fail>
- Timestamp: <when>
- Notes: <summary>
```

## Completion

```
✓ Manual verification recorded: <PASS/FAIL>

Task: <task-slug>
Notes: <summary>

Saved to: $FANTASIA_DIR/plans/<task>/MANUAL-VERIFICATION.md
```
