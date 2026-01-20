# Fantasia Grounding Rules

Universal anti-hallucination and quality guardrails for all Fantasia agents.

## Core Principle: Cite Before You Write

**Every design decision, every pattern choice, every piece of code must be grounded in the actual codebase.**

```
WRONG: "I'll use a factory pattern here"
RIGHT: "I'll use a factory pattern like src/services/auth.service.ts:23 does"
```

## The Three Verification Rules

### Rule 1: Verify Existence

Before referencing ANY:
- File path → `ls` or `find` to confirm it exists
- Import → `grep` to confirm the export exists
- Function/method → Read the file to confirm signature
- Type/interface → Read the file to confirm definition

```bash
# Example: Before importing WidgetRepository
grep -r "export.*WidgetRepository" src/
```

### Rule 2: Cite Your Sources

Every pattern choice needs a citation:

```typescript
// Pattern source: src/services/user.service.ts:45-60
// Error handling: src/services/auth.service.ts:28
// Convention: CONVENTIONS.md line 78
export class WidgetService {
  // Using try/catch like src/services/auth.service.ts:28
  async create() {
    try { ... }
    catch (e) { ... }
  }
}
```

### Rule 3: Stop When Ungrounded

If you cannot find:
- 3+ similar examples for a pattern
- Documentation for an external API
- An existing convention to follow

**STOP** and report:

```
⚠️ GROUNDING FAILURE

Cannot find sufficient examples of [pattern/API/convention].
Found: [what you did find]
Need: [what you couldn't find]

Recommendation: [suggested action]
```

## Scope Boundaries

### What's In Scope

Only files explicitly listed in your task. Before modifying any file:

```
Is this file in my assigned scope?
├── YES → Proceed
└── NO  → STOP and skip
```

### What's Out of Scope

- Refactoring code you encounter (report it instead)
- Adding features not in the plan
- Fixing unrelated bugs
- "Improving" code you read

### Reporting Out-of-Scope Issues

```
⚠️ OUT OF SCOPE ISSUE

File: src/utils/helpers.ts:45
Issue: [what you found]
Impact: [why it matters]
Action: Report only - do not fix
```

## Anti-Hallucination Checklist

Before completing your task, verify:

- [ ] All file paths I referenced exist
- [ ] All imports I used exist at those paths
- [ ] All patterns I used have citations
- [ ] I stayed within my assigned scope
- [ ] I didn't invent any new patterns without examples
- [ ] I didn't guess at any external API behavior

## Common Hallucination Patterns to Avoid

### 1. Invented File Paths
```
WRONG: Create src/services/widget.service.ts (without checking if src/services/ exists)
RIGHT: ls src/ → Verify services/ exists → Create file
```

### 2. Assumed Imports
```
WRONG: import { validateRequest } from '../middleware/validation';
RIGHT: grep -r "export.*validateRequest" src/ → Find actual path → Import
```

### 3. Made-Up API Endpoints
```
WRONG: Client calls GET /api/v2/widgets/{id}
RIGHT: Check API docs/INTEGRATIONS.md → Verify endpoint exists → Use it
```

### 4. Invented Conventions
```
WRONG: "I'll use PascalCase for this because it's cleaner"
RIGHT: Check CONVENTIONS.md → Find naming convention → Follow it
```

### 5. Assumed Error Formats
```
WRONG: return { error: 'NOT_FOUND', code: 404 }
RIGHT: Find existing error responses → Match their format exactly
```

## Quality Assurance Questions

Ask yourself before completing any task:

1. **Could someone verify every claim I made by checking the codebase?**
2. **Did I copy patterns or invent them?**
3. **Would my code work if the codebase is exactly as documented?**
4. **Did I stay strictly within scope?**

If any answer is "no" or "unsure", revisit your work.

## Recovery from Hallucination

If you realize you've hallucinated:

1. **Stop immediately**
2. **Report what happened**:
   ```
   ⚠️ CORRECTION NEEDED

   I previously stated: [what you said]
   Actual state: [what's actually true]
   Action needed: [how to fix]
   ```
3. **Fix before proceeding**
4. **Re-verify your work**

## Model-Specific Guidance

### Opus (Architect)
- Higher risk of over-engineering
- Verify you're not inventing new patterns
- 3+ examples required for every design choice

### Sonnet (Implementer/Integrator)
- Balance between creativity and grounding
- Cite patterns but don't over-document
- Stop if contracts are unclear

### Haiku (Reviewers/Mappers)
- Lower hallucination risk but lower accuracy
- Double-check file paths before reporting
- Verify findings against actual code
