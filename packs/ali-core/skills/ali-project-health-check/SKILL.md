---
name: ali-project-health-check
description: |
  Validates that an Aliunde project meets all required setup standards. Use when:

  PLANNING: designing a new project setup, onboarding a project to ali-ai, planning project structure

  IMPLEMENTATION: running a health check, fixing project configuration, initializing missing project files

  GUIDANCE: asking what a project needs, understanding project requirements, checking what's required for org sync

  REVIEW: auditing project setup, verifying project compliance, validating configuration before first launch
---

# Project Health Check

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing a new Aliunde project setup
- Onboarding an existing project to the ali-ai org
- Planning what files and structure a project needs
- Evaluating whether a project is ready for the CoS workflow

**Implementation:**
- Running the health check script on a project
- Fixing issues found by the health check
- Initializing missing required files
- Correcting role path or hook configuration

**Guidance/Best Practices:**
- Asking what a project needs to be valid
- Understanding why a particular requirement exists
- Asking how to fix a specific health check failure
- Understanding the difference between .claude/, .claude-local/, and .claude-config/

**Review/Validation:**
- Auditing a project's setup for compliance
- Verifying all required files are present and correct
- Validating configuration before first claude-ali launch
- Pre-merge checks for project configuration

## When NOT to Use This Skill

This skill does NOT activate for:
- General infrastructure questions (use ali-infrastructure instead)
- Creating new agents or skills (use ali-claude-code instead)
- Debugging claude-ali launcher failures (use ali-infrastructure instead)

---

## Key Principles

- **The .aliunde-project marker is the gatekeeper:** Without it, claude-ali will not sync org resources. This is the first check.
- **Role path must be absolute and point to ~/.claude/roles/:** Tilde paths (~/) fail silently. Paths to ~/ali-ai/roles/ break when ali-ai is not at the expected location. Roles ship via the release tarball to ~/.claude/roles/.
- **Hooks require .claude-config/settings.json:** The CoS session enforcement hooks live in settings.json. Without this file, hooks are not active.
- **architecture_stack.txt enables automated routing:** The expert-panel and database-developer agents use this file for auto-detection. Missing it degrades specialist routing quality.
- **ARCHITECTURE.md is required documentation:** All Aliunde projects must have human-readable architecture docs.
- **Automated script is the canonical check:** ~/.claude/bin/ali-health-check runs all checks and produces a structured report. Always use it when available.

---

## What the Health Check Validates

### Check 1: .aliunde-project Marker

**File:** .aliunde-project (project root)

**Why it matters:** The claude-ali launcher looks for this marker to decide whether to sync org resources (agents, skills, hooks). Without it, the project launches as a plain Claude Code session with no org tools.

**Pass:** File exists at project root (contents do not matter)

**Fail:** File missing

**Fix:**
```bash
cat > .aliunde-project << 'EOF'
# Aliunde Project Marker
# Enables org resource sync from ~/ali-ai via claude-ali launcher
EOF
```

---

### Check 2: CLAUDE.md with ## Role Section

**File:** CLAUDE.md (project root)

**Why it matters:** The ## Role section imports the Chief-of-Staff-Core.md role definition. Without it, Claude Code launches without the CoS role active, meaning no ISPR enforcement, no delegation gates, and no session protocol.

**Pass:** CLAUDE.md exists AND contains a ## Role section with an @-import line

**Fail (missing file):** CLAUDE.md does not exist

**Fail (missing section):** CLAUDE.md exists but has no ## Role section

**Fix:** Add to CLAUDE.md:
```markdown
## Role

<!-- Path must be absolute, not tilde. Roles are installed to ~/.claude/roles/ by the ali-ai release. -->
@/Users/yourusername/.claude/roles/Chief-of-Staff-Core.md
```

Replace yourusername with the actual macOS username.

---

### Check 3: Role Path Points to ~/.claude/roles/

**Why it matters:** This is a common misconfiguration. Three variants are wrong:

| Wrong Path | Problem |
|-----------|---------|
| `~/ali-ai/roles/Chief-of-Staff-Core.md` | Breaks when ali-ai is not at ~/ali-ai; older pattern |
| `~/.claude/roles/Chief-of-Staff-Core.md` | Tilde syntax fails in Claude Code @-imports |
| `/Users/yourusername/ali-ai/roles/...` | Points to wrong location; roles ship to ~/.claude/roles/ |

**Correct path:** `/Users/{username}/.claude/roles/Chief-of-Staff-Core.md`

The roles directory is populated by the ali-ai release tarball, not the git clone. Using the ~/ali-ai/roles/ path couples the project to the development checkout rather than the published release.

**Pass:** @-import line uses absolute path matching `/Users/*/\.claude/roles/Chief-of-Staff-Core\.md`

**Fail:** Path uses tilde, points to ali-ai/roles/, or is missing entirely

**Fix:** Update the @-import in the ## Role section:
```markdown
@/Users/yourusername/.claude/roles/Chief-of-Staff-Core.md
```

---

### Check 4: .claude-config/settings.json Exists

**File:** .claude-config/settings.json (project root)

**Why it matters:** This file contains the hooks configuration (session protocol, git guard, etc.). It also isolates credentials so multiple Claude sessions can run with different accounts. Without it, hooks are not active and credential isolation is not set up.

**Pass:** .claude-config/settings.json exists

**Fail:** Directory or file missing

**Fix:**
```bash
# Run the dev environment setup script (interactive, sets up credentials + hooks)
~/.claude/bin/setup_dev_environment.sh

# Or create minimal settings.json manually
mkdir -p .claude-config
cp ~/.claude/templates/settings.json .claude-config/settings.json
```

The setup script is preferred — it configures credentials (OAuth isolation) in addition to the settings file.

---

### Check 5: architecture_stack.txt Exists

**File:** architecture_stack.txt (project root)

**Why it matters:** The expert-panel agent uses this file to boost expert scores for your tech stack. The database-developer agent uses it to auto-detect database type (PostgreSQL, Snowflake, etc.). The backend-developer verifies approved frameworks. Missing this file degrades automated agent/skill routing.

**Pass:** architecture_stack.txt exists

**Fail:** File missing

**Fix:** Create using the project template:
```bash
cp ~/ali-ai/PROJECT_TEMPLATE_architecture_stack.txt architecture_stack.txt
# Then edit it to match your project's actual tech stack
```

Required sections:
- project (name, type, description)
- databases (operational, warehouse, cache)
- languages (with versions)
- frameworks (backend, frontend, etc.)
- infrastructure (cloud, compute, storage)
- testing (frameworks used)
- integrations (external services)
- security (auth, compliance)
- monitoring (APM, logging)

---

### Check 6: ARCHITECTURE.md Exists

**File:** ARCHITECTURE.md (project root)

**Why it matters:** All Aliunde projects must have human-readable architecture documentation. The documentation-review agent validates consistency between ARCHITECTURE.md and architecture_stack.txt. New team members and onboarding agents use this file to understand system design.

**Pass:** ARCHITECTURE.md exists

**Fail:** File missing

**Fix:** Create ARCHITECTURE.md with sections covering:
- System overview and purpose
- Technology stack (must stay in sync with architecture_stack.txt)
- Component architecture
- Data flow
- Integration points
- Security model
- Infrastructure

---

## Running the Health Check

### Automated Script (Preferred)

```bash
# Run health check on current directory
~/.claude/bin/ali-health-check

# Run on specific project
~/.claude/bin/ali-health-check /path/to/project

# Verbose output (show passing checks)
~/.claude/bin/ali-health-check --verbose
```

The script prints a structured report:

```
Project Health Check: /Users/don/ali-ai/projects/my-project
============================================================

PASS  .aliunde-project marker present
PASS  CLAUDE.md exists
PASS  CLAUDE.md has ## Role section
FAIL  Role path should point to ~/.claude/roles/ (found: ~/ali-ai/roles/)
FAIL  .claude-config/settings.json missing
PASS  architecture_stack.txt present
FAIL  ARCHITECTURE.md missing

Summary: 4 passed, 3 failed
Run with --fix to auto-correct fixable issues
```

### Manual Check

If the script is not available, verify each file manually:

```bash
# Check all required files at once
ls -la .aliunde-project CLAUDE.md architecture_stack.txt ARCHITECTURE.md .claude-config/settings.json

# Check role path specifically
grep -A3 "## Role" CLAUDE.md
```

### CoS Session Invocation

From a CoS session, request a health check like this:

```
"Run the project health check on this project and report what's failing."
```

The CoS will delegate to an admin agent to run ~/.claude/bin/ali-health-check and report findings.

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Role path uses ~/ | Tilde fails in @-imports | Use full absolute path: /Users/username/... |
| Role path points to ali-ai/roles/ | Couples to dev checkout, not release | Point to ~/.claude/roles/ |
| Missing .aliunde-project | Org resources never sync | Create marker file at project root |
| .claude-config/ checked into git | Exposes OAuth credentials | Add .claude-config/ to .gitignore |
| Skipping architecture_stack.txt | Expert routing degrades silently | Create from template, keep in sync with ARCHITECTURE.md |
| ARCHITECTURE.md out of sync with stack | Documentation expert flags inconsistencies | Update both files when stack changes |

---

## Fix Reference

| Check | Automated Fix Available? | Manual Fix |
|-------|------------------------|------------|
| .aliunde-project missing | Yes (--fix flag) | Create file with marker content |
| CLAUDE.md missing | Partial (creates template) | Copy from PROJECT_TEMPLATE.md |
| Role section missing | No | Add ## Role section manually |
| Role path wrong | Yes (--fix flag) | Update @-import path |
| .claude-config/settings.json missing | No (credentials require interactive setup) | Run setup_dev_environment.sh |
| architecture_stack.txt missing | Partial (copies template) | Copy template and edit |
| ARCHITECTURE.md missing | No | Write from scratch |

---

## Quick Reference

```bash
# Run health check
~/.claude/bin/ali-health-check

# Run with auto-fix for fixable issues
~/.claude/bin/ali-health-check --fix

# Check role path in CLAUDE.md
grep -A2 "## Role" CLAUDE.md

# Correct role path pattern
@/Users/$(whoami)/.claude/roles/Chief-of-Staff-Core.md

# Setup dev environment (handles .claude-config/settings.json)
~/.claude/bin/setup_dev_environment.sh

# Copy architecture stack template
cp ~/ali-ai/PROJECT_TEMPLATE_architecture_stack.txt architecture_stack.txt
```

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| Health check script | ~/.claude/bin/ali-health-check |
| Project template | ~/ali-ai/PROJECT_TEMPLATE.md |
| Architecture stack template | ~/ali-ai/PROJECT_TEMPLATE_architecture_stack.txt |
| Role definition | ~/.claude/roles/Chief-of-Staff-Core.md |
| Dev environment setup | ~/.claude/bin/setup_dev_environment.sh |
| Settings template | ~/.claude/templates/settings.json |

---

**Document Version:** 1.0
**Last Updated:** 2026-02-21
**Maintained By:** ALI AI Team
