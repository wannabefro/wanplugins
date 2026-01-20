---
description: Investigate and fix a bug, optionally from a Sentry issue
argument-hint: "<issue> [--sentry] [--yolo]"
allowed-tools: ["Read", "Bash", "Glob", "Grep", "Write", "Edit", "Task", "AskUserQuestion"]
---

# Fantasia Fix Mode

You are orchestrating a **bug fixing workflow** - systematic investigation and resolution.

## Determine Fantasia Directory

```bash
# Use environment variable if set, otherwise calculate default
if [ -z "$FANTASIA_DIR" ]; then
  REPO_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
  PROJECT_SLUG=$(basename "$REPO_ROOT" | tr '[:upper:]' '[:lower:]' | tr ' ' '-')
  FANTASIA_DIR="$HOME/.claude/fantasia/$PROJECT_SLUG"
fi

echo "FANTASIA_DIR=$FANTASIA_DIR"
mkdir -p "$FANTASIA_DIR/plans"
```

## Parse Arguments

**Issue**: `$ARGUMENTS.issue`
**Flags**: `$ARGUMENTS.flags`

Detect issue type:
```
IS_SENTRY = issue contains "sentry.io" OR "--sentry" in flags OR issue matches Sentry ID pattern
YOLO_MODE = "--yolo" in flags
```

## Phase 1: Issue Gathering

### If Sentry Issue

When the issue is from Sentry, use the Sentry MCP tools to gather context:

1. **Fetch issue details** using `/sentry:seer` or direct MCP calls:
   ```
   Get full details for Sentry issue: <issue-id-or-url>

   Need:
   - Error message and type
   - Stack trace
   - Affected users/frequency
   - Environment context
   - Tags and breadcrumbs
   - First/last seen dates
   ```

2. **Save Sentry context** to `$FANTASIA_DIR/plans/fix-<slug>/SENTRY.md`:
   ```markdown
   # Sentry Issue: <issue-id>

   **Error**: <error type and message>
   **First Seen**: <date>
   **Last Seen**: <date>
   **Events**: <count>
   **Users Affected**: <count>

   ## Stack Trace
   <full stack trace>

   ## Breadcrumbs
   <relevant breadcrumbs>

   ## Tags
   <environment, release, etc.>

   ## Additional Context
   <any other relevant info>
   ```

### If Manual Bug Report

If not a Sentry issue, gather information:

1. **What's the bug?** - Error message, unexpected behavior
2. **How to reproduce?** - Steps, conditions
3. **What's expected?** - Correct behavior
4. **Any context?** - When it started, related changes

Save to `$FANTASIA_DIR/plans/fix-<slug>/BUG-REPORT.md`.

## Phase 2: Investigation

### Read Codebase Maps (if available)

```bash
ls $FANTASIA_DIR/codebase/
```

If maps exist, read:
- `ARCHITECTURE.md` - Understand where the bug likely lives
- `CONCERNS.md` - Check if this is a known issue area

### Spawn Parallel Investigation Agents

**TOKEN EFFICIENCY**: Pass only relevant information to each investigator:
- Stack Tracer: Stack trace + error location + relevant file excerpts only
- Git Historian: File paths involved + date range only
- Similar Bug Finder: Bug pattern description + file paths only

Do NOT pass full SENTRY.md or entire codebase maps to investigators.

Use the Task tool to launch **3 investigators in parallel**:

```
Agent 1 - Stack Tracer:
Trace through the code for this bug:

**Issue**: [description or Sentry summary]
**Stack Trace**: [if available]
**Error Location**: [file:line if known]

Your job:
1. Follow the stack trace frame by frame
2. Understand what each function was trying to do
3. Find where the bug ORIGINATES (not just where it crashes)
4. Document the full execution path

Write your findings to: $FANTASIA_DIR/plans/fix-<slug>/STACK-TRACE.md

---

Agent 2 - Git Historian:
Investigate git history for this bug:

**Bug Location**: [file(s) involved]
**Suspected Area**: [from Sentry or description]

Your job:
1. Find when the buggy code was introduced or last changed
2. Check if this is a regression
3. Find related commits that might provide context
4. Identify who wrote the code and related changes

Write your findings to: $FANTASIA_DIR/plans/fix-<slug>/GIT-HISTORY.md

---

Agent 3 - Similar Bug Finder:
Search for similar bugs and patterns:

**Bug Pattern**: [description of what's wrong]
**Code Location**: [file(s) involved]

Your job:
1. Search for similar code patterns that might have the same bug
2. Find related error handling that might be inadequate
3. Look for previous fixes to similar issues
4. Determine if this is a systemic problem

Write your findings to: $FANTASIA_DIR/plans/fix-<slug>/SIMILAR-BUGS.md
```

**Launch all three in parallel** - they will write their findings to separate files.

### Spawn Skeptic Agent

After the 3 investigators complete, spawn the **bug-skeptic** agent to challenge their findings:

```
Agent 4 - Bug Skeptic:
Review and challenge the investigation findings:

**Investigation Files**:
- $FANTASIA_DIR/plans/fix-<slug>/STACK-TRACE.md
- $FANTASIA_DIR/plans/fix-<slug>/GIT-HISTORY.md
- $FANTASIA_DIR/plans/fix-<slug>/SIMILAR-BUGS.md

Your job:
1. Audit evidence quality - are claims backed by specific file:line citations?
2. Check for contradictions between investigators
3. Generate alternative explanations for the bug
4. Identify what verification would increase confidence

Write your findings to: $FANTASIA_DIR/plans/fix-<slug>/SKEPTIC-REVIEW.md
```

### Synthesize Investigation Results

After skeptic review completes:

1. **Read all investigation files**:
   - STACK-TRACE.md - Where the bug originates
   - GIT-HISTORY.md - When and why it was introduced
   - SIMILAR-BUGS.md - Related issues and patterns
   - SKEPTIC-REVIEW.md - Challenges and alternative explanations

2. **Check for contradictions** (from skeptic review):
   - Do investigators agree on bug location?
   - Do investigators agree on root cause?
   - Do investigators agree on when it was introduced?
   - Are there alternative explanations not ruled out?

3. **Assess confidence level**:
   - **High**: Investigators agree, evidence is specific (file:line), skeptic found no major issues
   - **Medium**: Some disagreement or gaps, but main theory is solid
   - **Low**: Major contradictions, weak evidence, or plausible alternatives

4. **Consolidate findings** into `$FANTASIA_DIR/plans/fix-<slug>/INVESTIGATION.md`:

```markdown
# Bug Investigation: <bug-slug>

## Summary
**Root Cause**: <synthesized from all investigators>
**Location**: <file:line>
**Severity**: Critical/High/Medium/Low
**Is Regression**: Yes/No
**Confidence**: High/Medium/Low

## Investigator Agreement

| Question | Stack Tracer | Git Historian | Similar Finder | Agree? |
|----------|--------------|---------------|----------------|--------|
| Bug location | <answer> | <answer> | <answer> | Yes/No |
| Root cause | <answer> | <answer> | <answer> | Yes/No |
| When introduced | <answer> | <answer> | <answer> | Yes/No |

### Contradictions (if any)
<List any disagreements between investigators and which evidence is stronger>

## Stack Trace Analysis
<Summary from stack-tracer>
- Origin point: <where bug starts>
- Crash point: <where error is thrown>
- Execution path: <how it gets there>
- **Evidence quality**: <Strong/Moderate/Weak - does it cite specific lines?>

## Git History Analysis
<Summary from git-historian>
- Introduced: <commit/date>
- Last changed: <commit/date>
- Regression evidence: <if applicable>
- **Evidence quality**: <Strong/Moderate/Weak - does it cite specific commits?>

## Similar Bugs Found
<Summary from similar-bug-finder>
- Similar patterns: <count and locations>
- Systemic issue: Yes/No
- Previous fixes: <relevant commits>
- **Evidence quality**: <Strong/Moderate/Weak - does it cite specific locations?>

## Skeptic Challenges

### Alternative Explanations Considered
1. <Alternative theory from skeptic>
   - Why it might be right: <evidence for>
   - Why it's probably wrong: <evidence against>

### Verification Recommendations
<What the skeptic says we should verify before proceeding>

## Consolidated Root Cause
<Detailed explanation combining all perspectives>
<Note any remaining uncertainty>

## Confidence Assessment
**Level**: High/Medium/Low
**Reasoning**: <Why this confidence level?>
**What would increase confidence**: <What verification would help?>

## Fix Recommendation
<Initial thoughts on how to fix>
<If confidence is Low: recommend further investigation first>
```

### Review Investigation

Read `INVESTIGATION.md` and verify:
- Does the root cause make sense?
- Do we understand WHY it's happening?
- Is evidence specific (file:line citations)?
- Do investigators agree on key points?
- Have alternative explanations been considered?

**If confidence is Low**: Ask clarifying questions or investigate further before planning fix.

**If confidence is Medium/High**: Proceed to fix planning, noting any remaining uncertainty.

## Phase 3: Fix Planning

### Create Fix Plan

Based on investigation, plan the fix:

```markdown
# Fix Plan: <bug-slug>

## Bug Summary
<One-line description>

## Root Cause
<From investigation>

## Fix Approach
<How we'll fix it>

## Files to Modify
- <file1>: <what changes>
- <file2>: <what changes>

## Test Strategy
- [ ] Add regression test for this bug
- [ ] Verify existing tests still pass
- [ ] Test edge cases: <list>

## Risks
<What could go wrong with this fix>

## Verification
<How we'll know it's fixed>
```

### Get Approval (unless YOLO)

If not YOLO_MODE:
- Present the fix plan
- Highlight any risks
- Ask: "Ready to implement fix? (yes/no/modify)"

## Phase 4: Implementation

### Write Regression Test First

Before fixing, add a test that:
1. Reproduces the bug (currently fails)
2. Will pass after the fix
3. Prevents regression

```
Create a test that:
- Reproduces: <the bug condition>
- Currently: Should FAIL (proving the bug exists)
- After fix: Should PASS
```

### Implement the Fix

Make minimal changes to fix the root cause:
- Follow patterns from CONVENTIONS.md (if maps exist)
- Keep the fix focused - don't refactor while fixing
- Add comments if the fix isn't obvious

### Run Tests

```bash
# Run the test suite
# Verify:
# 1. New regression test now passes
# 2. All existing tests still pass
```

## Phase 5: Verification

### Run Parallel Verification

Use the Task tool to spawn the **parallel-verifier** agent:

```
Run verification checks for this bug fix:

**Changed Files**: [list of modified files]
**Test Focus**: [regression test file/name]
**Codebase Type**: [detected from maps or repo structure]

Your job:
1. Run the test suite (focused on affected areas)
2. Run type checking (if applicable)
3. Run linting (if applicable)
4. Provide consolidated pass/fail assessment

Write results to: $FANTASIA_DIR/plans/fix-<slug>/VERIFICATION.md
```

The verifier will run tests, types, and lint in parallel and consolidate results.

### Check Verification Results

After verification completes, read `VERIFICATION.md`:

```bash
cat $FANTASIA_DIR/plans/fix-<slug>/VERIFICATION.md

# Check: did the regression test pass?
# Check: did other tests break?
```

**If regression test FAILS** → Go to Phase 5b (Feedback Loop)

**If regression test PASSES but other tests FAIL**:
- Ask: "The bug appears fixed, but other tests broke. Would you like to investigate the new failures?"
- If yes, treat as a new issue or iterate on the fix
- If no, proceed to Phase 6

**If all tests PASS** → Proceed to Code Review below, then Phase 6

### Spawn Code Reviewer

Use the Task tool for quick review:

```
Review this bug fix:

**Bug**: [summary]
**Root Cause**: [from investigation]
**Fix**: [what changed]

Verify:
1. Does the fix address the root cause (not just symptom)?
2. Are there edge cases not handled?
3. Could this fix cause other issues?
4. Is the regression test adequate?
```

## Phase 5b: Feedback Loop (on verification failure)

This phase handles the case where the fix didn't work. It can be triggered:
- Automatically when verification fails
- Manually via `/fantasia:feedback` command

### Capture Failure Context

```bash
# Get current attempt number from INVESTIGATION.md
ATTEMPT=$(grep -c "## Attempt" $FANTASIA_DIR/plans/$FIX_SLUG/INVESTIGATION.md 2>/dev/null || echo "1")
NEXT_ATTEMPT=$((ATTEMPT + 1))
```

### Present Failure

```
❌ Verification Failed
======================

**Attempt**: $ATTEMPT
**Test Result**: <failure output>

The regression test is still failing. Let's figure out what's happening.
```

### Check Soft Limit

If `NEXT_ATTEMPT > 3`:

```
⚠️  Multiple Attempts Warning
==============================

This will be attempt $NEXT_ATTEMPT at fixing this bug. Multiple failed attempts might indicate:

- A deeper architectural issue
- Missing context about the system
- Need for a fundamentally different approach

**Options**:
1. **Continue** - Try attempt $NEXT_ATTEMPT with new information
2. **Re-map** - Step back and re-analyze the relevant code area
3. **Pause** - Stop here and discuss with a teammate

What would you like to do?
```

If user chooses to pause or re-map, update state and exit.

### Gather Feedback

Ask clarifying questions based on the failure:

```
What did you observe? For example:
- Is the error identical to before, or different?
- Did you see any partial improvement?
- Any new information about when/how it fails?
```

Wait for user response.

### Decide Path

Based on feedback, determine next step:

**Re-investigate** (root cause may be wrong):
- Completely different error
- No improvement despite targeted fix
- User says "wrong code path"

**Iterate on fix** (root cause correct, fix insufficient):
- Partial improvement
- Works for some cases
- Edge case not handled

Announce decision:
```
Based on your feedback, it looks like [the root cause was misidentified / the fix
approach needs adjustment]. Proceeding to [re-investigate / try a different fix].
```

### Update Files

**Update state** (`$FANTASIA_DIR/fantasia-state.md`):
```markdown
**Phase**: fixing
**Plan**: $FIX_SLUG
**Attempt**: $NEXT_ATTEMPT
**Mode**: feedback-loop
**Feedback**: <user's observation>
**Decision**: <reinvestigate | iterate>
```

**Append to INVESTIGATION.md** (if re-investigating):
```markdown

## Attempt $NEXT_ATTEMPT (after feedback)

**Previous Attempt Summary**: <what was tried>
**Feedback Received**: <user's observation>
**Why Re-investigating**: <rationale>

### New Investigation
<Filled by bug-investigator>
```

**Append to PLAN.md** (if iterating):
```markdown

## Attempt $NEXT_ATTEMPT (after feedback)

**Previous Attempt Summary**: <what was tried>
**Feedback Received**: <user's observation>
**Why Iterating**: <rationale>

### Revised Fix Approach
<Filled during planning>
```

### Continue Workflow

**If re-investigating**: Go back to Phase 2 (Investigation) with new context

**If iterating**: Go back to Phase 3 (Fix Planning) with new context

The workflow then continues: Planning → Implementation → Verification

If verification fails again, this feedback loop repeats.

---

## Phase 6: Completion

### Write Summary

Create `$FANTASIA_DIR/plans/fix-<slug>/SUMMARY.md`:

```markdown
# Bug Fix Complete: <slug>

## Issue
<Original bug description>
<Sentry link if applicable>

## Root Cause
<What was wrong>

## Fix
<What we changed>

## Files Modified
<List>

## Tests Added
<New test file/function>

## Verification
- Regression test: PASS
- Full test suite: PASS
- Manual verification: <if done>

## Sentry Resolution
<If Sentry issue: Note to mark as resolved>
```

### Generate PR Description

Create `$FANTASIA_DIR/plans/fix-<slug>/PR-DESCRIPTION.md` with a well-formatted PR description ready to paste:

```markdown
## Summary

<1-2 sentence description of what this PR fixes>

## Bug Details

| | |
|---|---|
| **Issue** | <Sentry URL or ticket link if applicable> |
| **Severity** | <Critical/High/Medium/Low> |
| **Users Affected** | <count or "unknown"> |
| **First Reported** | <date if known> |

## Root Cause

<Clear explanation of what was causing the bug - technical but understandable>

## Changes Made

<Bulleted list of what was changed and why>

- `<file>`: <what changed>
- `<file>`: <what changed>

## Verification

- [x] Regression test added: `<test file/name>`
- [x] All existing tests pass
- [x] Type checking passes
- [x] Lint passes
<If applicable>
- [x] Manual verification: <what was tested manually>
</If>

## Test Plan

<How to verify this fix works>

1. <Step 1>
2. <Step 2>
3. Expected: <what should happen>

## Related

<If Sentry>
- Sentry: <link> (mark as resolved after merge)
</If>
<If ticket>
- Ticket: <link>
</If>
<If similar bugs were found>
- Related fixes may be needed: <brief note about similar patterns found>
</If>

---
🤖 Generated with [Fantasia](https://github.com/anthropics/claude-plugins)
```

**Present to user**:
```
📝 PR Description Ready

Saved to: $FANTASIA_DIR/plans/fix-<slug>/PR-DESCRIPTION.md

---
<Display the generated PR description>
---

Copy the above for your PR, or view the file directly.
```

### Update Sentry (if applicable)

If this was a Sentry issue:
- Note that the issue should be marked as resolved in Sentry
- Mention the commit/PR that contains the fix

### Save State

Update `$FANTASIA_DIR/fantasia-state.md`:
```markdown
**Phase**: idle
**Last Action**: fix complete
**Plan**: fix-<slug>
```

### Suggest Next Steps

- "Bug fix complete. Ready to commit."
- "Don't forget to mark the Sentry issue as resolved after merging."
- If in a worktree: "Use `/fantasia:worktree finish` when ready to merge."

---

## Bug Fixing Principles

1. **Understand before fixing** - Never fix what you don't understand
2. **Find the root cause** - Symptoms lie, root causes don't
3. **Test first** - Write the regression test before the fix
4. **Minimal changes** - Fix the bug, don't refactor
5. **Verify thoroughly** - A fix that breaks something else isn't a fix

## Sentry Integration

This command integrates with Sentry when available:

- **Input**: Sentry issue URL or ID (e.g., `PROJ-123` or full URL)
- **Fetches**: Error details, stack trace, frequency, user impact
- **Provides**: Rich context for investigation
- **Suggests**: Marking resolved after merge

Use the Sentry MCP tools (`/sentry:seer`, `sentry:getIssues`) for direct Sentry access.
