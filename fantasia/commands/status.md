---
description: Show current Fantasia workflow state - what's been mapped, planned, built, and what's next
allowed-tools: ["Read", "Bash", "Glob"]
---

Check the current Fantasia workflow state for this project.

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

## Gather Information

1. Read state file: `cat $FANTASIA_DIR/fantasia-state.md 2>/dev/null`
2. Check maps: `ls $FANTASIA_DIR/codebase/ 2>/dev/null`
3. Check plans: `ls -la $FANTASIA_DIR/plans/ 2>/dev/null`

## Present Status Report

```
🎬 Fantasia Status
==================

## Codebase Maps: <Mapped | Not mapped>
<List map files if they exist>

## Active Plan: <Plan name | None>
<Plan summary if exists>

## Workflow Position: <Position>
## Next Step: <Recommended command>
```

### Position Logic

| Maps | Plan | Review | Position | Next |
|------|------|--------|----------|------|
| No | - | - | Start | `/fantasia:map` |
| Yes | No | - | Mapped | `/fantasia:plan <task>` |
| Yes | Yes | No | Planned | `/fantasia:build` |
| Yes | Yes | Yes | Reviewed | Commit or fix issues |
