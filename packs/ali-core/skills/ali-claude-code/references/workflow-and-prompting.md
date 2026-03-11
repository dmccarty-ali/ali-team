# Workflow Phases

Effective Claude Code work follows four phases: **Explore → Plan → Implement → Commit**

## The Four Phases

| Phase | Purpose | Tools | Permission Mode |
|-------|---------|-------|----------------|
| Explore | Understand codebase | Read, Grep, Glob | plan |
| Plan | Design solution | Read-only | plan |
| Implement | Write code | All tools | default |
| Commit | Finalize changes | Git | default |

## Plan Mode - Safe Analysis

Use /plan or --permission-mode plan for safe, read-only exploration:

```bash
# Enter plan mode
/plan

# Or start with plan mode
claude --permission-mode plan

# Plan mode allows:
# - Read, Grep, Glob
# - No Write, Edit, Bash (except safe commands)
```

**When to use plan mode:**
- Unfamiliar codebase
- Large refactoring (need to map dependencies)
- Architectural decisions
- Investigating bugs

**Benefits:**
- Can't accidentally modify files
- Explore freely without risk
- Cheaper (read-only, less context)
- Better quality (separate thinking from doing)

## When to Skip Planning

**Skip planning for:**
- Small changes (<50 lines)
- Clear, well-defined tasks
- Familiar code patterns
- Obvious fixes

**Example skip-worthy tasks:**
- "Add email field to User model"
- "Fix typo in error message"
- "Update dependency version"

## When Planning is Essential

**Always plan for:**
- Multi-file changes
- Architectural decisions
- Unfamiliar codebase areas
- Complex refactoring
- Cross-cutting concerns

**Example planning-required tasks:**
- "Refactor authentication to use OAuth"
- "Add caching layer to API"
- "Migrate database from PostgreSQL to Aurora"

## Example Workflow

```bash
# Phase 1: Explore
claude --permission-mode plan
> "Map all files that handle user authentication"
> "Show me the current error handling pattern"

# Phase 2: Plan
> "Design approach for adding OAuth support"
> "What files need to change? What's the migration path?"

# Phase 3: Implement
/model default
> "Implement the OAuth changes we planned"

# Phase 4: Commit
> "Run tests and commit with message: feat(auth): add OAuth support"
```

---

# Prompting Patterns

How you ask determines what you get. Effective prompts are specific, provide context, and point to sources.

## Prompting Best Practices

| Pattern | Example |
|---------|---------|
| Scope specifically | "Fix login validation in auth.py, lines 45-67" not "improve auth" |
| Point to sources | "Follow the pattern in api/routes.py:120-145" |
| Reference history | "The bug was introduced in commit abc123f" |
| Describe symptoms | "API returns 500 when user email is null, likely in validation logic" |
| Let Claude interview | For large features, let Claude ask clarifying questions |

## Effective vs Anti-Pattern Prompts

| Anti-Pattern | Why Bad | Effective Pattern |
|--------------|---------|-------------------|
| "Fix the bug" | Too vague | "Fix NullPointerException in UserService.validate(), line 67" |
| "Make it better" | No direction | "Refactor to follow repository pattern (see OrderRepo.py)" |
| "Add feature X" | Underspecified | "Add email validation to signup form - follow pattern in auth.py:45" |
| "Why doesn't this work?" | No context | "API returns 500, logs show 'database connection timeout', likely in db.py" |
| Silence (on complex features) | Missing requirements | "I want to add X. Ask me questions to clarify requirements." |

## Let Claude Interview You

For larger, ambiguous features, explicitly invite questions:

```
"I want to add a payment processing feature. Ask me questions to understand:
- Which payment providers to support
- Where in the workflow payments occur
- What happens on payment failure
- Security and compliance requirements"
```

Claude will use the AskUserQuestion tool to gather requirements before starting.

## Point to Existing Patterns

"We do X in the codebase" is more effective than describing X:

**BAD:**
> "Add error handling with try/catch and log to our logging service"

**GOOD:**
> "Add error handling following the pattern in user_service.py:89-105"

## Use Git History as Context

Reference commits, branches, or PRs:

```
"The refactor in commit f3a8b introduced async patterns.
Apply the same pattern to the payment service."

"PR #234 shows how we migrated auth to use OAuth.
Follow that approach for the admin panel."
```

## Describe Symptoms with Likely Location

When debugging, provide symptoms AND where you think the issue is:

```
"User logout sometimes fails silently (no error, but session persists).
Likely in session_manager.py or the /logout route handler.
Happens intermittently when user has multiple tabs open."
```

This helps Claude narrow the search space quickly.

---

# Rich Content Patterns

Claude Code supports rich input formats - use them to provide better context.

## @file References

Reference files directly in prompts:

```
"Review @src/auth.py for security issues"
"Compare @config/dev.yaml and @config/prod.yaml"
```

## Paste Images Directly

Ctrl+V to paste screenshots, diagrams, error screens:

```
# Terminal
Ctrl+V  # Pastes clipboard image

# Then describe
"This error appears when users try to login. Fix it."
```

## URLs for Documentation

Claude can fetch URLs:

```
"Implement pagination following this spec: https://api.example.com/docs/pagination"
```

## Pipe Data

Pipe command output directly:

```bash
git log --oneline -10 | claude "Analyze commit patterns"
cat error.log | claude "Find root cause"
docker ps | claude "Are all services healthy?"
```

## Let Claude Fetch What It Needs

Don't pre-fetch everything. Let Claude use Read, Grep, Glob:

**BAD:**
> [Paste 20 files into prompt]
> "Find where we handle user sessions"

**GOOD:**
> "Find where we handle user sessions"
> [Claude uses Grep to search]

Claude is better at finding relevant code than you are at guessing what's needed.
