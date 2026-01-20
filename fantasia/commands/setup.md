---
description: Configure Fantasia for your organization and project - auto-detects patterns and guides you through setup
argument-hint: "[--org]"
allowed-tools: ["Read", "Bash", "Glob", "Grep", "Write", "AskUserQuestion"]
---

You are running Fantasia setup. Your job is to configure organization-wide context and project-specific settings through a hybrid auto-detect + guided questions approach.

## Determine Fantasia Directory

```bash
# Use environment variable if set, otherwise calculate default
if [ -z "$FANTASIA_DIR" ]; then
  REPO_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
  PROJECT_SLUG=$(basename "$REPO_ROOT" | tr '[:upper:]' '[:lower:]' | tr ' ' '-')
  FANTASIA_DIR="$HOME/.claude/fantasia/$PROJECT_SLUG"
fi

ORG_CONTEXT_FILE="$HOME/.claude/fantasia/org-context.md"

echo "FANTASIA_DIR=$FANTASIA_DIR"
echo "ORG_CONTEXT_FILE=$ORG_CONTEXT_FILE"
```

## Parse Arguments

```bash
ARGS="$ARGUMENTS"
ORG_ONLY=false

if echo "$ARGS" | grep -q "\-\-org"; then
  ORG_ONLY=true
fi
```

## Step 1: Check Organization Context

```bash
if [ -f "$HOME/.claude/fantasia/org-context.md" ]; then
  echo "ORG_CONTEXT_EXISTS=true"
else
  echo "ORG_CONTEXT_EXISTS=false"
fi
```

### If org context doesn't exist:

Ask the user:
```
No organization context found at ~/.claude/fantasia/org-context.md

Would you like to set up organization-wide context first?
This captures repository patterns, integrations, and coding standards
that apply across all your projects.

Options:
1. Yes, set up organization context (recommended for first-time setup)
2. No, just configure this project
```

If they choose option 1, proceed to Step 2.
If they choose option 2, skip to Step 3.

### If org context exists and --org flag not set:

```
✓ Organization context loaded from ~/.claude/fantasia/org-context.md

Proceeding to project setup...
```

Skip to Step 3.

### If --org flag is set:

Proceed to Step 2 (edit/recreate org context).

## Step 2: Organization Context Setup

### 2a. Organization Name

Ask: "What's your organization name?"

### 2b. Auto-Detect Current Repo as Template

Scan the current repository for patterns:

```bash
echo "=== Scanning repository for patterns ==="

# Check for various build systems/frameworks
[ -f "pants.toml" ] && echo "DETECTED: pants.toml (Pants build system)"
[ -f "BUILD" ] || [ -f "BUILD.bazel" ] && echo "DETECTED: BUILD files (Bazel/Pants)"
[ -f "turbo.json" ] && echo "DETECTED: turbo.json (Turborepo)"
[ -f "package.json" ] && echo "DETECTED: package.json (Node.js)"
[ -f "yarn.lock" ] && echo "DETECTED: yarn.lock (Yarn)"
[ -f "pnpm-lock.yaml" ] && echo "DETECTED: pnpm-lock.yaml (pnpm)"
[ -f "manage.py" ] && echo "DETECTED: manage.py (Django)"
[ -d ".venv" ] && echo "DETECTED: .venv/ (Python virtual environment)"
[ -f "pyproject.toml" ] && echo "DETECTED: pyproject.toml (Python project)"
[ -f "setup.py" ] && echo "DETECTED: setup.py (Python package)"
[ -f "Cargo.toml" ] && echo "DETECTED: Cargo.toml (Rust)"
[ -f "go.mod" ] && echo "DETECTED: go.mod (Go module)"
[ -f "Makefile" ] && echo "DETECTED: Makefile"
[ -f ".pre-commit-config.yaml" ] && echo "DETECTED: .pre-commit-config.yaml (pre-commit hooks)"
[ -f "docker-compose.yml" ] || [ -f "docker-compose.yaml" ] && echo "DETECTED: docker-compose (Docker)"
[ -f "Dockerfile" ] && echo "DETECTED: Dockerfile"

# Check for specific directory patterns
[ -d "src" ] && echo "DETECTED: src/ directory"
[ -d "client" ] && echo "DETECTED: client/ directory"
[ -d "server" ] && echo "DETECTED: server/ directory"
[ -d "packages" ] && echo "DETECTED: packages/ (monorepo)"
```

Present findings:
```
Detected in this repository:
- turbo.json (Turborepo)
- package.json (Node.js)
- yarn.lock (Yarn)
- .pre-commit-config.yaml (pre-commit hooks)
- client/ directory

Would you like to use this as a template for an organization repository pattern?
```

### 2c. Guided Questions for Repository Pattern

For each repository pattern, ask:

1. **Pattern name**: "What should we call this pattern? (e.g., 'Frontend Monorepo', 'Python Backend')"

2. **Detection markers**: "What files/directories identify this repo type?"
   - Pre-fill from auto-detection
   - Allow additions/removals

3. **Build system**: "What build system does this use?"
   - Options based on detection: Turbo, Pants, Make, Cargo, Go, npm, yarn, pnpm, pip, poetry, custom

4. **Languages**: "What languages are used?"
   - Options: TypeScript, JavaScript, Python, Go, Rust, Java, etc.
   - Allow multiple selection

5. **Commands** (for each, ask or auto-detect from package.json/Makefile/etc.):
   - "Command to run tests?"
   - "Command to run linting?"
   - "Command to run type checking?"
   - "Command to format code?"

6. **Pre-commit**:
   - If `.pre-commit-config.yaml` detected: "Pre-commit is configured. What's the setup command?" (default: `pre-commit install`)
   - "Command to run pre-commit manually?" (default: `pre-commit run --files <files>`)
   - "Any notes about pre-commit?"

7. **Environment**:
   - If `.venv` detected: "Virtual environment detected. What's the activation command?" (default: `source .venv/bin/activate`)
   - "Any commands that require special environment setup?"

8. **Additional notes**: "Any other important notes for this repo type?"

### 2d. Add More Patterns?

Ask: "Would you like to add another repository pattern? (y/n)"

If yes, repeat 2c for the new pattern.

### 2e. Integrations (Optional)

Ask: "Would you like to configure external integrations?"

If yes:
- **Jira**: "Jira domain? (e.g., yourcompany.atlassian.net)" + "Common project keys? (comma-separated)"
- **Linear**: "Linear team identifiers? (comma-separated)"
- **Sentry**: "Sentry organization slug?"
- **GitHub**: "GitHub organization?"

### 2f. Coding Standards (Optional)

Ask: "Would you like to add organization-wide coding standards?"

If yes, ask for free-form input:
```
Enter coding standards (one per line, empty line to finish):
Examples:
- Use TypeScript strict mode
- Prefer CSS Modules over styled-components
- Test observable behavior, not implementation
```

### 2g. Write Organization Context

Create the directory and write the file:

```bash
mkdir -p "$HOME/.claude/fantasia"
```

Write `~/.claude/fantasia/org-context.md`:

```markdown
---
organization: <org-name>
created: <timestamp>

repository_patterns:
  - name: <pattern-name>
    detection:
      - <marker1>
      - <marker2>
    build_system: <build-system>
    languages:
      - <lang1>
      - <lang2>
    commands:
      test: <test-command>
      lint: <lint-command>
      typecheck: <typecheck-command>
      format: <format-command>
    precommit:
      enabled: <true|false>
      setup: <setup-command>
      run: <run-command>
      notes: <notes>
    environment:
      activation: <activation-command>
      notes: <notes>
    notes: |
      <additional-notes>

integrations:
  jira:
    domain: <domain>
    project_keys:
      - <key1>
      - <key2>
  linear:
    teams:
      - <team1>
  sentry:
    org: <org-slug>
  github:
    org: <github-org>

coding_standards:
  - <standard1>
  - <standard2>
---

# Additional Notes

<any-freeform-notes>
```

Present:
```
✅ Organization context saved to ~/.claude/fantasia/org-context.md

Repository patterns: X configured
Integrations: Jira, Sentry (or "none")
Coding standards: Y defined
```

If `--org` flag was set, stop here. Otherwise continue to Step 3.

## Step 3: Project Setup

### 3a. Auto-Detect Project Type

Run the same detection as Step 2b for the current repository.

### 3b. Match Against Org Patterns

If org context exists, parse `repository_patterns` and match detection markers:

```bash
# For each pattern in org-context.md, check if detection markers match
# e.g., if pattern has detection: [turbo.json, client/]
# check if those files/dirs exist in current repo
```

Present match results:
```
🔍 Scanning repository...

Detected:
- turbo.json
- package.json
- yarn.lock
- client/
- .pre-commit-config.yaml

✓ Matched organization pattern: "Frontend Monorepo"

This pattern includes:
- Build system: Turbo + Yarn
- Languages: TypeScript, React
- Test: turbo test --filter=<package>
- Lint: yarn lint <file>
- Typecheck: yarn tsc:b <project>
- Format: yarn format <file>
- Pre-commit: enabled

Is this correct? (y/n/customize)
```

### 3c. Handle No Match

If no org pattern matches (or no org context exists):

```
🔍 Scanning repository...

Detected:
- pyproject.toml
- src/
- tests/

No matching organization pattern found.

Would you like to:
1. Configure this project manually
2. Add this as a new organization pattern (recommended if you have similar repos)
3. Skip project setup
```

If option 2, go back to Step 2c to add the pattern, then return here.

### 3d. Project-Specific Overrides

Ask: "Any project-specific overrides or additions to the pattern?"

Examples:
- Different test command for this project
- Additional environment variables
- Project-specific notes

### 3e. Model Configuration (Optional)

Ask: "Would you like to customize which AI models Fantasia uses? (y/n)"

If yes, present current defaults and allow overrides:
```
Default model assignments:
- Mappers (tech, quality): haiku
- Mappers (arch, concerns): sonnet
- Architect: opus
- Implementer, Integrator: sonnet
- Reviewers: sonnet (code, bug), haiku (test)

Enter overrides (or press enter to keep defaults):
- Default model for all agents: [haiku/sonnet/opus]
- Specific overrides: (e.g., "architect: sonnet")
```

### 3f. Write Project Config

Create directory and write config:

```bash
mkdir -p "$FANTASIA_DIR"
```

Write `$FANTASIA_DIR/fantasia-config.md`:

```markdown
---
project: <project-slug>
created: <timestamp>
matched_pattern: <pattern-name-or-null>

# Model overrides (optional)
models:
  default-model: <model>
  # agent-specific overrides...

# Workflow settings
workflow:
  auto-review: false
  map-cache-days: 7
  default-worktree: false

# Project-specific overrides to org pattern
overrides:
  commands:
    test: <override-if-different>
  notes: |
    <project-specific-notes>
---

# Project Notes

<any-project-specific-notes>
```

## Step 4: Summary

Present completion summary:

```
✅ Fantasia setup complete!

Organization Context:
  📁 ~/.claude/fantasia/org-context.md
  • Repository patterns: 2 configured
  • Integrations: Jira, Sentry
  • Coding standards: 5 defined

Project Configuration:
  📁 ~/.claude/fantasia/<project>/fantasia-config.md
  • Matched pattern: "Frontend Monorepo"
  • Model config: defaults
  • Overrides: none

Ready to use:
  /fantasia:map     → Analyze codebase (context will be applied)
  /fantasia:ticket  → Start from Jira/Linear ticket
  /fantasia:plan    → Plan a feature

To edit organization context later:
  /fantasia:setup --org
```

## Error Handling

### Permission errors
```
❌ Cannot write to ~/.claude/fantasia/

Please check directory permissions or set FANTASIA_DIR to a writable location:
  export FANTASIA_DIR=/path/to/writable/dir
```

### Invalid YAML in existing config
```
⚠️ Found existing org-context.md but YAML is invalid.

Options:
1. Back up and recreate
2. Show me the error so I can fix manually
3. Cancel setup
```
