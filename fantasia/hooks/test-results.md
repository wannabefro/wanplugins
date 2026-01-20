---
name: test-results-parser
description: Parses test output and auto-triggers feedback loop on failures
event: PostToolUse
match_tool: Bash
match_pattern: "(pytest|npm test|yarn test|go test|cargo test|jest)"
---

# Test Results Parser Hook

When a test command completes, analyze the output and update state.

## Parse Test Output

Extract from the Bash output:
- Total tests run
- Passed/Failed/Skipped counts
- Failed test names and locations
- Coverage percentage (if present)

## Write Results

```bash
FANTASIA_DIR="${FANTASIA_DIR:-$HOME/.claude/fantasia/$(basename $(git rev-parse --show-toplevel))}"
TASK_SLUG=$(cat $FANTASIA_DIR/fantasia-state.md 2>/dev/null | grep -E "^\*\*Plan\*\*:" | sed 's/.*: //')

cat > "$FANTASIA_DIR/plans/$TASK_SLUG/TEST-RESULTS.json" << EOF
{
  "timestamp": "$(date -Iseconds)",
  "total": <count>,
  "passed": <count>,
  "failed": <count>,
  "skipped": <count>,
  "coverage": <percent or null>,
  "failures": [
    {"name": "<test>", "file": "<path:line>", "error": "<message>"}
  ]
}
EOF
```

## Auto-Feedback Trigger

If `failed > 0` AND current mode is `yolo`:
- Automatically inject failure context into next iteration
- No user prompt needed - continue fixing

If `failed > 0` AND current mode is `interactive`:
- Suggest: "Tests failed. Run `/fantasia:feedback` to iterate?"

## Token Efficiency

Return ONLY:
- "✓ Tests passed (X/Y)"
- "✗ Tests failed: X failures - see TEST-RESULTS.json"

Do NOT return full test output in hook response.
