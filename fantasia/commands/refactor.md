---
description: Plan and execute a refactoring with behavior preservation verification
argument-hint: "<target> [--yolo]"
allowed-tools: ["Read", "Bash", "Glob", "Grep", "Write", "Edit", "Task", "AskUserQuestion"]
---

# Fantasia Refactor Mode

You are orchestrating a **refactoring workflow** - improving code structure without changing behavior.

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

**Target**: `$ARGUMENTS.target`
**Flags**: `$ARGUMENTS.flags`

Check for YOLO mode:
```
YOLO_MODE = "--yolo" in flags
```

## Prerequisites

1. **Check for codebase maps**:
   ```bash
   ls $FANTASIA_DIR/codebase/
   ```

   If maps don't exist, offer to run `/fantasia:map` first. Refactoring benefits greatly from understanding existing patterns.

2. **Read relevant maps**:
   - `ARCHITECTURE.md` - Understand component boundaries
   - `CONVENTIONS.md` - Know the patterns to follow
   - `CONCERNS.md` - Identify existing tech debt context

## Phase 1: Analysis

### Identify Refactoring Scope

Based on the target "$ARGUMENTS.target", determine:

1. **What files/code are involved?**
   - Use Glob/Grep to find the target
   - Note the file paths for the parallel analyzers

2. **Gather initial context**
   - Read the target code
   - Get a sense of what we're working with

### Spawn Parallel Analysis Agents

Use the Task tool to launch **3 analyzers in parallel**:

```
Agent 1 - Code Smell Detector:
Analyze this code for code smells and structural issues:

**Target**: [target description]
**Files**: [list of files]

Your job:
1. Identify specific code smells with locations
2. Categorize by severity (Critical/Major/Minor)
3. Prioritize what to fix first
4. Flag any red flags

Write your findings to: $FANTASIA_DIR/plans/refactor-<slug>/CODE-SMELLS.md

---

Agent 2 - Dependency Mapper:
Map dependencies for this refactoring target:

**Target**: [target description]
**Files**: [list of files]

Your job:
1. Map incoming dependencies (what uses this)
2. Map outgoing dependencies (what this uses)
3. Assess blast radius of changes
4. Identify coupling issues

Write your findings to: $FANTASIA_DIR/plans/refactor-<slug>/DEPENDENCIES.md

---

Agent 3 - Test Coverage Checker:
Evaluate test coverage for this refactoring target:

**Target**: [target description]
**Files**: [list of files]

Your job:
1. Find all tests that cover this code
2. Assess test quality and completeness
3. Identify coverage gaps
4. Recommend: proceed or add tests first

Write your findings to: $FANTASIA_DIR/plans/refactor-<slug>/TEST-COVERAGE.md
```

**Launch all three in parallel** - they will write their analysis to separate files.

### Synthesize Analysis Results

After all analyzers complete:

1. **Read all analysis files**:
   - CODE-SMELLS.md - What to refactor
   - DEPENDENCIES.md - What might break
   - TEST-COVERAGE.md - Is it safe to refactor

2. **Consolidate into** `$FANTASIA_DIR/plans/refactor-<slug>/ANALYSIS.md`:

```markdown
# Refactoring Analysis: <target>

## Executive Summary
<2-3 sentence overview combining all analyses>

## Refactoring Readiness
- **Safe to Proceed**: Yes/No/Needs Tests First
- **Risk Level**: Low/Medium/High
- **Blast Radius**: <number of affected files>

## Code Smells Found
<Summary from code-smell-detector>
- Critical: <count>
- Major: <count>
- Minor: <count>
- Top priority: <most important issue>

## Dependency Analysis
<Summary from dependency-mapper>
- Incoming dependencies: <count>
- Outgoing dependencies: <count>
- Circular dependencies: Yes/No
- High-risk changes: <list>

## Test Coverage Assessment
<Summary from test-coverage-checker>
- Coverage level: Good/Partial/Poor
- Tests found: <count>
- Critical gaps: <list>

## Recommendation
<Synthesized recommendation: proceed, add tests, or reconsider>

## Refactoring Priorities
1. <Highest priority refactoring>
2. <Second priority>
3. ...
```

### Legacy: Spawn Refactor Analyzer (if additional analysis needed)

If the parallel analysis is insufficient, use the Task tool to launch the refactor-analyzer agent for deeper analysis:

```
Analyze the following code for refactoring:

**Target**: [target description]
**Files**: [list of files]
**Prior Analysis**: See CODE-SMELLS.md, DEPENDENCIES.md, TEST-COVERAGE.md

Your job:
1. Synthesize the parallel analysis findings
2. Identify any gaps in the analysis
3. Provide final refactoring recommendation

Update: $FANTASIA_DIR/plans/refactor-<slug>/ANALYSIS.md
```

## Phase 2: Planning

### Create Refactoring Plan

Based on the analysis, create a safe refactoring plan:

1. **Behavior Preservation Strategy**
   - What tests verify current behavior?
   - Do we need to add tests first?
   - What's our rollback strategy?

2. **Step-by-Step Approach**
   - Small, atomic changes
   - Each step should pass tests
   - Clear commit points

3. **Verification Checkpoints**
   - After each step: run tests
   - Before/after: verify behavior equivalence

### Write Plan

Create `$FANTASIA_DIR/plans/refactor-<slug>/PLAN.md`:

```markdown
# Refactoring Plan: <target>

## Goal
<What we're improving and why>

## Current State
<Summary of analysis findings>

## Risks
<What could go wrong, mitigation strategies>

## Pre-Refactoring Checklist
- [ ] All existing tests pass
- [ ] Test coverage is adequate (or we'll add tests first)
- [ ] Dependencies mapped

## Steps

### Step 1: <description>
- Files: <list>
- Changes: <what changes>
- Verify: <how to verify>

### Step 2: ...

## Post-Refactoring Verification
- [ ] All tests pass
- [ ] No behavior changes (unless intentional)
- [ ] Code follows CONVENTIONS.md patterns
```

### Get Approval (unless YOLO)

If not YOLO_MODE:
- Present the plan summary
- Ask: "Ready to proceed with refactoring? (yes/no/modify)"
- Wait for approval

## Phase 3: Execution

### Pre-Flight Checks

```bash
# Run existing tests to establish baseline
# (Use appropriate test command for the repo)
```

If tests fail before refactoring, STOP and report.

### Execute Refactoring Steps

For each step in the plan:

1. **Make the change**
   - Keep changes minimal and focused
   - Follow patterns from CONVENTIONS.md

2. **Verify immediately**
   ```bash
   # Run tests after each step
   ```

3. **Checkpoint**
   - If tests pass, note progress
   - If tests fail, stop and assess

### Handle Failures

If a step breaks tests:
1. Analyze what went wrong
2. Either fix forward or revert the step
3. Update the plan if needed
4. Ask user before continuing (unless YOLO)

## Phase 4: Verification

### Environment Setup (if in worktree)

If working in a git worktree, ensure the environment is set up:

```bash
# Check if in a worktree
if git rev-parse --is-inside-work-tree &>/dev/null && [ -f .envrc ]; then
  # Enable direnv for this worktree
  direnv allow
fi
```

### Run Parallel Verification

Use the Task tool to spawn the **parallel-verifier** agent:

```
Run verification checks for this refactoring:

**Changed Files**: [list of modified files]
**Codebase Type**: [detected from maps or repo structure]

Your job:
1. Run the test suite (focused on affected areas)
2. Run type checking (if applicable)
3. Run linting (if applicable)
4. Provide consolidated pass/fail assessment

Write results to: $FANTASIA_DIR/plans/refactor-<slug>/VERIFICATION.md
```

### Review Verification Results

After parallel verification completes:

1. **All tests pass** - Behavior preserved
2. **Type checks pass** - No type regressions
3. **Lint passes** - Code meets standards
4. **No unintended behavior changes**

### Spawn Code Reviewer

Use the Task tool to launch code-reviewer agent on the changes:

```
Review the refactoring changes:

**Original goal**: [from plan]
**Files changed**: [list]

Focus on:
1. Behavior preservation - did we change anything we shouldn't?
2. Pattern compliance - does new code match CONVENTIONS.md?
3. Missed opportunities - anything else to clean up?
4. Risk assessment - any concerns about the changes?
```

## Phase 5: Completion

### Write Summary

Create/update `$FANTASIA_DIR/plans/refactor-<slug>/SUMMARY.md`:

```markdown
# Refactoring Complete: <target>

## What Changed
<Summary of changes made>

## Files Modified
<List of files>

## Verification
- Tests: PASS
- Behavior: Preserved
- Patterns: Compliant

## Follow-up Recommendations
<Any additional improvements identified>
```

### Generate PR Description

Create `$FANTASIA_DIR/plans/refactor-<slug>/PR-DESCRIPTION.md`:

```markdown
## Summary

<1-2 sentence description of the refactoring>

## Motivation

<Why this refactoring was needed - code smells addressed, maintainability improvements>

## Changes

### Code Structure
<High-level description of structural changes>

### Files Modified

| File | Change |
|------|--------|
| `<file>` | <what changed> |
| `<file>` | <what changed> |

## Behavior Preservation

This refactoring makes **no changes to external behavior**:

- [x] All existing tests pass
- [x] No public API changes
- [x] No changes to function signatures (or changes are backwards compatible)
<If applicable>
- [x] Manual verification: <what was tested>
</If>

## Verification

- [x] Test suite: PASS
- [x] Type checking: PASS
- [x] Lint: PASS
- [x] Code review: No behavior changes detected

## Code Quality Improvements

<Bulleted list of improvements>

- Reduced complexity in `<area>`
- Improved readability of `<area>`
- Better separation of concerns in `<area>`

## Follow-up

<Any additional refactoring or improvements identified but not included in this PR>

---
🤖 Generated with [Fantasia](https://github.com/anthropics/claude-plugins)
```

**Present to user**:
```
📝 PR Description Ready

Saved to: $FANTASIA_DIR/plans/refactor-<slug>/PR-DESCRIPTION.md

---
<Display the generated PR description>
---

Copy the above for your PR, or view the file directly.
```

### Save State

Update `$FANTASIA_DIR/fantasia-state.md`:
```markdown
**Phase**: idle
**Last Action**: refactor complete
**Plan**: refactor-<slug>
```

### Suggest Next Steps

- "Refactoring complete. Run tests one more time before committing."
- "Consider running `/fantasia:review` for additional verification."
- If in a worktree: "Use `/fantasia:worktree finish` when ready to merge."

---

## Refactoring Principles

1. **Never refactor without tests** - Add them first if needed
2. **Small steps** - Each change should be independently verifiable
3. **One thing at a time** - Don't mix refactoring with feature work
4. **Preserve behavior** - The goal is better structure, not different behavior
5. **Follow existing patterns** - Refactoring should make code MORE consistent
