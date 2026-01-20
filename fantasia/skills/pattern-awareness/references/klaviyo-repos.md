# Klaviyo Repository Reference

This document helps Fantasia understand Klaviyo's main repositories and their patterns.

## Repository Detection

Identify which Klaviyo repo you're in by checking for these markers:

| Repo | Detection Markers |
|------|-------------------|
| **k-repo** | `pants.toml`, `python/klaviyo/`, `go/`, BUILD files |
| **fender** | `turbo.json`, `client/`, `@klaviyo/*` packages, `entrypoints/` |
| **app** | `.venv/`, `manage.py`, Django imports, `src/` with Django structure |

## k-repo (Python/Go Monorepo)

### Tech Stack
- **Build System**: Pants
- **Languages**: Python, Go
- **Structure**: `python/klaviyo/<package>/`, `go/<package>/`

### Required Commands Before Push
```bash
pants --no-dynamic-ui test <target>    # Testing
pants --no-dynamic-ui lint <target>    # Linting
pants --no-dynamic-ui check <target>   # Type checking
pants fmt <target>                      # Formatting
```

### Target Syntax
- Specific file: `python/klaviyo/package/file.py`
- All in directory: `python/klaviyo/package/tests/unit/::`
- Entire package: `python/klaviyo/package::`

### Patterns to Document
- Pants BUILD file configurations
- Python type hint conventions
- Go module organization
- Test fixtures and patterns

## fender (Frontend Monorepo)

### Tech Stack
- **Build System**: Turbo + Yarn workspaces
- **Languages**: TypeScript, React
- **Styling**: CSS Modules with theme tokens

### Structure
```
fender/
├── client/
│   ├── app/              # Domain-specific packages
│   ├── shared/           # Shared frontend packages
│   └── onsite/           # Customer-facing packages
├── universal/packages/   # Isomorphic packages
├── backend/              # Backend scripts, eslint plugin
├── entrypoints/packages/ # Built applications
└── cypress/              # E2E tests
```

### Required Commands Before Push
```bash
turbo test --filter="@klaviyo/package"           # Testing
NODE_OPTIONS="--max-old-space-size=6000" yarn lint <file>  # Linting
yarn tsc:b <file or project>                     # Type checking
yarn format <file>                               # Formatting
```

### Key Packages
- `@klaviyo/component-library` - Shared UI components
- `@klaviyo/api-hooks` - React Query hooks
- `@klaviyo/api` - API request methods
- `@klaviyo/api-types` - API TypeScript types
- `@klaviyo/i18n` - Internationalization
- `@klaviyo/navigation` - Routing
- `@klaviyo/theme` - Design tokens
- `@klaviyo/testing-react-utils` - Test utilities

### Patterns to Document
- CSS Modules patterns (no styled-components)
- React component structure
- API hooks patterns
- Testing philosophy (observable behavior, not implementation)
- Import organization (@klaviyo/* imports)

## app (Django Backend)

### Tech Stack
- **Framework**: Django
- **Language**: Python
- **Environment**: Virtual environment at `.venv`

### CRITICAL: Virtual Environment
Many commands require the venv to be activated:
```bash
source .venv/bin/activate
```

### Required Commands Before Push
```bash
pre-commit run --files <filepath>              # Linting + formatting
source .venv/bin/activate && mypy <filepath>   # Type checking
```

### Testing
```bash
bin/pytest <filepath>                    # Direct pytest
make unit TEST=<filepath>                # Unit tests
make test TEST=<filepath>                # Integration tests
```

### Patterns to Document
- Django model patterns
- View/serializer patterns
- Type hints (PEP 484)
- Test fixture patterns
- Migration conventions

## Cross-Repo Patterns

### External Dependencies (EXTERNAL-DEPS.md)
When working in fender and needing backend APIs, document:
- API endpoint specifications
- Request/response formats
- Authentication requirements

When working in app and needing frontend types, document:
- TypeScript type definitions
- API contract expectations

### Shared Conventions
- All repos use type hints/type safety
- All repos have required checks before push
- All repos follow linting/formatting standards
- Look for existing CLAUDE.md files for package-specific guidance

## Integration with klaviyo-repos-plugin

If the `klaviyo-repos-plugin` is installed, Fantasia should:
1. Defer to `/k-repo`, `/fender`, `/app` skills for command reference
2. Use Fantasia for architecture/pattern mapping that complements those skills
3. Leverage `repo-discovery` for package-specific caching

Fantasia adds value by:
- Deep architecture analysis beyond command reference
- Pattern documentation for consistency
- Cross-repo dependency tracking
- Technical debt identification specific to Klaviyo patterns
