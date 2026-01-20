---
description: Analyze codebase with parallel mapper agents to produce codebase maps
argument-hint: "[focus-area]"
allowed-tools: ["Read", "Bash", "Glob", "Grep", "Write", "Task"]
---

You are orchestrating a codebase mapping operation for the Fantasia plugin. Your job is to spawn parallel mapper agents that analyze the codebase and write documentation.

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

All paths below use `$FANTASIA_DIR` instead of `.claude/`:
- Maps: `$FANTASIA_DIR/codebase/`
- Plans: `$FANTASIA_DIR/plans/`
- State: `$FANTASIA_DIR/fantasia-state.md`
- Config: `$FANTASIA_DIR/fantasia-config.md`

## Prerequisites Check

1. Check if `$FANTASIA_DIR/codebase/` already exists
2. If it exists, warn the user: "Codebase maps already exist. Running this will overwrite them. Continue? (y/n)"
3. Wait for confirmation before proceeding

## Setup

Create the output directory:
```bash
mkdir -p "$FANTASIA_DIR/codebase"
```

## Load Configuration

Check for project-specific configuration:

```bash
CONFIG_FILE="$FANTASIA_DIR/fantasia-config.md"
if [ -f "$CONFIG_FILE" ]; then
  echo "CONFIG_EXISTS=true"
fi
```

If config exists, read it and extract model overrides from YAML frontmatter:
- Look for `models:` section
- Extract agent-specific model settings (e.g., `tech-mapper: haiku`)
- Extract `default-model:` if set
- Also read any project notes in the markdown body for context

**Model override logic**: When spawning each agent, use:
1. Config-specified model for that agent (if set)
2. Config `default-model` (if set)
3. Agent's built-in default (from agent file)

## Klaviyo Repository Detection

Before spawning agents, detect if this is a Klaviyo repository:

```bash
# Check for k-repo markers
if [ -f "pants.toml" ] || [ -d "python/klaviyo" ]; then
  echo "KLAVIYO_REPO=k-repo"
fi

# Check for fender markers
if [ -f "turbo.json" ] && [ -d "client" ]; then
  echo "KLAVIYO_REPO=fender"
fi

# Check for app markers
if [ -d ".venv" ] && [ -f "manage.py" ]; then
  echo "KLAVIYO_REPO=app"
fi
```

If a Klaviyo repo is detected, add these special instructions to EACH agent prompt:

### For k-repo:
```
KLAVIYO CONTEXT: This is Klaviyo's k-repo (Pants monorepo).
- Build system: Pants
- Languages: Python, Go
- Required commands: pants test/lint/check/fmt
- Document Pants target syntax in CONVENTIONS.md
- Note BUILD file patterns in STRUCTURE.md
- Include required checks before push in TESTING.md
```

### For fender:
```
KLAVIYO CONTEXT: This is Klaviyo's fender (frontend monorepo).
- Build system: Turbo + Yarn workspaces
- Languages: TypeScript, React
- Required commands: turbo test, yarn lint/tsc:b/format
- Document @klaviyo/* package patterns in ARCHITECTURE.md
- Note CSS Modules usage (NOT styled-components) in CONVENTIONS.md
- Include testing philosophy (observable behavior) in TESTING.md
- Document client/shared/universal structure in STRUCTURE.md
```

### For app:
```
KLAVIYO CONTEXT: This is Klaviyo's app (Django backend).
- Framework: Django
- Language: Python
- CRITICAL: .venv must be activated for mypy
- Required commands: pre-commit, mypy (with venv)
- Document Django patterns in ARCHITECTURE.md
- Note venv requirement prominently in CONVENTIONS.md
- Include pre-commit hooks in TESTING.md
```

## Optional Focus Area

If the user provided a focus area argument (e.g., `/fantasia:map auth`), inform each agent to pay special attention to that area while still covering their domain.

Focus area provided: `$ARGUMENTS`

## Spawn Parallel Mapper Agents

Use the Task tool to spawn 4 mapper agents in parallel. Each agent writes directly to files and returns only a confirmation.

**CRITICAL FOR TOKEN EFFICIENCY:**
- Each agent writes their output directly to files
- Agents return ONLY a brief confirmation (e.g., "✓ STACK.md and INTEGRATIONS.md written")
- Do NOT ask agents to return file contents to you

**MODEL SELECTION:**
When spawning each agent via the Task tool, set the `model` parameter based on:
1. If `$FANTASIA_DIR/fantasia-config.md` exists and specifies a model for this agent → use that
2. If config specifies `default-model` → use that
3. Otherwise use the agent's built-in default:
   - tech-mapper: haiku
   - arch-mapper: sonnet
   - quality-mapper: haiku
   - concerns-mapper: sonnet

Example Task call with model override:
```
Task(subagent_type="tech-mapper", model="haiku", prompt="...")
```

### Agent 1: tech-mapper
```
Analyze this codebase's technology stack and integrations.

Focus area (if any): $ARGUMENTS

Write these files directly:

1. `$FANTASIA_DIR/codebase/STACK.md` - Technologies used:
   - Languages and versions
   - Frameworks and libraries (with versions from package files)
   - Build tools and bundlers
   - Development dependencies
   - Runtime requirements

2. `$FANTASIA_DIR/codebase/INTEGRATIONS.md` - External integrations:
   - APIs consumed
   - Databases and data stores
   - Third-party services
   - Authentication providers
   - Monitoring/logging services

Use Glob and Grep to find package.json, requirements.txt, go.mod, Cargo.toml, etc.
Read key configuration files to understand the stack.

When done, respond with ONLY: "✓ STACK.md and INTEGRATIONS.md written"
```

### Agent 2: arch-mapper
```
Analyze this codebase's architecture and structure.

Focus area (if any): $ARGUMENTS

Write these files directly:

1. `$FANTASIA_DIR/codebase/ARCHITECTURE.md` - System design:
   - High-level architecture pattern (monolith, microservices, etc.)
   - Key components and their responsibilities
   - Data flow between components
   - Entry points (API routes, CLI commands, etc.)
   - Dependency injection / service patterns

2. `$FANTASIA_DIR/codebase/STRUCTURE.md` - Code organization:
   - Directory structure explanation
   - Module/package organization
   - Naming conventions observed
   - Key files and their purposes
   - Where different concerns live (models, views, controllers, etc.)

Use Glob to understand directory structure.
Read key files to understand architecture patterns.

When done, respond with ONLY: "✓ ARCHITECTURE.md and STRUCTURE.md written"
```

### Agent 3: quality-mapper
```
Analyze this codebase's conventions and testing approaches.

Focus area (if any): $ARGUMENTS

Write these files directly:

1. `$FANTASIA_DIR/codebase/CONVENTIONS.md` - Coding patterns:
   - Code style (formatting, naming)
   - Common patterns used (factories, repositories, hooks, etc.)
   - Error handling patterns
   - Logging conventions
   - Comment/documentation style
   - Import organization

2. `$FANTASIA_DIR/codebase/TESTING.md` - Test strategy:
   - Testing frameworks used
   - Test file organization
   - Test naming conventions
   - Mocking/stubbing patterns
   - Test coverage approach
   - Integration vs unit test separation

Use Grep to find patterns across the codebase.
Read test files and configuration to understand testing approach.

When done, respond with ONLY: "✓ CONVENTIONS.md and TESTING.md written"
```

### Agent 4: concerns-mapper
```
Analyze this codebase for technical concerns and risks.

Focus area (if any): $ARGUMENTS

Write this file directly:

1. `$FANTASIA_DIR/codebase/CONCERNS.md` - Issues and risks:
   - Technical debt identified
   - Security considerations
   - Performance hotspots
   - Missing or outdated dependencies
   - Areas lacking tests
   - Code smells observed
   - Documentation gaps
   - Potential improvement areas

Be constructive, not critical. Frame concerns as opportunities.

Use Grep to find TODOs, FIXMEs, deprecated patterns.
Look for security anti-patterns, unhandled errors, etc.

When done, respond with ONLY: "✓ CONCERNS.md written"
```

## Completion

After all 4 agents complete, summarize:

```
✅ Codebase mapping complete!

Created in $FANTASIA_DIR/codebase/:
- STACK.md - Technologies and dependencies
- INTEGRATIONS.md - External services and APIs
- ARCHITECTURE.md - System design
- STRUCTURE.md - Code organization
- CONVENTIONS.md - Coding patterns
- TESTING.md - Test strategies
- CONCERNS.md - Technical debt and risks

Next step: Run /fantasia:plan <task> to plan a feature
```

### Klaviyo-Specific Completion

If a Klaviyo repo was detected, add to the summary:

**For k-repo:**
```
📦 Detected: Klaviyo k-repo (Pants monorepo)
Tip: Use /k-repo skill for command reference, Fantasia maps for patterns.
Required before push: pants test, lint, check, fmt
```

**For fender:**
```
📦 Detected: Klaviyo fender (frontend monorepo)
Tip: Use /fender skill for commands, Fantasia maps for architecture.
Required before push: turbo test, yarn lint, tsc:b, format
```

**For app:**
```
📦 Detected: Klaviyo app (Django backend)
Tip: Use /app skill for commands, Fantasia maps for Django patterns.
Required before push: pre-commit, mypy (remember to activate .venv!)
```

## Save Checkpoint

After mapping completes, save state for context preservation:

```bash
# Preserve mode from previous state if it exists (handles Mode: and **Mode**:)
CURRENT_MODE=$(sed -n -E 's/^[[:space:]]*([*][*])?Mode([*][*])?:[[:space:]]*//p' $FANTASIA_DIR/fantasia-state.md 2>/dev/null | head -1)
if [ -z "$CURRENT_MODE" ]; then
  CURRENT_MODE="interactive"
fi

cat > $FANTASIA_DIR/fantasia-state.md << EOF
# Fantasia Checkpoint

**Saved**: $(date -Iseconds)
**Phase**: idle
**Last Action**: mapping complete
**Mode**: $CURRENT_MODE

## Context
- Maps: STACK.md, INTEGRATIONS.md, ARCHITECTURE.md, STRUCTURE.md, CONVENTIONS.md, TESTING.md, CONCERNS.md
- Plan: none
- Build: not started
- Review: not started

## To Resume
Run \`/fantasia:resume\` or \`/fantasia:plan <task>\` to continue.
EOF
```

This checkpoint will be injected into context during compaction via the PreCompact hook.
