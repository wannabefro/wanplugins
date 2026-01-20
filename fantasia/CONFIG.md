# Fantasia Configuration

Fantasia stores all outputs in `~/.claude/fantasia/<project>/` by default to keep your repos clean. This includes configuration.

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
