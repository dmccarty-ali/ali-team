---
name: ali-github-admin
description: |
  Git and GitHub operations with safety enforcement. Use when:

  PLANNING: Planning git workflows, branch strategies, release processes,
  considering merge approaches, designing CI/CD pipelines

  IMPLEMENTATION: Executing git commands, creating commits, managing branches,
  creating PRs, managing releases, configuring repositories

  GUIDANCE: Asking about git best practices, commit message conventions,
  branch naming, merge vs rebase, release workflows

  REVIEW: Reviewing git history, auditing commits, checking branch state,
  validating release readiness
---

# GitHub Admin

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Planning branch strategies (GitFlow, trunk-based, etc.)
- Designing release workflows
- Choosing merge strategies (squash, rebase, merge)
- Planning repository structure

**Implementation:**
- Executing git commands (commit, push, merge, rebase)
- Creating and managing branches
- Creating pull requests
- Managing releases and tags
- Configuring GitHub settings

**Guidance/Best Practices:**
- Commit message conventions
- Branch naming standards
- When to rebase vs merge
- Release versioning (semver)
- Git workflow patterns

**Review/Validation:**
- Reviewing git history
- Auditing branch state
- Checking for uncommitted changes
- Validating release readiness

---

## REQUIRED: Bypass Prefix for All Git Bash Commands

**Every git Bash command executed from this skill MUST be prefixed with `ALIUNDE_GIT_ADMIN=1`.**

### Why This Is Mandatory

The ali-git-guard.sh hook intercepts all Bash git commands and blocks them, redirecting callers to this skill. Without the bypass prefix, this creates a circular block: the hook tells you to use ali-github-admin, then blocks ali-github-admin when it tries to run git commands.

The bypass prefix signals to the hook that execution is already inside the authorized skill context and the command should be allowed through.

**Without the prefix:** The hook blocks the command. You see a DELEGATE message redirecting back to this skill. The git command never runs.

**With the prefix:** The hook recognizes authorized execution, logs it, and allows the command through. The prefix is stripped before the actual git command runs.

### Bypass Prefix Pattern

```bash
# Pattern: ALIUNDE_GIT_ADMIN=1 <git command>

# Commit
ALIUNDE_GIT_ADMIN=1 git commit -m "feat(scope): description"

# Push
ALIUNDE_GIT_ADMIN=1 git push origin branch-name

# Add files
ALIUNDE_GIT_ADMIN=1 git add src/auth/oauth.py

# Create tag
ALIUNDE_GIT_ADMIN=1 git tag -a v1.2.0 -m "Release v1.2.0"

# Push tag
ALIUNDE_GIT_ADMIN=1 git push origin v1.2.0

# Force push (personal branch only)
ALIUNDE_GIT_ADMIN=1 git push --force-with-lease origin feat/my-branch

# Create branch
ALIUNDE_GIT_ADMIN=1 git checkout -b feat/TAX-123-new-feature

# Switch branch
ALIUNDE_GIT_ADMIN=1 git switch main
```

### What Does NOT Need the Prefix

Read-only git commands are not intercepted by the hook. These run normally without the prefix:

```bash
git status
git log
git diff
git show
git branch -a
git remote -v
git fetch
gh pr list
gh pr view
gh issue list
```

### Note on gh (GitHub CLI) Write Commands

The same rule applies to gh commands that write or modify state:

```bash
# gh write operations also need the prefix
ALIUNDE_GIT_ADMIN=1 gh pr create --title "..." --body "..."
ALIUNDE_GIT_ADMIN=1 gh release create v1.2.0 --title "v1.2.0" --notes "..."
ALIUNDE_GIT_ADMIN=1 gh pr merge 42 --squash
```

---

## Key Principles

- **Feature branch is the org-wide default** - All work happens on feature branches; no one commits directly to main/master in any Aliunde project
- **main is merge-only** - main is only ever touched via PR merge; direct push to main is blocked by policy and by hook
- **Never force push to main/master** - Rewrites shared history, causes team disruption
- **Never skip hooks** - Pre-commit hooks exist for a reason (lint, test, security)
- **Atomic commits** - Each commit should be a single logical change
- **Meaningful messages** - Commit messages explain WHY, not just WHAT
- **Branch hygiene** - Delete merged branches, keep active branches minimal
- **Tag releases** - Use semantic versioning (vX.Y.Z) for all releases
- **Review before push** - Always `git diff` and `git status` before pushing

---

## Commit Message Standards

### Format

```
<type>(<scope>): <subject>

<body>

<footer>
```

### Types

| Type | Description |
|------|-------------|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `style` | Formatting, no code change |
| `refactor` | Code restructuring, no behavior change |
| `perf` | Performance improvement |
| `test` | Adding or updating tests |
| `chore` | Build, CI, dependencies |

### Examples

```bash
# Good
feat(auth): add OAuth2 login with Google

Implements Google OAuth2 flow for user authentication.
Adds callback handler and token refresh logic.

Closes #123

# Bad
fixed stuff
updated code
WIP
```

### Rules

- Subject line: 50 chars max, imperative mood ("add" not "added")
- Body: Wrap at 72 chars, explain WHY not WHAT
- Footer: Reference issues, breaking changes
- **NO Claude branding** - No Co-Authored-By, no emoji, no "Generated with Claude Code" (Aliunde SOP)

---

## Branch Naming

### Format

```
<type>/<ticket>-<description>
```

### Examples

| Branch | Purpose |
|--------|---------|
| `feat/TAX-123-add-login` | Feature work |
| `fix/TAX-456-null-pointer` | Bug fix |
| `chore/update-dependencies` | Maintenance |
| `release/v1.2.0` | Release branch |
| `hotfix/v1.2.1` | Production hotfix |

### Protected Branches

| Branch | Protection |
|--------|------------|
| `main` | Require PR, require reviews, require CI pass |
| `release/*` | Require PR, require reviews |

---

## Dangerous Operations

These operations are blocked by git-guard and require explicit confirmation:

| Operation | Risk | Why Dangerous |
|-----------|------|---------------|
| `git push --force` | High | Rewrites remote history, affects collaborators |
| `git push --force-with-lease` | Medium | Still rewrites history, safer but risky |
| `git reset --hard` | High | Permanently discards uncommitted changes |
| `git branch -D` | Medium | Deletes branch without merge check |
| `git clean -f` | High | Permanently deletes untracked files |
| `git stash drop/clear` | Medium | Permanently deletes stashed changes |
| `git rebase` (on main) | High | Rewrites shared history |
| `gh repo delete` | Critical | Destroys entire repository |
| `gh secret set/delete` | High | Modifies CI/CD secrets |

### When Force Push is Acceptable

1. **Personal feature branch** - Only you are working on it
2. **After rebase** - To update PR with rebased commits
3. **Fixing sensitive data leak** - Removing secrets from history

Always use `--force-with-lease` instead of `--force` when possible.

---

## Safe Operations

These operations are always safe and don't require confirmation:

```bash
# Read-only operations
git status
git log
git diff
git show
git branch -a
git remote -v
git fetch

# Standard workflow (remember: prefix required for write operations)
ALIUNDE_GIT_ADMIN=1 git add <files>
ALIUNDE_GIT_ADMIN=1 git commit -m "message"
ALIUNDE_GIT_ADMIN=1 git push  # (without --force)
git pull
ALIUNDE_GIT_ADMIN=1 git checkout <branch>
ALIUNDE_GIT_ADMIN=1 git switch <branch>

# GitHub CLI read operations
gh pr list
gh pr view
gh issue list
gh issue view
gh repo view
```

---

## Git Workflows

**Org-wide policy:** Feature branch is the required workflow for all Aliunde projects, not an option among many. Every change — no matter how small — goes to a feature branch and merges to main via PR. main is never pushed to directly.

### Feature Branch Workflow

```bash
# 1. Create feature branch from main
git checkout main
git pull
ALIUNDE_GIT_ADMIN=1 git checkout -b feat/TAX-123-new-feature

# 2. Make changes and commit
ALIUNDE_GIT_ADMIN=1 git add .
ALIUNDE_GIT_ADMIN=1 git commit -m "feat(module): add new feature"

# 3. Push and create PR
ALIUNDE_GIT_ADMIN=1 git push -u origin feat/TAX-123-new-feature
ALIUNDE_GIT_ADMIN=1 gh pr create --title "feat: add new feature" --body "..."

# 4. After review, merge via GitHub (squash recommended)

# 5. Clean up
ALIUNDE_GIT_ADMIN=1 git checkout main
git pull
ALIUNDE_GIT_ADMIN=1 git branch -d feat/TAX-123-new-feature
```

### Hotfix Workflow

```bash
# 1. Create hotfix from main
git checkout main
git pull
ALIUNDE_GIT_ADMIN=1 git checkout -b hotfix/v1.2.1

# 2. Fix and commit
ALIUNDE_GIT_ADMIN=1 git add .
ALIUNDE_GIT_ADMIN=1 git commit -m "fix(critical): resolve production issue"

# 3. Tag and push
ALIUNDE_GIT_ADMIN=1 git tag -a v1.2.1 -m "Hotfix: resolve production issue"
ALIUNDE_GIT_ADMIN=1 git push origin hotfix/v1.2.1 --tags

# 4. Create PR to main
ALIUNDE_GIT_ADMIN=1 gh pr create --title "hotfix: v1.2.1" --body "..."
```

### Rebase Workflow (for clean history)

```bash
# 1. Update feature branch with latest main
ALIUNDE_GIT_ADMIN=1 git checkout feat/my-feature
git fetch origin
ALIUNDE_GIT_ADMIN=1 git rebase origin/main

# 2. Resolve conflicts if any
# Edit files, then:
ALIUNDE_GIT_ADMIN=1 git add <resolved-files>
ALIUNDE_GIT_ADMIN=1 git rebase --continue

# 3. Force push (only if branch is personal)
ALIUNDE_GIT_ADMIN=1 git push --force-with-lease

# Note: Never rebase shared branches
```

---

## Release Process

### Semantic Versioning

```
MAJOR.MINOR.PATCH
  │     │     └── Bug fixes (backwards compatible)
  │     └──────── New features (backwards compatible)
  └────────────── Breaking changes
```

### Creating a Release

```bash
# 1. Ensure main is up to date
git checkout main
git pull

# 2. Create release tag
ALIUNDE_GIT_ADMIN=1 git tag -a v1.2.0 -m "Release v1.2.0: Add feature X, fix bug Y"

# 3. Push tag
ALIUNDE_GIT_ADMIN=1 git push origin v1.2.0

# 4. Create GitHub release
ALIUNDE_GIT_ADMIN=1 gh release create v1.2.0 \
  --title "v1.2.0" \
  --notes "## What's New
- Feature X
- Bug fix Y

## Breaking Changes
None"
```

---

## Pre-Push Checklist

Before pushing any changes:

- [ ] `git status` - No unexpected changes
- [ ] `git diff` - Review what you're committing
- [ ] Tests pass locally
- [ ] Lint/format checks pass
- [ ] Commit messages follow conventions
- [ ] Branch is up to date with main
- [ ] No secrets or credentials in code
- [ ] No large binary files being committed

---

## Anti-Patterns

| Anti-Pattern | Why Bad | Do Instead |
|--------------|---------|------------|
| `git add .` without review | May commit unintended files | `git add -p` or explicit files |
| Committing directly to main | Bypasses review; violates org-wide policy | Always use a feature branch |
| Pushing directly to main | main is merge-only; direct push is blocked | Open a PR from your feature branch |
| Force push to shared branch | Breaks collaborators | Only force push personal branches |
| Giant commits | Hard to review, hard to revert | Small, atomic commits |
| "WIP" commits | No information value | Meaningful messages |
| Skipping hooks | Bypasses quality gates | Fix the underlying issue |
| Not pulling before push | Causes merge conflicts | Always pull first |
| Git write commands without `ALIUNDE_GIT_ADMIN=1` | Hook blocks the command, circular redirect | Always prefix write commands with `ALIUNDE_GIT_ADMIN=1` |

---

## Audit Trail

All git operations are logged by git-guard to:

```
~/.claude/logs/git-operations.log  # Detailed log
~/.claude/logs/git-audit.log       # Structured audit trail
```

### Audit Log Format

```
[2026-01-15 10:30:45] ACTION=ALLOW COMMAND="git commit -m 'feat: add login'" RESULT=commit USER=donmccarty PWD=/Users/donmccarty/project
[2026-01-15 10:31:02] ACTION=BLOCK COMMAND="git push --force" RESULT=force-push USER=donmccarty PWD=/Users/donmccarty/project
```

---

## Quick Reference

### Daily Commands

```bash
git status                                        # Check state (no prefix needed)
git pull                                          # Update from remote (no prefix needed)
ALIUNDE_GIT_ADMIN=1 git add -p                   # Stage interactively
ALIUNDE_GIT_ADMIN=1 git commit -m "type: msg"    # Commit
ALIUNDE_GIT_ADMIN=1 git push                     # Push to remote
```

### Branch Management

```bash
git branch -a                                     # List all branches (no prefix needed)
ALIUNDE_GIT_ADMIN=1 git checkout -b <name>       # Create and switch
ALIUNDE_GIT_ADMIN=1 git switch <name>            # Switch branch
ALIUNDE_GIT_ADMIN=1 git branch -d <name>         # Delete (safe)
ALIUNDE_GIT_ADMIN=1 git branch -D <name>         # Delete (force) - REQUIRES CONFIRMATION
```

### History

```bash
git log --oneline -10         # Recent commits
git log --graph --oneline     # Visual history
git show <commit>             # Show commit details
git diff <branch>             # Compare branches
```

### Stash

```bash
ALIUNDE_GIT_ADMIN=1 git stash                    # Stash changes
git stash list                                    # List stashes (no prefix needed)
ALIUNDE_GIT_ADMIN=1 git stash pop                # Apply and remove
ALIUNDE_GIT_ADMIN=1 git stash apply              # Apply and keep
ALIUNDE_GIT_ADMIN=1 git stash drop               # Delete stash - REQUIRES CONFIRMATION
```

---

## References

- [Conventional Commits](https://www.conventionalcommits.org/)
- [Semantic Versioning](https://semver.org/)
- [GitHub Flow](https://docs.github.com/en/get-started/workflows/github-flow)
- [Git Best Practices](https://git-scm.com/book/en/v2)

---

## ali-ai Session Workflow

<!-- source-doc: RUNBOOK.md — Git Workflow section, "Daily Workflow" and "Pull Request Batching" -->

The ali-ai project uses a session-batched workflow: commit freely during a session, open one PR at session end covering all session work.

### Daily Workflow (ali-ai)

```bash
# 1. Start new work - create feature branch from main
git checkout main
git pull
ALIUNDE_GIT_ADMIN=1 git checkout -b feat/my-feature

# 2. Work freely on feature branch - commit as you go
ALIUNDE_GIT_ADMIN=1 git add .
ALIUNDE_GIT_ADMIN=1 git commit -m "feat: implement my feature"
ALIUNDE_GIT_ADMIN=1 git push  # Push freely to feature branch (backup, no CI trigger)

# 3. Continue work with additional commits throughout session
ALIUNDE_GIT_ADMIN=1 git commit -m "fix: address edge case"
ALIUNDE_GIT_ADMIN=1 git commit -m "test: add test coverage"
ALIUNDE_GIT_ADMIN=1 git push

# 4. At session end - open PR covering all session work
ALIUNDE_GIT_ADMIN=1 gh pr create --title "Add my feature" --body "..."

# 5. Merge after review/CI pass
# (done via GitHub UI or gh pr merge)

# 6. Delete feature branch after merge
ALIUNDE_GIT_ADMIN=1 git checkout main
git pull
ALIUNDE_GIT_ADMIN=1 git branch -d feat/my-feature
```

**Key difference from standard workflow:** Open PR at session end (not after each commit), batching all session work into one review.

### PR Batching Pattern

**Batch one PR per session, not one PR per change.**

For a small team, creating a PR for every individual file change generates too much overhead. Instead:

**During a session:**
- Commit freely to your feature branch as you work
- Push commits to keep your work backed up
- No need to create a PR after each commit

**At session end:**
- Push your branch if not already pushed
- Create one PR covering all session work
- PR title and body should summarize the full session's changes

**PRs needing testing before merge:**
- Can stay open across multiple sessions
- Continue committing to the same branch
- Merge when testing complete

**Example multi-session workflow:**

```bash
# Session 1 - Friday afternoon
ALIUNDE_GIT_ADMIN=1 git checkout -b feat/user-dashboard
ALIUNDE_GIT_ADMIN=1 git commit -m "feat: add dashboard route"
ALIUNDE_GIT_ADMIN=1 git commit -m "feat: add dashboard component"
ALIUNDE_GIT_ADMIN=1 git push -u origin feat/user-dashboard
# End session - NO PR YET

# Session 2 - Monday morning (same branch)
ALIUNDE_GIT_ADMIN=1 git commit -m "fix: dashboard styling"
ALIUNDE_GIT_ADMIN=1 git commit -m "test: add dashboard tests"
ALIUNDE_GIT_ADMIN=1 git push
ALIUNDE_GIT_ADMIN=1 gh pr create  # NOW create PR with all session work

# PR stays open for testing
# Merge after review/CI pass
```

### Feature Branch Operations

**Update feature branch with latest main:**

```bash
ALIUNDE_GIT_ADMIN=1 git checkout feat/my-feature
git fetch origin
ALIUNDE_GIT_ADMIN=1 git merge origin/main  # or git rebase origin/main
```

**Push to feature branch (first time):**

```bash
ALIUNDE_GIT_ADMIN=1 git push -u origin feat/my-feature
```

**Amend last commit (before pushing):**

```bash
ALIUNDE_GIT_ADMIN=1 git add .
ALIUNDE_GIT_ADMIN=1 git commit --amend --no-edit
```

**Amend last commit (after pushing — safe on personal feature branch only):**

```bash
ALIUNDE_GIT_ADMIN=1 git add .
ALIUNDE_GIT_ADMIN=1 git commit --amend --no-edit
ALIUNDE_GIT_ADMIN=1 git push --force-with-lease
```

### ali-ai CI/CD Integration

**Current CI triggers:**
- Tag push (`v*.*.*`) → Runs build-resources.yml
  - Builds and validates agents, skills, hooks
  - Creates GitHub Release with artifacts
  - claude-ali launcher on team machines pulls from latest GitHub Release

**Creating a release tag:**

```bash
# Ensure main is up to date
git checkout main
git pull

# Create and push version tag (semantic versioning)
ALIUNDE_GIT_ADMIN=1 git tag v1.2.3
ALIUNDE_GIT_ADMIN=1 git push origin v1.2.3

# CI automatically creates GitHub Release with artifacts
```

**Tag naming:** `v*.*.*` (semantic versioning)
- `v1.0.0` — Major version (breaking changes)
- `v1.1.0` — Minor version (new features, backward compatible)
- `v1.1.1` — Patch version (bug fixes)

### ali-ai Quick Reference

<!-- source-doc: RUNBOOK.md — Git Workflow section, "Quick Reference" -->

| Task | Command |
|------|---------|
| Create feature branch | `ALIUNDE_GIT_ADMIN=1 git checkout -b feat/my-feature` |
| Push to feature branch | `ALIUNDE_GIT_ADMIN=1 git push -u origin feat/my-feature` |
| Create PR | `ALIUNDE_GIT_ADMIN=1 gh pr create` |
| Update from main | `ALIUNDE_GIT_ADMIN=1 git merge origin/main` |
| Delete merged branch | `ALIUNDE_GIT_ADMIN=1 git branch -d feat/my-feature` |
| Create release tag | `ALIUNDE_GIT_ADMIN=1 git tag v1.2.3 && ALIUNDE_GIT_ADMIN=1 git push origin v1.2.3` |
| View branch protection | `gh repo view --web` (Settings → Branches) |
