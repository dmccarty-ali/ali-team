# CLAUDE.md Files

## Hierarchy and Precedence (Highest to Lowest)

| Level | Location | Purpose |
|-------|----------|---------|
| Enterprise | /Library/Application Support/ClaudeCode/CLAUDE.md | Organization policies |
| Project Memory | ./CLAUDE.md or ./.claude/CLAUDE.md | Team standards (git) |
| Project Rules | ./.claude/rules/*.md | Path-specific rules |
| User Memory | ~/.claude/CLAUDE.md | Personal preferences |
| Local Project | ./CLAUDE.local.md | Personal, gitignored |

## What to Include

**Include:**
- Build commands (test, lint, run)
- Code standards unique to your project
- Patterns you want Claude to follow
- Files Claude should never modify
- Compliance requirements
- Technology-specific conventions

**Example:**
```markdown
## Build Commands
- Test: pytest tests/ -v
- Lint: ruff check src/
- Build: python -m build

## Code Standards
- Use 4-space indentation (not tabs)
- Type hints required for public functions
- Docstrings for modules and classes

## Patterns to Follow
- Repository pattern for data access
- Dependency injection via FastAPI Depends
- Async/await for I/O operations

## Files to Never Modify
- config/production.yaml
- .env files
```

## What to Exclude

**Exclude (Claude can figure out from code):**
- Basic language syntax
- Standard conventions (PEP 8, etc.)
- General explanations of frameworks
- Detailed API documentation (link instead)
- Frequently changing information
- Things obvious from reading code

**Example (what NOT to include):**
```markdown
❌ "Python functions are defined with 'def' keyword"
❌ "FastAPI is a web framework for building APIs"
❌ "Full SQLAlchemy ORM documentation: [20 pages]"
❌ "Current database host: db-prod-42.example.com"  # Changes often
```

## Pruning Recommendations

**CLAUDE.md should be <200 lines.** If longer, prune:

| Bloat Pattern | Fix |
|---------------|-----|
| Explaining standard tools | Remove (Claude knows) |
| Copy-paste from other projects | Remove non-applicable sections |
| Detailed API docs | Replace with links |
| Step-by-step tutorials | Replace with "Follow pattern in file:line" |
| Frequently changing info | Move to config files, reference those |

**Prune quarterly:**
1. Review CLAUDE.md for outdated info
2. Remove sections Claude doesn't need
3. Update patterns that have changed
4. Consolidate duplicate instructions

## Before/After Example

**BEFORE (Bloated, 450 lines):**
```markdown
# Project Instructions

## Python Basics
Python is a programming language. Functions are defined with def...
[30 lines of basic Python explanation]

## FastAPI Tutorial
FastAPI is a modern web framework. To create an endpoint...
[50 lines of FastAPI tutorial]

## Database Connection
PostgreSQL is a relational database. To connect...
[40 lines of database basics]

## Full SQLAlchemy Documentation
[200 lines of ORM documentation]

## Testing
pytest is a testing framework. To write a test...
[30 lines of pytest basics]

## Current Server IPs
Production: 10.0.1.45
Staging: 10.0.2.89
Development: 10.0.3.12
```

**AFTER (Concise, 45 lines):**
```markdown
# Project Instructions

## Build Commands
- Test: pytest tests/ -v
- Lint: ruff check src/
- Run: uvicorn src.main:app --reload

## Code Standards
- Type hints required for public functions
- Use repository pattern for data access (see repos/base.py)
- Async/await for all I/O operations

## Key Patterns
- Authentication: src/auth/middleware.py:45-89
- File validation: src/utils/validators.py:23-67
- Error handling: Follow pattern in src/api/routes.py:120

## Never Modify
- config/production.yaml (managed by DevOps)
- .env files (local secrets)

## Compliance
- Log all user actions (IRS requirement)
- Session timeout: 30min idle, 12h absolute
- File uploads: validate size, type, magic bytes
```

## Import Syntax

Use @ to include other files (keeps CLAUDE.md lean).

**CRITICAL: @import paths must be absolute. Claude Code does NOT expand tilde (~).**

```markdown
# WRONG - tilde is not expanded; file silently not loaded
@~/.claude/docs/claude-aliunde.md

# CORRECT - use absolute path
@/Users/donmccarty/.claude/docs/claude-aliunde.md
@/Users/donmccarty/ali-ai/roles/Chief-of-Staff-Core.md
```

For paths that vary by user or machine, the claude-ali launcher uses `${HOME}` shell expansion to write absolute paths into CLAUDE.md at launch time. Never hand-write a tilde @import — it will silently fail (the file is never loaded, no error is shown).

```markdown
# Correct pattern for project-relative and local files
@docs/api-reference.md
@shared/coding-standards.md
```

Project-relative paths (no leading /) work fine. Only tilde expansion is broken.

## Path-Specific Rules

Create .claude/rules/ for path-specific instructions:

```yaml
# .claude/rules/api-security.md
---
paths:
  - src/api/**/*.py
  - src/routes/**/*.py
---

When modifying API endpoints:
- Validate all user input
- Use Pydantic models for request/response
- Add rate limiting to public endpoints
```
