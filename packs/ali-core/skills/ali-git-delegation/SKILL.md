---
name: ali-git-delegation
description: |
  Git operations delegation enforcement. Use when:

  PLANNING: Planning git workflows, considering version control strategies,
  designing branch approaches, evaluating commit patterns

  IMPLEMENTATION: About to execute git commands, preparing commits, managing branches,
  creating PRs, pushing changes, performing any git operation

  GUIDANCE: Asking about git operations, commit conventions, branch management,
  how to use version control, git best practices

  REVIEW: Reviewing git history, checking branch state, validating commit messages,
  auditing git operations
---

# Git Delegation

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Planning git workflows or branching strategies
- Considering how to structure commits
- Designing release or deployment processes
- Evaluating merge vs rebase approaches

**Implementation:**
- Any attempt to run git or gh commands
- Preparing to commit changes
- About to push, pull, merge, or rebase
- Creating branches or PRs
- Tagging releases

**Guidance/Best Practices:**
- Asking how to use git commands
- Asking about commit message conventions
- Asking about branch management
- Seeking git workflow advice

**Review/Validation:**
- Reviewing commit history
- Checking git repository state
- Validating branch structure
- Auditing past git operations

---

## Key Principles

- **You do NOT run git/gh commands directly** - Always delegate to /github-admin
- **This applies even with Bash permissions** - Having permission doesn't mean you should use it
- **Delegation is mandatory, not optional** - No exceptions for "simple" operations
- **github-admin provides critical safety** - Hooks can be bypassed by permissions, delegation cannot
- **Read-only is still delegated** - Even git status should go through github-admin for consistency

---

## Why Delegation Exists

### The Permission Bypass Problem

**Context:** We have a PreToolUse hook (ali-git-guard.sh) that blocks git commands and suggests /github-admin. However, if users click "always allow" for git commands, the permission bypasses the hook entirely.

**What happens:**
1. User runs git command → hook blocks it → suggests /github-admin
2. User clicks "always allow" for git commands
3. Future git commands bypass hook completely
4. No safety checks, no audit trail, no enforcement

**Solution:** This skill provides behavioral guidance at the context level. The model learns to delegate BEFORE considering running git directly.

### What github-admin Provides

| Capability | Why It Matters |
|------------|----------------|
| **No Claude branding** | Aliunde SOP prohibits Co-Authored-By, emoji, "Generated with Claude Code" |
| **Conventional commits** | Enforces feat/fix/docs format, prevents "WIP" or "fixed stuff" |
| **Pre-push safety** | Checks git status, git diff, branch state before pushing |
| **Dangerous operation gates** | Requires explicit confirmation for force push, hard reset, etc. |
| **Audit trail** | All operations logged to ~/.claude/logs/git-operations.log |
| **Error recovery** | Provides actionable guidance when git operations fail |

---

## How to Delegate

### Pattern: Always Use /github-admin

```markdown
# User asks to commit changes
/github-admin

# Then describe what you want
"Create a commit with these changes: [describe changes]"
```

### For ALL Git Operations

| User Request | Your Response |
|--------------|---------------|
| "Commit these changes" | Spawn /github-admin agent, describe changes |
| "Push to remote" | Spawn /github-admin agent, request push |
| "Create a branch" | Spawn /github-admin agent, describe branch |
| "Show git status" | Spawn /github-admin agent, request status |
| "Create a PR" | Spawn /github-admin agent, describe PR |
| "Check commit history" | Spawn /github-admin agent, request log |

### Example Delegations

**User:** "Commit the authentication changes"

**Your response:**
```markdown
I'll delegate this to the github-admin agent to ensure proper commit formatting
and safety checks.

[Spawn /github-admin agent]

Task: Create a commit with the authentication changes we just implemented.
Files modified:
- src/auth/oauth.py (added OAuth flow)
- src/auth/middleware.py (updated session handling)
- tests/test_auth.py (added OAuth tests)

Suggested commit message: feat(auth): implement OAuth2 login flow
```

**User:** "Push these changes"

**Your response:**
```markdown
I'll delegate this push operation to github-admin to ensure pre-push safety checks.

[Spawn /github-admin agent]

Task: Push the current branch to remote.
Branch: feat/oauth-login
Commits ready: 3 commits since main
```

**User:** "What's the current git status?"

**Your response:**
```markdown
I'll check the repository state through github-admin.

[Spawn /github-admin agent]

Task: Check current git repository status (git status, current branch, uncommitted changes).
```

---

## What About Read-Only Operations?

**Q:** "git status is safe, can't I just run it directly?"

**A:** No. Delegate even read-only operations to github-admin.

**Why:**
1. **Consistency** - All git operations go through one path
2. **Context** - github-admin maintains state awareness
3. **Audit** - Even read operations should be logged
4. **Pattern reinforcement** - Builds delegation habit

**Safe read-only operations that still delegate:**
- git status
- git log
- git diff
- git show
- git branch -a
- gh pr list
- gh issue list

---

## When Delegation Happens

### Before ANY git Command

**Pattern recognition triggers:**
- User mentions "git" or "github"
- User requests "commit", "push", "branch", "merge"
- User asks about "PR" or "pull request"
- User wants to "tag" or "release"
- User says "check git status"

**Your immediate action:**
1. Recognize this is a git operation
2. Spawn /github-admin agent
3. Provide context about what's needed
4. Let github-admin handle the execution

### Example Conversation Flow

**User:** "I made some changes to the database schema. Can you commit them?"

**You (WRONG - don't do this):**
```bash
# Running git add...
# ERROR: You should NOT be running git commands
```

**You (CORRECT):**
```markdown
I'll delegate this to github-admin to ensure proper commit formatting and safety checks.

[Spawn /github-admin agent]

Task: Create a commit for database schema changes.
Files modified: src/models/schema.py, migrations/001_add_users.sql
Suggested message: feat(db): add user authentication tables
```

---

## Common Scenarios

### Scenario: User Requests a Commit

**User:** "Commit these changes with message 'fixed login bug'"

**Your delegation:**
```markdown
I'll handle this commit through github-admin, which will enforce our commit message
conventions (the message should follow conventional commit format).

[Spawn /github-admin agent]

Task: Create a commit for the login bug fix.
Files changed: [list files]
User's message: "fixed login bug"
Note: github-admin will reformat to conventional commit: "fix(auth): resolve login validation bug"
```

### Scenario: User Wants to Push

**User:** "Push this to remote"

**Your delegation:**
```markdown
I'll delegate this push to github-admin for pre-push safety checks.

[Spawn /github-admin agent]

Task: Push current branch to remote.
Branch: [branch name]
Ensure: Up to date with main, tests pass, no conflicts
```

### Scenario: User Creates PR

**User:** "Create a PR for this feature"

**Your delegation:**
```markdown
I'll create the PR through github-admin to ensure proper formatting and checklist completion.

[Spawn /github-admin agent]

Task: Create pull request for [feature name]
Branch: [branch]
Title: feat: [feature description]
Include: Summary, test plan, checklist
```

### Scenario: User Asks About Git State

**User:** "What's the status of the repository?"

**Your delegation:**
```markdown
I'll check the repository state through github-admin.

[Spawn /github-admin agent]

Task: Report repository status:
- Current branch
- Uncommitted changes
- Commits ahead/behind remote
- Untracked files
```

---

## Anti-Patterns to Catch

| Anti-Pattern | Why Bad | Do Instead |
|--------------|---------|------------|
| Running git commands directly | Bypasses safety, breaks SOP | Always delegate to /github-admin |
| "It's just git status, I'll run it" | Inconsistent pattern | Delegate even read-only operations |
| Assuming you have permission | Permissions can bypass hooks | Delegation is mandatory regardless |
| Only delegating "dangerous" operations | Creates confusion about when to delegate | Delegate ALL git operations |
| Running gh commands directly | Same issues as git | Delegate gh operations too |

---

## Quick Reference

### Delegation Commands

```markdown
# Commit changes
/github-admin
"Create commit for [changes]"

# Push to remote
/github-admin
"Push [branch] to remote"

# Create branch
/github-admin
"Create branch feat/[feature-name]"

# Create PR
/github-admin
"Create PR for [feature]"

# Check status
/github-admin
"Check git status and repository state"

# View history
/github-admin
"Show recent commit history"
```

### What github-admin Handles

- ✅ Commit message formatting (conventional commits)
- ✅ No Claude branding (no Co-Authored-By, no emoji)
- ✅ Pre-push safety checks
- ✅ Branch naming conventions
- ✅ Dangerous operation confirmation
- ✅ Audit logging
- ✅ Error recovery guidance

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| Git guard hook | ~/ali-ai/hooks/ali-git-guard.sh |
| github-admin agent | ~/ali-ai/agents/ali-github-admin.md |
| github-admin skill | ~/ali-ai/skills/ali-github-admin/SKILL.md |
| Git operation logs | ~/.claude/logs/git-operations.log |
| Git audit trail | ~/.claude/logs/git-audit.log |
| Aliunde SOP (no Claude branding) | ~/.claude/docs/claude-aliunde.md:241-242 |

---

## References

- [Conventional Commits](https://www.conventionalcommits.org/)
- [GitHub Admin Agent](~/ali-ai/agents/ali-github-admin.md)
- [Git Guard Hook](~/ali-ai/hooks/ali-git-guard.sh)
- [Aliunde Development Standards](~/.claude/docs/claude-aliunde.md)

---

**Document Version:** 1.0
**Last Updated:** 2026-01-29
**Maintained By:** ALI AI Team
