---
name: pattern-awareness
description: Use when building features in a codebase that has Fantasia maps (~/.claude/fantasia/<project>/codebase/). This skill teaches how to read and apply mapped patterns when writing new code.
---

You have access to codebase maps created by Fantasia. Use them to write code that fits naturally into this codebase.

## Fantasia Directory

Fantasia stores all outputs in the home directory to keep repos clean. The location can be customized via the `FANTASIA_DIR` environment variable.

```bash
# Calculate the Fantasia directory (or use env override)
if [ -z "$FANTASIA_DIR" ]; then
  REPO_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
  PROJECT_SLUG=$(basename "$REPO_ROOT" | tr '[:upper:]' '[:lower:]' | tr ' ' '-')
  FANTASIA_DIR="$HOME/.claude/fantasia/$PROJECT_SLUG"
fi
```

All paths below use `$FANTASIA_DIR/codebase/` for maps.

## When to Use This

Use this skill when:
- You're writing new code and `$FANTASIA_DIR/codebase/` exists
- You need to understand project conventions
- You want to match existing patterns
- You're deciding how to structure new code

## How to Use the Maps

### Before Writing Code

1. **Check if maps exist**:
   ```bash
   ls $FANTASIA_DIR/codebase/
   ```

2. **Read relevant maps** based on what you're doing:

   | Task | Read These |
   |------|------------|
   | Any code change | CONVENTIONS.md |
   | New component/service | ARCHITECTURE.md, STRUCTURE.md |
   | External integration | INTEGRATIONS.md, STACK.md |
   | Tests | TESTING.md |
   | Fixing tech debt | CONCERNS.md |

### Applying Patterns

When you read a map file, look for:

1. **Naming conventions**: How things are named in this codebase
2. **File organization**: Where new code should live
3. **Error handling**: How errors are typically handled
4. **Logging**: How and when to log
5. **Type patterns**: How types/interfaces are defined
6. **Import style**: How imports are organized

### Example Workflow

```
User: Add a new UserPreferences service

You should:
1. Read ARCHITECTURE.md - understand service patterns
2. Read STRUCTURE.md - find where services live
3. Read CONVENTIONS.md - match naming/style
4. Find an existing service as reference
5. Write new code following the patterns
```

## Map File Reference

### STACK.md
Contains: Languages, frameworks, key dependencies
Use for: Understanding what tools/libraries to use

### ARCHITECTURE.md
Contains: System design, component responsibilities, data flow
Use for: Understanding where new code fits, how components interact

### STRUCTURE.md
Contains: Directory layout, file organization
Use for: Knowing where to put new files

### CONVENTIONS.md
Contains: Coding style, patterns, error handling
Use for: Matching code style exactly

### TESTING.md
Contains: Test framework, test organization, mocking patterns
Use for: Writing tests that match existing style

### INTEGRATIONS.md
Contains: External services, APIs, databases
Use for: Understanding how to connect to external systems

### CONCERNS.md
Contains: Technical debt, risks, improvement areas
Use for: Avoiding known issues, understanding fragile areas

## Important Rules

1. **Always read before writing**: Don't guess at patterns - read the maps
2. **Match exactly**: Your code should look like it was written by the same team
3. **Don't introduce new patterns**: Use what's already there unless asked to change
4. **Ask if unsure**: If maps don't cover your case, ask before inventing

---

## Klaviyo-Specific Context

If you're working in a Klaviyo repository, Fantasia has additional knowledge. See `references/klaviyo-repos.md` for details.

### Quick Repo Detection

| Marker | Repo | Key Commands |
|--------|------|--------------|
| `pants.toml`, BUILD files | **k-repo** | `pants test/lint/check/fmt` |
| `turbo.json`, `@klaviyo/*` | **fender** | `turbo test`, `yarn lint/tsc:b/format` |
| `.venv/`, `manage.py` | **app** | `pre-commit`, `mypy` (requires venv) |

### Klaviyo Patterns to Follow

**In k-repo:**
- Use Pants target syntax (`package::`)
- Follow Python type hints (PEP 484)
- Tests in `tests/unit/` and `tests/integration/`

**In fender:**
- Use CSS Modules (not styled-components)
- Follow `@klaviyo/*` import patterns
- Test observable behavior, not implementation
- Use theme tokens for styling

**In app:**
- Always activate `.venv` for mypy
- Use Django patterns for models/views
- Follow type hint conventions

### Integration with Other Plugins

If `klaviyo-repos-plugin` or `fender` plugin is installed:
- Use their skills (`/k-repo`, `/fender`, `/app`) for command reference
- Use Fantasia for deeper architecture/pattern analysis
- The plugins complement each other

### Required Checks Before Push

All Klaviyo repos require passing:
1. Tests
2. Linting
3. Type checking
4. Formatting

Document these in TESTING.md and CONVENTIONS.md during mapping.
