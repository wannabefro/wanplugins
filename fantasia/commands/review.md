---
description: Review changes with parallel agents - code quality, bugs, and test coverage analysis
argument-hint: "[--task <name>]"
allowed-tools: ["Read", "Bash", "Glob", "Grep", "Write", "Task"]
---

You are orchestrating a Fantasia code review operation. Your job is to spawn parallel review agents that analyze recent changes for quality, bugs, and test coverage.

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

## Determine Scope

1. Check if `$ARGUMENTS` contains `--task`:
   - If yes, extract the task name and look for `$FANTASIA_DIR/plans/<task-name>/PLAN.md` to understand what was built
   - If no, find the most recent plan and review changes from that task

2. Identify what to review:
   - Read the plan to understand what files were created/modified
   - Or use `git diff` / `git status` to find recent changes

## Prerequisites Check

1. Verify there are changes to review:
   ```bash
   git status --porcelain
   ```

2. If no changes found:
   "No uncommitted changes found. What would you like to review?
   - Recent commits: I can review the last N commits
   - Specific files: Tell me which files to review
   - Full codebase: Run /fantasia:map for a full analysis"

## Load Configuration

Check for project-specific configuration:

```bash
CONFIG_FILE="$FANTASIA_DIR/fantasia-config.md"
[ -f "$CONFIG_FILE" ] && echo "CONFIG_EXISTS=true"
```

If config exists, extract model overrides for reviewer agents:
- `code-reviewer`: default sonnet
- `bug-hunter`: default sonnet
- `test-reviewer`: default haiku

Also check for `default-model` setting.

## Read Context

Read relevant codebase maps to inform the review:
- CONVENTIONS.md (for code quality assessment)
- TESTING.md (for test coverage assessment)
- CONCERNS.md (to avoid introducing known issues)

## Identify Files to Review

```bash
# Get list of changed files
git diff --name-only HEAD 2>/dev/null || git status --porcelain | awk '{print $2}'
```

Create a file list to pass to review agents.

## Spawn Parallel Review Agents

Use the Task tool to spawn 3 review agents in parallel. Each writes findings and returns a summary.

**MODEL SELECTION:**
When spawning each agent, set the `model` parameter based on:
1. Config-specified model for this agent (if set in `$FANTASIA_DIR/fantasia-config.md`)
2. Config `default-model` (if set)
3. Built-in defaults:
   - code-reviewer: sonnet
   - bug-hunter: sonnet
   - test-reviewer: haiku

**CRITICAL FOR TOKEN EFFICIENCY:**
- Pass only the changed files to each agent (not entire codebase)
- Extract and pass only relevant sections from maps (not full files)
- Agents write ALL findings directly to files
- Agents return ONLY completion status (e.g., "✓ Done") - DO NOT parse responses for data
- YOU read the output files to compile the review - never ask agents to return findings in their response

### Agent 1: code-reviewer
```
Review these files for quality and patterns compliance.

Files: <list of changed files>
Conventions: <extract 10-20 key lines from CONVENTIONS.md>
Output: $FANTASIA_DIR/plans/<task>/CODE-QUALITY.md
```
Agent has built-in checklists for style, patterns, and maintainability.

### Agent 2: bug-hunter
```
Hunt for bugs and edge cases in these files.

Files: <list of changed files>
Output: $FANTASIA_DIR/plans/<task>/BUG-ANALYSIS.md
```
Agent has built-in checklists for logic errors, edge cases, error handling, and security.

### Agent 3: test-reviewer
```
Review test coverage and quality for these changes.

Files: <list of changed files>
Test patterns: <extract 10-20 key lines from TESTING.md>
Output: $FANTASIA_DIR/plans/<task>/TEST-COVERAGE.md
```
Agent has built-in checklists for coverage analysis and test quality.

## Check for Ticket Requirements

Before compiling review, check if this task originated from a ticket:

```bash
[ -f "$FANTASIA_DIR/plans/$TASK_SLUG/TICKET.md" ] && echo "has_ticket=true"
```

If a ticket exists, extract acceptance criteria for verification.

## Compile Review

After all 3 agents complete, read their output files and compile into `$FANTASIA_DIR/plans/<task>/REVIEW.md`:

**Read these files** (written by agents):
- `$FANTASIA_DIR/plans/<task>/CODE-QUALITY.md`
- `$FANTASIA_DIR/plans/<task>/BUG-ANALYSIS.md`
- `$FANTASIA_DIR/plans/<task>/TEST-COVERAGE.md`

**Compile into**:

```markdown
# Review: <Task Name>

**Date**: <timestamp>
**Files Reviewed**: <count>

## Summary

| Category | Issues | Critical | High | Medium | Low |
|----------|--------|----------|------|--------|-----|
| Code Quality | X | - | - | - | - |
| Potential Bugs | Y | - | - | - | - |
| Test Coverage | Z | - | - | - | - |

## Code Quality Review
<Detailed findings from code-reviewer>

## Bug Analysis
<Detailed findings from bug-hunter>

## Test Coverage
<Detailed findings from test-reviewer>

## Action Items

### Must Fix (Critical/High)
1. <Issue with file:line>
2. ...

### Should Fix (Medium)
1. <Issue with file:line>
2. ...

### Consider (Low)
1. <Issue with file:line>
2. ...

## Acceptance Criteria
<!-- Only include if TICKET.md exists -->
| Criterion | Status | Evidence |
|-----------|--------|----------|
| <from ticket> | ✅/❌ | <file:line or test> |

## Verdict
- [ ] ✅ Ready to commit
- [ ] 🟡 Minor issues - can commit with notes
- [ ] 🔴 Issues found - recommend fixes first
```

## Completion

Present summary to user:

```
📋 Review complete: <task-name>

## Summary
- Code Quality: X issues (Y high)
- Potential Bugs: X issues (Y critical/high)
- Test Coverage: <assessment>

<If ticket exists>
## Acceptance Criteria: X/Y met
- ✅ <criterion 1>
- ✅ <criterion 2>
- ❌ <criterion 3> - needs work
</If>

## Verdict: <Ready/Minor Issues/Needs Fixes>

<If issues found>
Top priorities to address:
1. <Most critical issue>
2. <Second priority>

Full review: $FANTASIA_DIR/plans/<task>/REVIEW.md

Would you like me to help fix any of these issues?
</If>

<If clean>
No significant issues found. Ready to commit!
</If>
```

## Save Checkpoint

After review completes, save state:

```bash
TASK_SLUG="<the-task-slug>"

# Preserve mode from previous state if it exists (handles Mode: and **Mode**:)
CURRENT_MODE=$(sed -n -E 's/^[[:space:]]*([*][*])?Mode([*][*])?:[[:space:]]*//p' $FANTASIA_DIR/fantasia-state.md 2>/dev/null | head -1)
if [ -z "$CURRENT_MODE" ]; then
  CURRENT_MODE="interactive"
fi

cat > $FANTASIA_DIR/fantasia-state.md << EOF
# Fantasia Checkpoint

**Saved**: $(date -Iseconds)
**Phase**: idle
**Last Action**: review complete
**Plan**: $TASK_SLUG
**Mode**: $CURRENT_MODE

## Context
- Maps: available in $FANTASIA_DIR/codebase/
- Plan: $FANTASIA_DIR/plans/$TASK_SLUG/PLAN.md
- Build: complete
- Review: complete ($FANTASIA_DIR/plans/$TASK_SLUG/REVIEW.md)

## Review Summary
<Verdict and key findings>

## Next Step
Address review findings or commit the changes.

## Workflow Complete
This task has been mapped → planned → built → reviewed.
Start a new task with \`/fantasia:plan <new-task>\`.
EOF
```

## Generate PR Description

After review completes with verdict "Ready" or "Minor Issues", generate a PR description.

Create `$FANTASIA_DIR/plans/<task>/PR-DESCRIPTION.md`:

```markdown
## Summary

<1-2 sentence description of what this PR does, derived from PLAN.md>

## Changes

<Bulleted list of key changes, grouped logically>

### <Category 1>
- <Change description>
- <Change description>

### <Category 2>
- <Change description>

## Files Changed

| File | Change |
|------|--------|
| `<file>` | <brief description> |
| `<file>` | <brief description> |

## Verification

- [x] All tests pass
- [x] Type checking passes
- [x] Lint passes
- [x] Code review complete (see REVIEW.md)

<If ticket exists>
## Acceptance Criteria

| Criterion | Status |
|-----------|--------|
| <criterion from TICKET.md> | ✅ |
| <criterion from TICKET.md> | ✅ |

Closes: <ticket link if available>
</If>

## Test Plan

<How to verify these changes work>

1. <Step 1>
2. <Step 2>
3. Expected: <what should happen>

<If review found issues that weren't fixed>
## Known Issues

<Minor issues identified in review that are deferred>

- <Issue>: <why deferred>
</If>

---
🤖 Generated with [Fantasia](https://github.com/anthropics/claude-plugins)
```

**Present to user**:
```
📝 PR Description Ready

Saved to: $FANTASIA_DIR/plans/<task>/PR-DESCRIPTION.md

---
<Display the generated PR description>
---

Copy the above for your PR, or view the file directly.
```
