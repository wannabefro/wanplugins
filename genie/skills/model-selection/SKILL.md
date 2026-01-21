---
name: model-selection
description: |
  Use this skill when analyzing a prompt or task to determine the optimal Claude model
  (haiku, sonnet, or opus). Triggers when needing to route tasks, select models for
  subagents, or optimize cost/quality tradeoffs.
---

# Model Selection Criteria

Analyze the task and select the optimal model using these criteria.

## Model Tiers

### Haiku - Fast & Efficient
**Select haiku when the task involves:**
- Simple lookups or searches
- Pattern matching across files
- Scanning for specific strings/patterns
- Quick questions with factual answers
- Listing files, directories, or contents
- Basic formatting or transformation
- Summarizing without deep analysis
- Validation checks (syntax, schema)

**Indicators in prompt:**
- "find", "search", "list", "scan", "check"
- "what is", "where is", "show me"
- Short, direct questions
- File/directory operations
- Simple transformations

### Sonnet - Balanced & Capable
**Select sonnet when the task involves:**
- Writing or modifying code
- Implementation of features
- Bug investigation and fixing
- Nuanced reasoning about tradeoffs
- Code review and analysis
- Test writing
- Documentation with context
- Multi-step reasoning
- Refactoring existing code

**Indicators in prompt:**
- "implement", "create", "build", "write"
- "fix", "debug", "investigate"
- "refactor", "improve", "optimize"
- "review", "analyze"
- Code snippets or file references
- Feature descriptions

### Opus - Deep & Complex
**Select opus when the task involves:**
- Architectural design decisions
- Complex system design
- Novel algorithm development
- Deep analysis requiring creativity
- Strategic planning
- Decisions with significant tradeoffs
- Problems requiring multiple expert perspectives
- Tasks where correctness is critical and complex

**Indicators in prompt:**
- "design", "architect", "plan"
- "complex", "tradeoffs", "strategy"
- "novel", "creative", "innovative"
- System-wide implications
- Abstract reasoning required
- Multiple interacting components

## Decision Algorithm

1. **Check for opus indicators first** - If task involves architecture, design, or complex reasoning → opus
2. **Check for haiku indicators** - If task is simple lookup, scan, or factual → haiku
3. **Default to sonnet** - Most coding tasks benefit from sonnet's balance

## Cost-Quality Tradeoff

| Model | Speed | Cost | Quality |
|-------|-------|------|---------|
| Haiku | ★★★★★ | ★★★★★ | ★★★ |
| Sonnet | ★★★ | ★★★ | ★★★★ |
| Opus | ★★ | ★ | ★★★★★ |

**When in doubt:** Prefer sonnet. It handles most tasks well and the cost/quality tradeoff is optimal for general development work.

## Output Format

After analysis, report:
```
✨ <Model> - <Brief reason>
```

Examples:
- `✨ Haiku - Simple file search task`
- `✨ Sonnet - Code implementation with moderate complexity`
- `✨ Opus - Complex architectural design requiring deep analysis`
