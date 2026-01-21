---
description: Address feedback from a GitHub PR (review comments, discussions, CI failures)
argument-hint: "<github-pr-url>"
allowed-tools: ["Read", "Bash", "Glob", "Grep", "Write", "Edit", "Task", "AskUserQuestion"]
---

# Fantasia PR Feedback Handler

Address feedback directly from a GitHub PR — code review comments, discussions, and CI failures.

## Determine Context

```bash
if [ -z "$FANTASIA_DIR" ]; then
  REPO_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
  PROJECT_SLUG=$(basename "$REPO_ROOT" | tr '[:upper:]' '[:lower:]' | tr ' ' '-')
  FANTASIA_DIR="$HOME/.claude/fantasia/$PROJECT_SLUG"
fi

CURRENT_BRANCH=$(git branch --show-current)
echo "FANTASIA_DIR=$FANTASIA_DIR"
echo "BRANCH=$CURRENT_BRANCH"
```

## Parse Arguments

**PR URL**: `$ARGUMENTS` (e.g., `https://github.com/owner/repo/pull/123`)

Extract from URL:
```
OWNER = <from URL path>
REPO = <from URL path>
PR_NUMBER = <from URL path>
```

Validate:
```bash
# Check gh CLI is available
if ! command -v gh &> /dev/null; then
  echo "ERROR: GitHub CLI (gh) required. Install: https://cli.github.com/"
  exit 1
fi

# Verify PR exists and user has access
gh pr view $PR_NUMBER --json number --jq '.number' 2>/dev/null || {
  echo "ERROR: Cannot access PR #$PR_NUMBER. Check URL and permissions."
  exit 1
}
```

## Phase 1: Fetch All Feedback

Create output directory:
```bash
mkdir -p "$FANTASIA_DIR/pr-feedback/PR-$PR_NUMBER"
```

### Fetch PR Metadata

```bash
gh pr view $PR_NUMBER --json title,body,state,author,baseRefName,headRefName,url \
  > "$FANTASIA_DIR/pr-feedback/PR-$PR_NUMBER/metadata.json"
```

Save to `CONTEXT.md`:
```markdown
# PR #<number>: <title>

**Author**: @<author>
**Branch**: <head> → <base>
**State**: <open|merged|closed>
**URL**: <url>

## Description
<body>
```

### Fetch Code Review Comments

```bash
gh api repos/$OWNER/$REPO/pulls/$PR_NUMBER/comments \
  --jq '.[] | {id, path, line, body, user: .user.login, created_at, in_reply_to_id}' \
  > "$FANTASIA_DIR/pr-feedback/PR-$PR_NUMBER/review-comments.json"
```

### Fetch PR Discussion (Issue Comments)

```bash
gh api repos/$OWNER/$REPO/issues/$PR_NUMBER/comments \
  --jq '.[] | {id, body, user: .user.login, created_at}' \
  > "$FANTASIA_DIR/pr-feedback/PR-$PR_NUMBER/discussion.json"
```

### Fetch CI Status

```bash
gh pr checks $PR_NUMBER --json name,state,conclusion \
  > "$FANTASIA_DIR/pr-feedback/PR-$PR_NUMBER/ci-status.json"
```

For any failed checks, fetch logs:
```bash
# Get failed run IDs
FAILED_RUNS=$(gh pr checks $PR_NUMBER --json name,conclusion,link \
  --jq '.[] | select(.conclusion == "failure") | .link' | head -3)

for RUN_URL in $FAILED_RUNS; do
  RUN_ID=$(echo $RUN_URL | grep -oE '[0-9]+$')
  gh run view $RUN_ID --log-failed 2>/dev/null | head -200 \
    >> "$FANTASIA_DIR/pr-feedback/PR-$PR_NUMBER/ci-failures.log"
done
```

### Skip Resolved/Outdated Comments

Filter out:
- Comments marked as "resolved" in review threads
- Comments on lines that no longer exist (outdated)

```bash
# Get review threads to check resolved status
gh api repos/$OWNER/$REPO/pulls/$PR_NUMBER/reviews \
  --jq '.[] | {id, state, user: .user.login}' \
  > "$FANTASIA_DIR/pr-feedback/PR-$PR_NUMBER/reviews.json"
```

## Phase 2: Triage Feedback

### Load Codebase Maps (if available)

```bash
if [ -f "$FANTASIA_DIR/codebase/CONVENTIONS.md" ]; then
  echo "MAPS_AVAILABLE=true"
fi
```

If maps exist, read only the relevant sections:
- `CONVENTIONS.md` - For pattern matching
- Don't load full architecture maps unless needed (token efficiency)

### Spawn Triager Agent

Use the Task tool to spawn **pr-feedback-triager** (haiku):

```
Triage PR feedback for PR #$PR_NUMBER:

Review comments:
<parsed from review-comments.json>

Discussion items:
<parsed from discussion.json>

CI status:
<parsed from ci-status.json>

Assign priority (HIGH/MEDIUM/LOW) to each item.
Write results to: $FANTASIA_DIR/pr-feedback/PR-$PR_NUMBER/TRIAGE.md
```

### Build Interactive Summary

After triage completes, read `TRIAGE.md` and display:

```
PR #123: "Add user authentication flow"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Found <N> feedback items:

 HIGH  [1] @reviewer1 on auth.ts:45 — "Add null check before accessing user.id"
 HIGH  [2] CI: jest tests failing — 2 tests in auth.test.ts
 HIGH  [3] @reviewer2 on auth.ts:89 — "This bypasses rate limiting"
 MED   [4] @reviewer1 on utils.ts:12 — "Extract this to a helper function"
 MED   [5] Discussion — "Should we add logging for failed attempts?"
 LOW   [6] @reviewer2 on auth.ts:3 — "Typo: 'authenication'"
 LOW   [7] Discussion — "Nice work on the error messages!" (no action)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Address: [a]ll, [h]igh only, [#,#,#] specific items, [n]one to review first?
```

### Get User Selection

Use AskUserQuestion:
```
Which feedback items would you like to address?

Options:
- All items (7 total)
- High priority only (3 items)
- Enter specific numbers (e.g., 1,2,3)
- None - just save feedback for manual review
```

Parse response:
- `a` or `all` → address all items
- `h` or `high` → address only HIGH priority
- `1,2,3` → address specific items by number
- `n` or `none` → skip to save and exit

## Phase 3: Address Feedback

### Prepare Feedback Batch

Group selected items by file for token efficiency:
```
auth.ts:
  - [1] Line 45: null check
  - [3] Line 89: rate limiting

utils.ts:
  - [4] Line 12: extract helper

CI:
  - [2] jest failures
```

### Load Relevant Context Only

For each file being modified:
1. Read only the file (not entire codebase)
2. Load only relevant section of CONVENTIONS.md
3. For CI failures, load only the error output (truncated to relevant lines)

### Spawn Handler Agent(s)

**For simple items** (typos, null checks, style fixes):
Use Task tool to spawn **pr-feedback-handler** (sonnet) with batched items:

```
Address these PR feedback items:

ITEMS:
[1] HIGH - auth.ts:45 - @reviewer1 - "Add null check before accessing user.id"
    Context: <±30 lines around line 45>

[6] LOW - auth.ts:3 - @reviewer2 - "Typo: 'authenication'"
    Context: <the line with the typo>

CONVENTIONS:
<relevant section from CONVENTIONS.md if available>

Make the changes, stage them with git add, and document in CHANGES.md.
Output file: $FANTASIA_DIR/pr-feedback/PR-$PR_NUMBER/CHANGES.md
```

**Check for escalation triggers in handler response:**

If handler returns `ESCALATE: bug-hunter`:
```
Spawning bug-hunter for CI failure investigation...
```
→ Use Task tool to spawn bug-hunter with CI failure context

If handler returns `ESCALATE: code-reviewer`:
```
Spawning code-reviewer for security concern...
```
→ Use Task tool to spawn code-reviewer with the security-related feedback

If handler returns `ESCALATE: approach-explorer`:
```
Spawning approach-explorer for architectural decision...
```
→ Use Task tool to spawn approach-explorer with the feedback requiring design choices

### Process Escalations

For each escalation:
1. Spawn the specialist agent
2. Wait for their analysis
3. Feed results back to pr-feedback-handler
4. Handler makes final changes based on specialist input

## Phase 4: Finalize

### Read Changes Made

```bash
cat "$FANTASIA_DIR/pr-feedback/PR-$PR_NUMBER/CHANGES.md"
```

### Stage Changes

```bash
# Stage all modified files (handler should have done this, but verify)
git add -A
```

### Display Summary

```
Addressing <N> feedback items...

 ✓ [1] auth.ts:45 — Added null check for user.id
 ✓ [2] CI — Fixed failing tests (missing mock for authService)
 ✓ [3] auth.ts:89 — Added rate limit check before token generation
   ↳ Escalated to code-reviewer: security concern
 ✓ [4] utils.ts:12 — Extracted to parseAuthHeader() helper
 ✓ [6] auth.ts:3 — Fixed typo

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Changes staged. Review diff:

  git diff --cached

Commit when ready:

  git commit -m "Address PR #123 feedback"

Details: $FANTASIA_DIR/pr-feedback/PR-$PR_NUMBER/CHANGES.md
```

### Save State

Update `$FANTASIA_DIR/fantasia-state.md`:
```markdown
**Phase**: idle
**Last Action**: pr feedback addressed
**PR**: #$PR_NUMBER
**Items Addressed**: <count>
```

## Token Efficiency

This command is designed to minimize token usage:

1. **Haiku for triage** - Classification doesn't need sonnet
2. **Lazy map loading** - Only load CONVENTIONS.md, skip architecture maps
3. **File batching** - Group feedback by file, one context load per file
4. **Truncated CI logs** - Only fetch `--log-failed`, limit to 200 lines
5. **Skip resolved** - Don't process already-resolved comments
6. **Diff-scoped context** - Pass ±30 lines around feedback, not full files

## Error Handling

**If gh CLI not installed:**
```
GitHub CLI (gh) is required for this command.
Install: https://cli.github.com/
Then authenticate: gh auth login
```

**If PR not accessible:**
```
Cannot access PR #123. Possible causes:
- URL is incorrect
- Repository is private and you're not authenticated
- PR has been deleted

Try: gh pr view 123 --repo owner/repo
```

**If no feedback found:**
```
No actionable feedback found on PR #123.

- Review comments: 0
- Discussion items: 0 (or all resolved)
- CI checks: all passing ✓

Nothing to address!
```
