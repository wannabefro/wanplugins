---
description: Provide feedback that a fix isn't working as expected, enabling iterative refinement
argument-hint: "<observation> [--reinvestigate] [--iterate]"
allowed-tools: ["Read", "Bash", "Glob", "Grep", "Write", "Edit", "Task", "AskUserQuestion"]
---

# Fantasia Feedback

You are handling **feedback on a fix that didn't work** — enabling iterative refinement.

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

**Observation**: `$ARGUMENTS.observation`
**Flags**: `$ARGUMENTS.flags`

```
FORCE_REINVESTIGATE = "--reinvestigate" in flags
FORCE_ITERATE = "--iterate" in flags
```

## Phase 1: Find Current Fix Context

### Locate Most Recent Fix

```bash
# Find fix plans (most recent first)
ls -t $FANTASIA_DIR/plans/ 2>/dev/null | grep "^fix-" | head -5

# Read current state
cat $FANTASIA_DIR/fantasia-state.md 2>/dev/null
```

### Validate Fix Exists

If no fix plan is found:
```
No active fix found. Use /fantasia:fix to start investigating a bug first.

If you're providing feedback on a build or other workflow, please describe
the issue and I'll help you address it directly.
```

### Load Fix Context

Read the relevant fix files:
```bash
FIX_SLUG=<detected from state or most recent fix plan>

# Core context
cat $FANTASIA_DIR/plans/$FIX_SLUG/INVESTIGATION.md
cat $FANTASIA_DIR/plans/$FIX_SLUG/PLAN.md

# Sentry context if exists
cat $FANTASIA_DIR/plans/$FIX_SLUG/SENTRY.md 2>/dev/null

# Previous summary if exists (indicates prior completion attempt)
cat $FANTASIA_DIR/plans/$FIX_SLUG/SUMMARY.md 2>/dev/null
```

### Determine Current Attempt

Check for existing attempt markers in INVESTIGATION.md:
- Count "## Attempt N" sections
- Current attempt = highest N + 1

If no attempt markers, this is transitioning from Attempt 1 to Attempt 2.

## Phase 2: Acknowledge Feedback

Present what we know:

```
📋 Feedback Received
====================

**Fix**: <fix-slug>
**Current Attempt**: <N>
**Your Observation**: <observation from arguments>

**Previous Root Cause**: <from INVESTIGATION.md>
**Previous Fix Approach**: <from PLAN.md>
```

## Phase 3: Clarifying Questions

Based on the observation, ask targeted questions to understand what happened.

### Detect Observation Type

Analyze the observation to categorize it:

| Pattern | Type | Follow-up Question |
|---------|------|-------------------|
| "same error", "still failing", "didn't work" | `SAME_ERROR` | "Is the error identical to before, or subtly different? Did you notice any partial improvement?" |
| "different error", "new error", "now shows" | `DIFFERENT_ERROR` | "Can you describe the new error? This might mean the original issue is fixed but uncovered something else." |
| "works for X but not Y", "partial", "sometimes" | `PARTIAL_FIX` | "Which specific cases work now, and which still fail? This helps narrow down what's different." |
| "works sometimes", "intermittent", "flaky" | `INTERMITTENT` | "What conditions seem to make it work vs fail? (timing, specific data, environment, load)" |
| Other | `UNCLEAR` | "Can you tell me more about what you observed? What behavior did you expect vs what happened?" |

### Ask the Question

Use AskUserQuestion or direct conversation to get the answer.

Wait for response before proceeding.

## Phase 4: Decide Path

Based on the observation and clarifying answers, determine the next step.

### Force Flags

If `FORCE_REINVESTIGATE`:
- Skip decision logic
- Go directly to re-investigation

If `FORCE_ITERATE`:
- Skip decision logic
- Go directly to fix iteration

### Decision Logic

**Signals for Re-investigation** (root cause was likely wrong):
- Completely different error type
- Error in a different part of the system
- "I think we were looking at the wrong thing"
- No improvement at all despite targeted fix

**Signals for Iteration** (root cause correct, fix insufficient):
- Partial improvement
- Same error but less frequent
- Works for some cases but not others
- Edge case not handled

### Present Decision

```
🔍 Analysis
===========

Based on your feedback, it appears that:

<If re-investigate>
**The root cause may have been misidentified.** The [reason from feedback] suggests
we need to look elsewhere.

→ Proceeding to re-investigate with this new information.
</If>

<If iterate>
**The root cause is likely correct, but the fix was insufficient.** The [partial
success / edge case / etc.] suggests we need to adjust our approach.

→ Proceeding to try a different fix approach.
</If>

Does this assessment seem right? (yes / no, actually...)
```

## Phase 5: Attempt Limits

### Hard Limit (Attempt 7+)

If this would be **Attempt 7 or higher**, stop and escalate. Do NOT continue the loop.

```
🛑 Max Attempts Reached
=======================

This would be attempt <N>. The hard limit is 6 attempts to avoid thrashing.

Recommended escalation paths:
1. **Pause and gather more evidence** (logs, repro steps, data samples)
2. **Escalate to a teammate or domain expert** for a fresh investigation
3. **Open a ticket** with the full timeline, evidence, and failures

Which path should we take?
```

Wait for user direction before proceeding.

### Soft Limit (Attempt 4+)

If this will be **Attempt 4 or higher**, show warning:

```
⚠️  Multiple Attempts Warning
==============================

This will be attempt <N> at fixing this bug. Multiple failed attempts might indicate:

- A deeper architectural issue
- Missing context about the system
- Need for a fundamentally different approach

**Options**:
1. **Continue** - Try attempt <N> with the new information
2. **Re-map** - Step back and re-analyze the relevant code area with /fantasia:map
3. **Pause** - Stop here and discuss with a teammate or think it over

What would you like to do?
```

If user chooses to continue, proceed. Otherwise, respect their choice.

## Phase 6: Update State and Files

### Update State File

```markdown
# Fantasia Checkpoint

**Saved**: <timestamp>
**Phase**: fixing
**Plan**: <fix-slug>
**Attempt**: <N>
**Mode**: feedback-loop

## Feedback
**Observation**: <user's observation>
**Decision**: <reinvestigate | iterate>
**Previous Attempts**: <N-1>

## Next Step
<Re-investigate root cause | Try different fix approach>
```

Write to `$FANTASIA_DIR/fantasia-state.md`.

### Append to Investigation (if re-investigating)

Add new attempt section to INVESTIGATION.md:

```markdown

## Attempt <N> (after feedback)

**Previous Attempt Summary**: <what was tried>
**Feedback Received**: <user's observation>
**Why Re-investigating**: <decision rationale>

### New Investigation

<To be filled by bug-investigator agent>
```

### Append to Plan (if iterating)

Add new attempt section to PLAN.md:

```markdown

## Attempt <N> (after feedback)

**Previous Attempt Summary**: <what was tried>
**Feedback Received**: <user's observation>
**Why Iterating**: <decision rationale>

### Revised Fix Approach

<To be filled during planning>
```

## Phase 7: Execute Next Step

### If Re-investigating

Spawn the bug-investigator agent with the new context:

```
Re-investigate this bug with new information:

**Original Issue**: <from SENTRY.md or BUG-REPORT.md>
**Previous Root Cause (Attempt <N-1>)**: <from INVESTIGATION.md>
**What Was Tried**: <from PLAN.md>
**Feedback**: <user's observation and clarification>

The previous fix didn't work because: <decision rationale>

Your job:
1. Consider what the feedback tells us about where the bug actually is
2. Look for alternative code paths or causes we might have missed
3. Re-examine assumptions from the previous investigation
4. Find the actual root cause

Update the "Attempt <N>" section in: $FANTASIA_DIR/plans/<fix-slug>/INVESTIGATION.md
```

After investigation completes, proceed to fix planning (Phase 3 of /fantasia:fix).

### If Iterating on Fix

Proceed directly to fix planning with the new context:

```
Plan a revised fix based on feedback:

**Root Cause**: <from INVESTIGATION.md - still believed correct>
**Previous Fix (Attempt <N-1>)**: <from PLAN.md>
**Why It Didn't Work**: <from feedback>

Design a fix that:
1. Addresses what the previous fix missed
2. Handles the cases that still fail
3. Maintains the fixes that worked

Update the "Attempt <N>" section in: $FANTASIA_DIR/plans/<fix-slug>/PLAN.md
```

After planning, proceed to implementation (Phase 4 of /fantasia:fix).

## Phase 8: Hand Off to Fix Workflow

After the appropriate phase completes, the fix workflow continues:

- **After re-investigation** → Planning → Implementation → Verification
- **After iteration planning** → Implementation → Verification

The verification phase will again check if the fix worked, potentially triggering another feedback loop if needed.

---

## Integration with /fantasia:fix

This command is designed to work seamlessly with the fix workflow:

1. `/fantasia:fix` runs through investigation → planning → implementation → verification
2. If verification fails, it can automatically trigger feedback flow
3. User can also invoke `/fantasia:feedback` manually at any time
4. Feedback updates state and continues the appropriate phase
5. The cycle repeats until the fix works or user decides to stop

## Examples

```bash
# After a fix that didn't work
/fantasia:feedback "The login still fails but now with a 401 instead of 500"

# When you notice partial success
/fantasia:feedback "Works for email login but SSO users still can't get in"

# Force a specific path
/fantasia:feedback "I think we're looking at the wrong code entirely" --reinvestigate

# When you have new information
/fantasia:feedback "I found out this only happens for users created before 2024"
```
