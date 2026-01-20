# Fantasia Configuration

Fantasia uses a two-tier configuration system:
- **Organization context** (`~/.claude/fantasia/org-context.md`): Shared across all projects
- **Project config** (`~/.claude/fantasia/<project>/fantasia-config.md`): Project-specific settings

Run `/fantasia:setup` to configure both interactively.

## Organization Context

The organization context file stores patterns, integrations, and standards shared across all your projects.

**Location**: `~/.claude/fantasia/org-context.md`

```yaml
---
organization: Acme Corp
created: 2026-01-20

repository_patterns:
  - name: Frontend Monorepo
    detection:
      - turbo.json
      - client/
    build_system: Turbo + Yarn
    languages: [TypeScript, React]
    commands:
      test: turbo test --filter=<package>
      lint: yarn lint <file>
      typecheck: yarn tsc:b <project>
      format: yarn format <file>
    precommit:
      enabled: true
      setup: pre-commit install
      run: pre-commit run --files <files>
      notes: Runs automatically on commit
    environment:
      activation: null
      notes: null
    notes: |
      Use CSS Modules, not styled-components
      Test observable behavior

  - name: Python Backend
    detection:
      - manage.py
      - .venv/
    build_system: Django
    languages: [Python]
    commands:
      test: bin/pytest <filepath>
      lint: pre-commit run --files <filepath>
      typecheck: source .venv/bin/activate && mypy <filepath>
      format: pre-commit run --files <filepath>
    precommit:
      enabled: true
      setup: pre-commit install
      run: pre-commit run --files <filepath>
    environment:
      activation: source .venv/bin/activate
      notes: CRITICAL - activate venv before mypy

integrations:
  jira:
    domain: yourcompany.atlassian.net
    project_keys: [PROJ, TEAM, INFRA]
  linear:
    teams: [engineering, platform]
  sentry:
    org: acme-corp
  github:
    org: acme-corp

coding_standards:
  - Use TypeScript strict mode
  - Prefer CSS Modules over styled-components
  - Test observable behavior, not implementation
  - Always handle errors explicitly
---

# Additional Notes

Any freeform notes that don't fit the YAML structure.
```

### How It's Used

When Fantasia commands run, they:
1. Load `org-context.md` if it exists
2. Match the current repo against `repository_patterns` using detection markers
3. Inject matched pattern context into agent prompts
4. Use `integrations` for ticket/Jira/Sentry commands
5. Include `coding_standards` in planning and review

### Editing Organization Context

```bash
/fantasia:setup --org    # Re-run org setup interactively
```

Or edit `~/.claude/fantasia/org-context.md` directly.

## Custom Output Location

Override the default location with the `FANTASIA_DIR` environment variable:

```bash
# Use a custom location
export FANTASIA_DIR=/path/to/custom/location

# Or store in the repo (if you want to commit Fantasia outputs)
export FANTASIA_DIR=.fantasia
```

This is useful for:
- Team sharing (point to a shared location)
- Repo-local storage (commit maps with the code)
- Custom organization of multiple projects

## Configuration File

Create `~/.claude/fantasia/<project>/fantasia-config.md` with YAML frontmatter:

```bash
# Find your Fantasia directory
REPO_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
PROJECT_SLUG=$(basename "$REPO_ROOT" | tr '[:upper:]' '[:lower:]' | tr ' ' '-')
echo "Create config at: ~/.claude/fantasia/$PROJECT_SLUG/fantasia-config.md"
```

```markdown
---
# Model overrides (haiku, sonnet, opus)
models:
  # Mappers (default: haiku for tech/quality, sonnet for arch/concerns)
  tech-mapper: haiku
  arch-mapper: sonnet
  quality-mapper: haiku
  concerns-mapper: sonnet

  # Builders (default: opus for architect, sonnet for others)
  architect: opus
  implementer: sonnet
  integrator: sonnet

  # Reviewers (default: haiku for test, sonnet for others)
  code-reviewer: sonnet
  bug-hunter: sonnet
  test-reviewer: haiku

  # Specialized (default: sonnet)
  refactor-analyzer: sonnet
  bug-investigator: sonnet

# Default model for any agent not specified above
default-model: sonnet

# Workflow settings
workflow:
  # Auto-run review after build completes
  auto-review: false

  # Skip mapping if maps exist and are < N days old
  map-cache-days: 7

  # Default to worktree mode for tickets
  default-worktree: false
---

# Project Notes

Add any project-specific notes here that should be considered during planning/building.

## Architecture Decisions
- ...

## Patterns to Follow
- ...

## Things to Avoid
- ...
```

## Example Configurations

### Cost-Conscious (all haiku)
```yaml
models:
  architect: sonnet  # keep some quality for design
default-model: haiku
```

### Quality-First (upgrade mappers)
```yaml
models:
  tech-mapper: sonnet
  quality-mapper: sonnet
  arch-mapper: opus
default-model: sonnet
```

### Balanced (default)
No config file needed - uses built-in defaults.

## How It Works

1. Commands check for `~/.claude/fantasia/<project>/fantasia-config.md` at startup
2. If found, YAML frontmatter is parsed for settings
3. Model overrides are passed to subagent spawning
4. Markdown content below frontmatter is included as project context

## Precedence

1. Command-line flags (highest)
2. `~/.claude/fantasia/<project>/fantasia-config.md`
3. Built-in defaults (lowest)
