# Fantasia

Codebase-aware development accelerator. Like Mickey orchestrating enchanted brooms, Fantasia coordinates parallel agents to map, plan, build, and review your code.

> **Note**: All Fantasia outputs go to `~/.claude/fantasia/<project>/` by default to keep your repos clean. Nothing is checked into git. Override with `FANTASIA_DIR` env var if needed.

## Workflow

```
/fantasia:setup   → Configure for your repo (first time only)
        ↓
/fantasia:ticket  → Start from Linear/Jira ticket (optional entry point)
        ↓
/fantasia:map     → Understand codebase patterns & architecture
        ↓
/fantasia:plan    → Design with verification before building
        ↓
/fantasia:build   → Execute with parallel specialist agents
        ↓
/fantasia:review  → Quality check with parallel reviewers
        ↓
/fantasia:verify  → Record manual testing results (optional)
        ↓
/fantasia:ci      → Check CI status, auto-fix on failure (optional)

Alternative workflows:
/fantasia:refactor <target>              → Safe refactoring with behavior preservation
/fantasia:fix <issue> [--sentry]         → Bug investigation and fixing (Sentry integration)
```

**First time?**: `/fantasia:setup` to configure org patterns and project settings

**Quick start**: `/fantasia:ticket PROJ-123` or `/fantasia:map` then `/fantasia:plan <task>`

**Bug fixing**: `/fantasia:fix SENTRY-123` or `/fantasia:fix "users can't login" --sentry`

**Fix not working?**: `/fantasia:feedback "still failing with different error"`

**Refactoring**: `/fantasia:refactor src/auth/` or `/fantasia:refactor "UserService class"`

**YOLO mode**: `/fantasia:yolo PROJ-123 --worktree` — full automated workflow in isolated worktree

## Commands

### `/fantasia:setup [--org]`
Configures Fantasia for your organization and project. Uses a hybrid approach: auto-detects repository patterns, then asks targeted questions for anything it couldn't detect.

**Two-tier configuration:**
- **Organization context** (`~/.claude/fantasia/org-context.md`): Shared patterns, integrations, and coding standards that apply across all your projects
- **Project config** (`~/.claude/fantasia/<project>/fantasia-config.md`): Project-specific settings and overrides

**What it captures:**

| Category | Examples |
|----------|----------|
| **Repository patterns** | Build systems, required commands (test/lint/typecheck/format), pre-commit hooks, environment setup |
| **Integrations** | Jira domain & project keys, Linear teams, Sentry org, GitHub org |
| **Coding standards** | Style preferences, architectural patterns, things to avoid |

**Flow:**
1. Checks if org context exists → if not, offers to create it
2. Auto-detects repo markers (pants.toml, turbo.json, package.json, etc.)
3. Matches against org patterns or guides you through setup
4. Writes configuration files

**Flags:**
| Flag | Description |
|------|-------------|
| `--org` | Edit organization context only (skip project setup) |

**Examples:**
```bash
/fantasia:setup                # Full setup (org if needed + project)
/fantasia:setup --org          # Edit organization context only
```

**First-time setup** creates both org context and project config.
**Returning users** skip straight to project setup (org context already exists).

### `/fantasia:ticket <url-or-id> [flags]`
Starts a workflow from a Linear or Jira ticket:
- Fetches ticket details (title, description, acceptance criteria)
- Saves original requirements to `TICKET.md`
- Transitions into planning phase
- Tracks acceptance criteria through review

**Flags**:
| Flag | Description |
|------|-------------|
| `--jira` | Explicitly use Jira (for ambiguous IDs) |
| `--linear` | Explicitly use Linear (for ambiguous IDs) |
| `--worktree` | Create isolated git worktree for this ticket |
| `--yolo` | Run full workflow without confirmations |

**Examples**:
```bash
/fantasia:ticket PROJ-123 --jira           # Explicitly Jira
/fantasia:ticket TEAM-456 --linear         # Explicitly Linear
/fantasia:ticket PROJ-123 --jira --worktree --yolo  # Full combo
```

**Supported sources**:
- Linear: `https://linear.app/team/issue/TEAM-123` or `TEAM-123`
- Jira: `https://yourcompany.atlassian.net/browse/PROJ-456` or `PROJ-456`

Outputs to `~/.claude/fantasia/<project>/plans/<ticket-id>/`:
- `TICKET.md` - Original ticket content and requirements

### `/fantasia:yolo <task-or-ticket> [flags]`
Runs the **full workflow without confirmations**:
1. Setup (+ worktree if flagged)
2. Map (if not already mapped)
3. Plan (auto-approved)
4. Build (all phases)
5. Review (all agents)

Perfect for when you trust the process and want to go fast.

**Flags**: `--jira`, `--linear`, `--worktree`

```bash
# From a Jira ticket
/fantasia:yolo PROJ-123 --jira

# From a Linear ticket with worktree
/fantasia:yolo TEAM-456 --linear --worktree

# From a task description
/fantasia:yolo "add user auth" --worktree
```

### `/fantasia:worktree <subcommand>`
Manages git worktrees for isolated feature development:

| Subcommand | Description |
|------------|-------------|
| `list` | Show all Fantasia-managed worktrees |
| `switch <task>` | Switch to a task's worktree |
| `finish` | Merge current worktree branch and cleanup |
| `cleanup` | Remove worktrees for merged/deleted branches |

Worktrees are created with `--worktree` flag on `/fantasia:ticket` or `/fantasia:yolo`.

### `/fantasia:map [focus]`
Analyzes your codebase with 4 parallel mapper agents:
- **tech-mapper**: Stack, dependencies, integrations
- **arch-mapper**: Architecture, structure, patterns
- **quality-mapper**: Conventions, testing approaches
- **concerns-mapper**: Technical debt, risks, gaps

Outputs to `~/.claude/fantasia/<project>/codebase/`:
- `STACK.md` - Technologies and dependencies
- `INTEGRATIONS.md` - External services and APIs
- `ARCHITECTURE.md` - System design
- `STRUCTURE.md` - Code organization
- `CONVENTIONS.md` - Coding patterns
- `TESTING.md` - Test strategies
- `CONCERNS.md` - Issues and risks

Optional focus area: `/fantasia:map auth` for deep-dive on specific areas.

### `/fantasia:plan <task>`
Plans a feature with thorough understanding:
1. **Discovery** - Explores relevant code, asks clarifying questions
2. **Verify** - Confirms sufficient context before designing
3. **Approach Exploration** - Spawns 3 parallel approach-explorer agents to evaluate:
   - Simplest possible approach
   - Most maintainable approach
   - Minimal changes approach
4. **Design** - Synthesizes approaches, creates implementation plan
5. **Readiness** - Presents plan for approval before build

Outputs to `~/.claude/fantasia/<project>/plans/<task-slug>/`:
- `PLAN.md` - Implementation plan
- `EXTERNAL-DEPS.md` - Cross-repo dependencies (if any)

### `/fantasia:build [--task <name>]`
Executes plan with parallel specialist agents:
- **architect**: Design, contracts, interfaces
- **implementer**: Core logic (adapts to stack)
- **integrator**: APIs, boundaries, connections

Uses most recent plan by default, or specify with `--task`.

On completion, suggests: "Ready for review? Run /fantasia:review"

### `/fantasia:review [--task <name>]`
Reviews changes with 3 parallel agents:
- **code-reviewer**: Quality, patterns compliance
- **bug-hunter**: Bugs, edge cases, error handling
- **test-reviewer**: Coverage, test quality

Outputs to `~/.claude/fantasia/<project>/plans/<task>/REVIEW.md` with actionable findings.

### `/fantasia:refactor <target> [--yolo]`
Plans and executes safe refactoring with behavior preservation:
1. **Analysis** - Spawns 3 parallel agents (all haiku for speed):
   - **code-smell-detector**: Identifies specific code smells with severity
   - **dependency-mapper**: Maps incoming/outgoing dependencies, assesses blast radius
   - **test-coverage-checker**: Evaluates test coverage, recommends if tests needed first
2. **Planning** - Synthesizes analysis, creates step-by-step refactoring plan
3. **Execution** - Makes incremental changes, verifying tests pass after each step
4. **Verification** - Uses parallel-verifier to run tests/types/lint simultaneously
5. **Review** - Runs code reviewer to verify quality

**Flags**:
| Flag | Description |
|------|-------------|
| `--yolo` | Execute without confirmations |

**Examples**:
```bash
/fantasia:refactor src/services/auth/       # Refactor a directory
/fantasia:refactor "UserService class"      # Refactor by description
/fantasia:refactor utils.ts --yolo          # Auto-mode
```

Outputs to `~/.claude/fantasia/<project>/plans/refactor-<slug>/`:
- `ANALYSIS.md` - Code smell detection, risk assessment
- `PLAN.md` - Step-by-step refactoring plan
- `SUMMARY.md` - Results and follow-up recommendations

### `/fantasia:fix <issue> [flags]`
Investigates and fixes bugs with optional Sentry integration:
1. **Gathering** - Fetches Sentry data or captures bug report
2. **Investigation** - Spawns 3 parallel agents to analyze the bug:
   - **stack-tracer**: Traces execution path, finds bug origin (requires file:line evidence)
   - **git-historian**: Finds when code was introduced, detects regressions
   - **similar-bug-finder**: Searches for related patterns and previous fixes
3. **Skeptic Review** - Spawns **bug-skeptic** to challenge findings:
   - Audits evidence quality (are claims backed by specific citations?)
   - Detects contradictions between investigators
   - Generates alternative explanations
   - Recommends verification steps
4. **Synthesis** - Consolidates findings with contradiction detection and confidence scoring
5. **Planning** - Designs minimal fix (only if confidence is Medium/High)
6. **Implementation** - Writes regression test first, then fixes
7. **Verification** - Uses parallel-verifier to run tests/types/lint simultaneously
8. **Feedback Loop** - If verification fails, iterates based on observations

**Flags**:
| Flag | Description |
|------|-------------|
| `--sentry` | Explicitly use Sentry (for ambiguous inputs) |
| `--yolo` | Execute without confirmations |

**Examples**:
```bash
/fantasia:fix PROJ-123 --sentry             # Fix from Sentry issue ID
/fantasia:fix https://sentry.io/...         # Fix from Sentry URL
/fantasia:fix "login fails for SSO users"   # Fix from description
/fantasia:fix PROJ-123 --sentry --yolo      # Auto-mode
```

**Sentry Integration**:
- Fetches error details, stack traces, breadcrumbs
- Shows frequency and user impact
- Reminds to mark issue resolved after merge

Outputs to `~/.claude/fantasia/<project>/plans/fix-<slug>/`:
- `SENTRY.md` - Sentry issue details (if applicable)
- `BUG-REPORT.md` - Bug report (if manual)
- `INVESTIGATION.md` - Root cause analysis
- `PLAN.md` - Fix plan
- `SUMMARY.md` - Results

### `/fantasia:feedback <observation> [flags]`
Provides feedback when a fix didn't work, enabling iterative refinement:
1. **Loads context** - Finds the most recent fix and its state
2. **Asks clarifying questions** - Understands what you observed
3. **Decides path** - Re-investigate (root cause wrong) or iterate (fix insufficient)
4. **Continues workflow** - Loops back to the appropriate phase

**Flags**:
| Flag | Description |
|------|-------------|
| `--reinvestigate` | Force re-investigation (skip decision logic) |
| `--iterate` | Force iteration on fix (skip decision logic) |

**Examples**:
```bash
/fantasia:feedback "The login still fails but with a 401 instead of 500"
/fantasia:feedback "Works for email login but SSO is still broken"
/fantasia:feedback "I think we're looking at the wrong code" --reinvestigate
```

**Soft limit**: After 3 failed attempts, Fantasia warns and offers options:
- Continue trying
- Re-map the relevant code area
- Pause and discuss with a teammate

This command integrates with `/fantasia:fix` - verification failures can automatically trigger the feedback flow, or you can invoke it manually anytime.

### `/fantasia:status`
Shows current workflow state:
- Which phase you're in
- What maps/plans exist
- What's been completed
- Suggested next steps

### `/fantasia:resume [--yolo]`
Restores context from saved checkpoint:
- Loads `~/.claude/fantasia/<project>/fantasia-state.md`
- Reads relevant maps and plans
- Continues from where you left off
- Suggests next action based on saved state

**Flag**: `--yolo` - Continue in YOLO mode (skip confirmations)

### `/fantasia:checkpoint`
Manually saves current workflow state:
- Captures phase, progress, and context
- Writes to `~/.claude/fantasia/<project>/fantasia-state.md`
- Useful for pausing work mid-session

### `/fantasia:verify [pass|fail] [notes]`
Records manual testing results:
- Tracks what was manually tested and outcome
- Appends to `MANUAL-VERIFICATION.md` in task directory
- Updates review status with verification evidence

### `/fantasia:ci [--wait] [--fix-on-fail]`
Checks CI/CD pipeline status for current branch:
- Uses `gh` CLI to query GitHub Actions
- `--wait` - Poll until CI completes (up to 30 min)
- `--fix-on-fail` - Auto-trigger `/fantasia:fix` if CI fails

**Requires**: GitHub CLI (`gh`) installed and authenticated

## Context Management

Fantasia automatically preserves context across sessions and compactions:

### Automatic Checkpoints
Each main command saves state after completion:
- `/fantasia:map` → Saves after maps written
- `/fantasia:plan` → Saves after plan approved
- `/fantasia:build` → Saves after each phase + completion
- `/fantasia:review` → Saves after review complete

### State File
Context stored in `~/.claude/fantasia/<project>/fantasia-state.md`:
```markdown
# Fantasia Checkpoint

**Saved**: <timestamp>
**Phase**: idle | mapping | planning | building | reviewing
**Plan**: <task-slug>

## Context
- Maps: available in ~/.claude/fantasia/<project>/codebase/
- Plan: ~/.claude/fantasia/<project>/plans/<task>/PLAN.md
- Build: complete | in-progress | not started
- Review: complete | not started

## Next Step
<Suggested action>
```

### Context Recovery
When context is compacted (long sessions), a PreCompact hook:
1. Reads `~/.claude/fantasia/<project>/fantasia-state.md`
2. Summarizes map headings from `~/.claude/fantasia/<project>/codebase/`
3. Injects essential context into the summary

This ensures you never lose your place in the workflow.

### Recovery Commands
- Start fresh session → Run `/fantasia:resume`
- Check where you are → Run `/fantasia:status`
- Save before leaving → Run `/fantasia:checkpoint`

## Git Worktrees

Fantasia supports isolated feature development with git worktrees:

### Why Worktrees?
- **Isolation**: Each ticket/task gets its own workspace
- **Parallel work**: Work on multiple tickets without stashing
- **Clean history**: One branch per feature
- **Safe experimentation**: Easy to abandon failed attempts

### Worktree Workflow
```bash
# Start ticket in worktree
/fantasia:ticket PROJ-123 --worktree
# Creates ../fantasia-proj-123/ with feat/proj-123 branch

# Work happens in isolated directory
cd ../fantasia-proj-123
/fantasia:build
/fantasia:review

# When done, merge and cleanup
/fantasia:worktree finish
# Merges to main, removes worktree
```

### Shared Data
All Fantasia data is in the home directory, so worktrees automatically share everything:
```
~/.claude/fantasia/<project>/
├── codebase/      ← Shared maps (all worktrees)
├── plans/         ← Shared plans (all tasks)
├── fantasia-state.md
└── worktrees.md   ← Worktree registry
```

No symlinks needed - worktrees naturally share the same Fantasia directory.

## YOLO Mode

For when you want to go from ticket to reviewed code without stopping:

```bash
/fantasia:yolo PROJ-123 --worktree
```

### What YOLO Does
1. **Setup**: Creates worktree (if flagged), fetches ticket
2. **Map**: Runs mappers if codebase maps don't exist
3. **Plan**: Auto-discovers and designs without asking questions
4. **Build**: Executes all phases without confirmation
5. **Review**: Runs all reviewers and compiles findings

### When to Use YOLO
- ✅ Well-defined tickets with clear acceptance criteria
- ✅ Familiar codebase patterns
- ✅ Time-sensitive work
- ✅ Exploratory/prototype work

### When NOT to Use YOLO
- ❌ Complex architectural decisions needed
- ❌ Ambiguous requirements
- ❌ First time in a new codebase
- ❌ Production-critical code

### Error Recovery
YOLO saves checkpoints at each phase. If it fails:
```bash
# Fix the issue, then resume
/fantasia:resume
```

## Token Efficiency

Fantasia is designed for minimal token usage:
- Agents write directly to files, return only confirmations
- Each agent receives only the context they need
- File-based handoffs between phases
- Lean orchestration

### Model Selection

Agents use different models based on task complexity:

| Model | Agents | Rationale |
|-------|--------|-----------|
| **haiku** | tech-mapper, quality-mapper, test-reviewer, code-smell-detector, dependency-mapper, test-coverage-checker, parallel-verifier | Fast scanning, pattern matching, straightforward analysis |
| **sonnet** | arch-mapper, concerns-mapper, implementer, integrator, code-reviewer, bug-hunter, approach-explorer, stack-tracer, git-historian, similar-bug-finder, bug-skeptic | Code writing, judgment calls, nuanced reasoning |
| **opus** | architect | Complex design decisions, interface contracts, architectural choices |

This tiered approach balances cost and quality - using heavyweight models only where reasoning depth matters.

### Custom Configuration

Override defaults per-project by creating `~/.claude/fantasia/<project>/fantasia-config.md`:

```yaml
---
models:
  architect: sonnet      # downgrade from opus
  tech-mapper: sonnet    # upgrade from haiku
default-model: sonnet    # fallback for unspecified agents

workflow:
  auto-review: false     # auto-run review after build
  map-cache-days: 7      # skip re-mapping if recent
  default-worktree: false
---

# Project-specific notes that agents will consider
```

See `CONFIG.md` in the plugin directory for full options.

## Prerequisites

Each command checks prerequisites and offers to run them:
- `build` requires: maps + approved plan
- `review` requires: recent build

## Output Structure

All outputs go to `~/.claude/fantasia/<project>/` by default to keep your repo clean:

```
~/.claude/fantasia/<project>/
├── fantasia-state.md   # Checkpoint for context recovery
├── fantasia-config.md  # Optional per-project configuration
├── worktrees.md        # Registry of active worktrees
├── codebase/           # From /fantasia:map
│   ├── STACK.md
│   ├── INTEGRATIONS.md
│   ├── ARCHITECTURE.md
│   ├── STRUCTURE.md
│   ├── CONVENTIONS.md
│   ├── TESTING.md
│   └── CONCERNS.md
└── plans/              # From /fantasia:plan, /fantasia:ticket, /fantasia:fix, /fantasia:refactor
    ├── <task-slug>/
    │   ├── TICKET.md       # From /fantasia:ticket (original requirements)
    │   ├── PLAN.md
    │   ├── EXTERNAL-DEPS.md
    │   └── REVIEW.md       # From /fantasia:review
    ├── fix-<slug>/         # From /fantasia:fix
    │   ├── SENTRY.md       # Sentry issue details (if applicable)
    │   ├── BUG-REPORT.md   # Manual bug report (if no Sentry)
    │   ├── INVESTIGATION.md
    │   ├── PLAN.md
    │   └── SUMMARY.md
    └── refactor-<slug>/    # From /fantasia:refactor
        ├── ANALYSIS.md
        ├── PLAN.md
        └── SUMMARY.md
```

The project slug is derived from your repo directory name (lowercase, hyphenated).

### Custom Output Location

Override the default location with the `FANTASIA_DIR` environment variable:

```bash
# Per-session override
export FANTASIA_DIR=/path/to/custom/location
claude

# Or inline
FANTASIA_DIR=.fantasia claude  # Store in repo (if you want to commit)
```

Use cases:
- **Team sharing**: Point to a shared location for collaborative mapping
- **Repo-local storage**: Set `FANTASIA_DIR=.fantasia` to store in the repo
- **Multiple projects**: Organize outputs differently than the default

