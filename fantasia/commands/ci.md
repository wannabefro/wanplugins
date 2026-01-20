---
description: Check CI/CD pipeline status for current branch
argument-hint: "[--wait] [--fix-on-fail]"
allowed-tools: ["Read", "Bash", "Write", "Task"]
---

Check CI status for the current branch and optionally trigger fixes on failure.

## Determine Context

```bash
if [ -z "$FANTASIA_DIR" ]; then
  REPO_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
  PROJECT_SLUG=$(basename "$REPO_ROOT" | tr '[:upper:]' '[:lower:]' | tr ' ' '-')
  FANTASIA_DIR="$HOME/.claude/fantasia/$PROJECT_SLUG"
fi

CURRENT_BRANCH=$(git branch --show-current)
REPO_URL=$(git remote get-url origin 2>/dev/null | sed 's/\.git$//' | sed 's/git@github.com:/https:\/\/github.com\//')
echo "BRANCH=$CURRENT_BRANCH"
echo "REPO=$REPO_URL"
```

## Parse Arguments

- `--wait` - Poll until CI completes (up to 30 min)
- `--fix-on-fail` - Auto-trigger `/fantasia:fix` if CI fails

## Check CI Status

Use `gh` CLI if available:

```bash
# Check if gh is available
if command -v gh &> /dev/null; then
  gh run list --branch "$CURRENT_BRANCH" --limit 1 --json status,conclusion,name,url
fi
```

If `gh` not available, provide manual check instructions:
```
CI status check requires GitHub CLI (gh).
Install: https://cli.github.com/

Or check manually:
$REPO_URL/actions?query=branch:$CURRENT_BRANCH
```

## Wait Mode

If `--wait` flag:

```bash
MAX_ATTEMPTS=60  # 30 minutes at 30s intervals
ATTEMPT=0

while [ $ATTEMPT -lt $MAX_ATTEMPTS ]; do
  STATUS=$(gh run list --branch "$CURRENT_BRANCH" --limit 1 --json status --jq '.[0].status')

  if [ "$STATUS" = "completed" ]; then
    break
  fi

  echo "CI status: $STATUS (attempt $ATTEMPT/$MAX_ATTEMPTS)"
  sleep 30
  ATTEMPT=$((ATTEMPT + 1))
done
```

## Report Status

```
🔄 CI Status: <branch>

Status: <pending|running|completed>
Conclusion: <success|failure|cancelled>
Workflow: <name>
URL: <link>

<If completed and success>
✅ CI passed! Ready to merge.
</If>

<If completed and failure>
❌ CI failed.

Failed steps:
- <step 1>
- <step 2>

<If --fix-on-fail>
Auto-triggering /fantasia:fix with CI failure context...
</If>
</If>
```

## Auto-Fix on Failure

If `--fix-on-fail` and CI failed:

1. Fetch failure logs:
```bash
gh run view --log-failed --json jobs
```

2. Create CI failure context file:
```bash
cat > "$FANTASIA_DIR/plans/ci-fix-$(date +%s)/CI-FAILURE.md" << EOF
# CI Failure Analysis

**Branch**: $CURRENT_BRANCH
**Workflow**: <name>
**Failed at**: <timestamp>

## Failed Steps
<from logs>

## Error Output
<relevant log sections>
EOF
```

3. Trigger fix:
```
/fantasia:fix --from-ci
```

## Save State

```bash
cat >> "$FANTASIA_DIR/fantasia-state.md" << EOF

## CI Status ($(date -Iseconds))
- Branch: $CURRENT_BRANCH
- Status: <status>
- Conclusion: <conclusion>
EOF
```
