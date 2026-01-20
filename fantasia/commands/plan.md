---
description: Plan a feature with thorough understanding before building - discovery, verification, design, and readiness check
argument-hint: "<task-description>"
allowed-tools: ["Read", "Bash", "Glob", "Grep", "Write", "Task", "AskUserQuestion"]
---

You are creating an implementation plan for a feature using the Fantasia plugin. This is NOT a quick outline - you must thoroughly understand the task before designing a plan.

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

## Argument Required

The user must provide a task description. If `$ARGUMENTS` is empty, ask:
"What feature or task would you like to plan? Describe it briefly."

Task: `$ARGUMENTS`

## Prerequisites Check

1. Check if `$FANTASIA_DIR/codebase/` exists with map files
2. If maps don't exist, offer: "No codebase maps found. Would you like me to run /fantasia:map first? (y/n)"
3. If user agrees, guide them to run `/fantasia:map` and return here

## Create Task Directory

Slugify the task name (lowercase, hyphens, no special chars):
```bash
TASK_SLUG=$(echo "$ARGUMENTS" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/-/g' | sed 's/--*/-/g' | sed 's/^-//' | sed 's/-$//')
mkdir -p "$FANTASIA_DIR/plans/$TASK_SLUG"
```

## Load Organization Context

Check for organization-wide context:

```bash
ORG_CONTEXT_FILE="$HOME/.claude/fantasia/org-context.md"
if [ -f "$ORG_CONTEXT_FILE" ]; then
  echo "ORG_CONTEXT_EXISTS=true"
fi
```

If org context exists:
1. Read and parse the YAML frontmatter
2. Match repository patterns against current repo (using `detection` markers)
3. Extract `coding_standards` for use in planning
4. Note any relevant `integrations` for the task

Store matched pattern info as `ORG_CONTEXT` for use in later phases:
- Build system and commands (for understanding what validation looks like)
- Coding standards (for design decisions)
- Pre-commit configuration (for understanding CI expectations)

---

## Phase 1: Discovery

**Goal**: Deeply understand what needs to be built.

### 1.1 Read Relevant Maps
Read the codebase maps that are relevant to this task:
- Always read: ARCHITECTURE.md, STRUCTURE.md, CONVENTIONS.md
- Read STACK.md if task involves new dependencies
- Read INTEGRATIONS.md if task involves external services
- Read TESTING.md if task involves test changes

### 1.2 Explore Task-Relevant Code
Based on the task description:
- Use Glob to find files that might be affected
- Use Grep to find related patterns, functions, or components
- Read key files to understand current implementation

### 1.3 Ask Clarifying Questions
If anything is unclear about the task, use AskUserQuestion to clarify:
- Scope boundaries (what's included, what's not)
- Technical preferences (specific libraries, patterns)
- Priority trade-offs (speed vs thoroughness)
- Edge cases to handle

Do NOT proceed until you have enough clarity.

---

## Phase 2: Verify Understanding

**Goal**: Confirm you understand enough to build this well.

Ask yourself:
1. Do I understand where this code will live?
2. Do I understand what existing patterns to follow?
3. Do I understand the data flow?
4. Do I understand the edge cases?
5. Do I understand how this will be tested?

If any answer is "no", go back to Discovery and gather more information.

Create a brief understanding summary (internal, not written to file yet):
- "I understand that..."
- "The key components involved are..."
- "This follows the pattern of..."
- "The main challenges are..."

---

## Phase 3: Design

**Goal**: Create a concrete implementation plan with input from multiple perspectives.

### 3.1 Parallel Approach Exploration

Before committing to one design, explore multiple approaches in parallel.

**Spawn 3 Approach Explorer Agents** using the Task tool:

```
Agent 1 - "Simplest Approach":
Explore the simplest implementation for: <task description>

Constraint: Optimize for SIMPLICITY - fewest moving parts, easiest to understand.
Context from maps: <relevant architecture/conventions>
Write to: $FANTASIA_DIR/plans/$TASK_SLUG/APPROACH-SIMPLE.md

Agent 2 - "Most Maintainable Approach":
Explore the most maintainable implementation for: <task description>

Constraint: Optimize for MAINTAINABILITY - future developers will thank us.
Context from maps: <relevant architecture/conventions>
Write to: $FANTASIA_DIR/plans/$TASK_SLUG/APPROACH-MAINTAINABLE.md

Agent 3 - "Minimal Changes Approach":
Explore the minimal-change implementation for: <task description>

Constraint: Optimize for MINIMAL DIFF - smallest change that solves the problem.
Context from maps: <relevant architecture/conventions>
Write to: $FANTASIA_DIR/plans/$TASK_SLUG/APPROACH-MINIMAL.md
```

**Launch all three in parallel** - they will write their approaches to separate files.

### 3.2 Synthesize Approaches

After all explorers complete, read and compare their outputs:

1. **Read all approach files**
2. **Compare trade-offs**:
   - Which approach best fits the codebase conventions?
   - Which approach has the lowest risk?
   - Which approach balances effort vs. quality?
3. **Select or synthesize**:
   - Pick the best approach, OR
   - Combine elements from multiple approaches

Present to user (unless YOLO mode):
```
📊 Approaches Explored

## Simplest Approach
<Summary and trade-offs>

## Most Maintainable Approach
<Summary and trade-offs>

## Minimal Changes Approach
<Summary and trade-offs>

## Recommendation
<Which approach or combination to use, and why>

Proceed with recommended approach? (yes/no/discuss)
```

### 3.3 Identify Work Breakdown
Break the selected approach into concrete steps:
- What files need to be created?
- What files need to be modified?
- What's the order of operations?
- What are the dependencies between steps?

### 3.4 Identify Specialists Needed
Based on the work, determine which Fantasia agents should handle each part:
- **architect**: Design decisions, contracts, interfaces
- **implementer**: Core logic implementation
- **integrator**: API boundaries, external connections

### 3.6 Identify External Dependencies
If this task requires work in other repos:
- What APIs/contracts are needed?
- What can be mocked for now?
- What's blocked until other work completes?

### 3.7 Write the Plan

Write to `$FANTASIA_DIR/plans/<task-slug>/PLAN.md`:

```markdown
# Plan: <Task Name>

## Summary
<2-3 sentence description of what we're building>

## Understanding
<Key insights from discovery phase>
- Current state: ...
- Target state: ...
- Key patterns to follow: ...

## Work Breakdown

### Phase 1: <Name>
**Agent**: architect | implementer | integrator
**Files**:
- Create: `path/to/new/file.ts`
- Modify: `path/to/existing/file.ts`

**Tasks**:
1. <Specific task>
2. <Specific task>

**Acceptance Criteria**:
- [ ] <Criterion>

### Phase 2: <Name>
...

## Execution Order
1. Phase 1 (architect) - can start immediately
2. Phase 2 (implementer) - depends on Phase 1
3. Phase 3 (integrator) - can run parallel with Phase 2

## Testing Strategy
- Unit tests: ...
- Integration tests: ...

## Risks & Mitigations
- Risk: ...
  Mitigation: ...
```

### 3.8 Write External Dependencies (if any)

If cross-repo work is needed, write to `$FANTASIA_DIR/plans/<task-slug>/EXTERNAL-DEPS.md`:

```markdown
# External Dependencies: <Task Name>

## Required from: <other-repo-name>

### <Dependency Name>
**Type**: API endpoint | shared type | service
**Status**: Not started | In progress | Available

**Specification**:
<Details of what's needed>

### Handling
- [ ] Mock locally until available
- [ ] Block until dependency ready
- [ ] Partial implementation possible
```

---

## Phase 3.9: File Path Validation (Anti-Hallucination)

**CRITICAL: Verify every file path in the plan before presenting to user.**

### Validation Steps

Before finalizing the plan, validate all file paths:

```bash
# For each "Create: path/to/file.ts" - verify parent directory exists
for dir in $(grep -E "^\s*-?\s*Create:" $FANTASIA_DIR/plans/$TASK_SLUG/PLAN.md | sed 's/.*Create:\s*//' | sed 's/`//g' | xargs -I{} dirname {}); do
  if [ ! -d "$dir" ] && [ ! -d "$(git rev-parse --show-toplevel)/$dir" ]; then
    echo "⚠️ CREATE PATH INVALID: $dir does not exist"
  fi
done

# For each "Modify: path/to/file.ts" - verify file exists
for file in $(grep -E "^\s*-?\s*Modify:" $FANTASIA_DIR/plans/$TASK_SLUG/PLAN.md | sed 's/.*Modify:\s*//' | sed 's/`//g'); do
  if [ ! -f "$file" ] && [ ! -f "$(git rev-parse --show-toplevel)/$file" ]; then
    echo "⚠️ MODIFY PATH INVALID: $file does not exist"
  fi
done
```

### If Paths Are Invalid

**DO NOT present the plan to the user.** Instead:

1. Fix the invalid paths by:
   - Re-reading the codebase structure
   - Finding the correct paths with `find` or `ls`
   - Updating the plan with correct paths

2. Re-validate until all paths pass

3. Only then proceed to Readiness Check

### Validation Output

```
✓ Plan path validation:
  - Create paths: X verified
  - Modify paths: Y verified
  - All paths valid: YES/NO
```

If NO, list invalid paths and fix them before proceeding.

---

## Phase 4: Readiness Check

**Goal**: Get user approval before building.

Present the plan summary to the user:

```
📋 Plan ready for: <task-name>

## Summary
<Brief description>

## Work Phases
1. <Phase 1 name> (architect)
2. <Phase 2 name> (implementer)
3. <Phase 3 name> (integrator)

## Files Affected
- Create: X new files
- Modify: Y existing files

## External Dependencies
<None | List dependencies>

---

Plan written to: $FANTASIA_DIR/plans/<task-slug>/PLAN.md

Ready to build? Run /fantasia:build
Or review the plan file first and let me know if you'd like changes.
```

Wait for user feedback. If they request changes, update the plan.

## Save Checkpoint

After plan is written, save state:

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
**Last Action**: plan complete
**Plan**: $TASK_SLUG
**Mode**: $CURRENT_MODE

## Context
- Maps: available in $FANTASIA_DIR/codebase/
- Plan: $FANTASIA_DIR/plans/$TASK_SLUG/PLAN.md
- Build: not started
- Review: not started

## Next Step
Run \`/fantasia:build\` to execute the plan.

## To Resume
Run \`/fantasia:resume\` to restore context and continue.
EOF
```
