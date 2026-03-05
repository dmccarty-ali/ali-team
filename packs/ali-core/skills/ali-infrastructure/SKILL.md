---
name: ali-infrastructure
description: |
  ali-ai infrastructure including launcher, migrations, hooks, scripts, and project structure. Use when:

  PLANNING: Designing new hooks, migrations, or automation features for ali-ai infrastructure
  IMPLEMENTATION: Creating hooks, writing migrations, updating launcher scripts, or modifying project setup scripts
  GUIDANCE: Asking how ali-ai infrastructure works, authentication flow, sync mechanisms, or migration best practices
  REVIEW: Auditing infrastructure scripts, validating migration idempotency, or checking hook implementation

  Do NOT use for:
  - General Claude Code usage (use ali-claude-code instead)
  - MCP server development (use ali-mcp instead)
  - Git operations and workflows (use github-admin skill instead)
  - Project architecture decisions (use project-specific expert agents)
---

<!-- Traceability convention: HTML comments (source-doc/related-skill) mark cross-system references between RUNBOOK.md and this skill. Prose callouts navigate within this skill to reference files. -->

# ali-ai Infrastructure

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing new hooks for validation or automation
- Planning migration strategies for organizational updates
- Architecting changes to the launcher or sync mechanisms
- Evaluating infrastructure improvements

**Implementation:**
- Creating new hooks for PreToolUse, PostToolUse, or other events
- Writing migration scripts for resource transformations
- Modifying claude-ali launcher behavior
- Updating project initialization scripts
- Building agent manifest compilation
- Setting up new team members

**Guidance/Best Practices:**
- Asking how the claude-ali launcher works
- Understanding authentication and subscription isolation
- Learning how migrations execute and track state
- Understanding the hook system and environment variables
- Clarifying directory structure and sync patterns
- Setting up development environments for new team members
- Finding relevant scripts for specific tasks

**Review/Validation:**
- Auditing hook implementations for safety
- Validating migration idempotency
- Reviewing launcher changes for subscription isolation
- Checking project setup scripts for correctness
- Verifying expert agent evidence requirements

---

## Key Principles

- **copy-on-launch:** Org resources rsync from ~/ali-ai to .claude/ at launch (not edit-in-place)
- **Subscription isolation:** CLAUDE_CONFIG_DIR controls billing (.claude-config/ = org, ~/.config/claude-personal/ = personal)
- **Migrations are idempotent:** All migrations must be safe to run multiple times (check-before-act pattern)
- **Hooks exit codes matter:** 0 = allow, 2 = block with message, 1 = error in hook itself
- **Flat skill structure:** Claude Code only finds skills one level deep (skill-name/SKILL.md)
- **Project marker determines sync:** .aliunde-project marker enables org resource sync
- **Source of truth:** ~/ali-ai/ is source, .claude/ is runtime copy (NEVER edit .claude/ directly)
- **Credential isolation per project:** .claude-config/ enables multiple Claude sessions with different accounts; gh CLI OAuth credentials isolated to .claude-config/gh/ via configure_gh_cli() (primary flow: gh auth login --web). See launcher-internals.md for the three-path credential decision tree.

---

## claude-ali Launcher (Quick Reference)

### What It Does

Unified entry point for launching Claude Code with subscription isolation and org resource sync.

**Location:** ~/.claude/bin/claude-ali

**Four-step launch:** Validate environment → Sync resources → Setup hooks → Launch Claude

### Three Operation Modes

| Mode | Command | Subscription | Resources | Use Case |
|------|---------|--------------|-----------|----------|
| Org | claude-ali | Organization (.claude-config/) | Synced from ~/ali-ai | Company work |
| Personal Isolated | claude-ali --personal | Personal (~/.config/claude-personal/) | None (isolated) | Personal projects |
| Personal + Org | claude-ali --personal --sync-org | Personal (~/.config/claude-personal/) | Synced from ~/ali-ai | Leverage org tools, bill personal |

### Authentication and Subscription

**CLAUDE_CONFIG_DIR determines billing:**
- Org mode: `.claude-config/` (company subscription)
- Personal mode: `~/.config/claude-personal/` (personal subscription)

**ANTHROPIC_API_KEY explicitly unset** to force subscription-based OAuth login.

**Credential isolation context:**
- Host machine: Uses OAuth from .claude-config/
- Docker container: Must pass ANTHROPIC_API_KEY explicitly
- Multiple sessions: Different .claude-config/ directories allow simultaneous sessions

For detailed execution flow, consult **references/launcher-internals.md** covering:
- Step-by-step launch sequence with error handling
- Hash-based change detection (~6ms)
- Remote update caching (30min)
- Hook configuration merge logic
- Performance optimizations
- gh CLI credential isolation (three-path decision tree)

---

## Migration Framework (Quick Reference)

### Purpose

Automated zero-touch organizational updates when ali-ai changes require transformation of existing project resources.

### How It Works

```
1. Migrations stored in:      ~/ali-ai/migrations/NNN-slug.sh
2. Executed by:                claude-ali launcher (auto on launch)
3. Registry tracked per-project: .claude/.migration-registry.json
4. Stop-on-failure:            First failure stops remaining migrations
```

### When to Create Migrations

**Create migrations for:**
- Resource renames (skill ali- prefix, hook renames)
- Deprecation cleanup (removing obsolete files)
- Configuration schema evolution (settings.json version updates)
- New required files (adding mandatory resources)

**Don't create migrations for:**
- New features that sync automatically via rsync
- Documentation updates (no file impact)

### Idempotency Requirement

**Every migration MUST be idempotent (safe to run multiple times).**

Techniques:
- Check before act: Verify resource exists before removing
- Skip if complete: Detect already-migrated state and exit 0
- Use -f flags: rm -f, cp -f (don't fail if missing)

For detailed migration development, consult **references/migration-framework.md** covering:
- Full migration template with comments
- Six idempotency patterns with examples
- Registry format and operations
- Testing procedures and automation
- Example migrations (skill cleanup, settings update, directory restructure)

---

## Hook System (Quick Reference)

### Hook Events

| Event | When | Use Case |
|-------|------|----------|
| SessionStart | Session begins | Display warnings, load context |
| PreToolUse | Before tool execution | Validation, blocking dangerous operations |
| PostToolUse | After tool completion | Formatting, logging, notifications |
| UserPromptSubmit | Before processing prompt | Pre-processing, content warnings |
| Stop | When Claude finishes | Cleanup, final reports |

### Hook Exit Codes

| Code | Effect | Usage |
|------|--------|-------|
| 0 | Allow execution | Validation passed, safe operation |
| 2 | Block execution | Show error to Claude, don't execute |
| 1 | Error in hook | Hook itself failed, report to user |

### JSON Output Examples

```bash
# Allow
echo '{"allow": true}'

# Block with reason and suggestion
echo '{
  "allow": false,
  "explanation": "Force push to main prohibited",
  "appendMessage": "Use /github-admin skill for git operations"
}'
```

### Example: ali-git-guard.sh

Enforces git/gh command safety and delegates to github-admin skill:
1. ALWAYS BLOCKED: force push to main, hard reset, git clean -f
2. REQUIRE CONFIRMATION: force push to branches, branch -D
3. ALWAYS ALLOWED: status, log, diff, show, fetch
4. SUGGEST DELEGATION: add, commit, push → /github-admin

For detailed hook development, consult **references/hook-system.md** covering:
- Complete environment variable reference
- Advanced JSON output patterns
- Four production hook examples (git-guard, file-protect, auto-format, session-protocol)
- Comprehensive hook creation template with checklist
- Testing patterns and debugging procedures

---

## Scripts Inventory (Quick Reference)

### Core Scripts Summary

| Category | Key Scripts | Purpose |
|----------|-------------|---------|
| **Launcher & Sync** | claude-ali, sync-org-resources.sh | Launch Claude with subscription isolation, sync org resources |
| **Project Setup** | init-aliunde-project.sh, setup_dev_environment.sh, setup-colleague.sh | Initialize projects, setup dev environments, onboard colleagues |
| **Agent & Manifest** | build_agent_manifest.py, list_experts.py, regenerate-agents.sh | Build agent manifests, score experts, validate agents |
| **Migration** | run-pending-migrations.sh | Execute pending migrations |
| **Cleanup & Validation** | cleanup-old-skills.sh, verify-expert-evidence.sh | Remove deprecated skills, verify expert compliance |

### Most Common Commands

```bash
# Launch Claude Code
claude-ali                              # Org mode
claude-ali --personal                   # Personal isolated
claude-ali --force-sync                 # Force re-sync

# Setup environment (new team member)
~/.claude/bin/setup_dev_environment.sh

# Initialize project
~/.claude/bin/init-aliunde-project.sh

# Rebuild agent manifest
cd ~/ali-ai
python3 scripts/build_agent_manifest.py
```

For complete scripts documentation, consult **references/scripts-inventory.md** covering:
- Full usage, options, examples for each script
- Exit codes and error handling
- Implementation notes and dependencies
- Performance characteristics
- When to use each script

---

## Project Structure (Quick Reference)

### Directory Layout

```
project/
├── .aliunde-project              # Marker (enables org sync)
├── CLAUDE.md                     # Project instructions (git-tracked)
├── .env                          # Service credentials (gitignored)
│
├── .claude/                      # Org resources (auto-synced, gitignored)
│   ├── .last-sync                # Git hash of last sync
│   ├── agents/                   # From ~/ali-ai/agents/ (ali-*.md)
│   ├── skills/                   # From ~/ali-ai/skills/ (ali-*/SKILL.md)
│   ├── hooks/                    # From ~/ali-ai/hooks/ (ali-*.sh)
│   ├── logs/                     # Migration and hook logs
│   └── .migration-registry.json  # Completed migrations tracking
│
├── .claude-local/                # Project-specific (git-tracked)
│   ├── agents/                   # Project-only agents
│   ├── skills/                   # Project-only skills
│   └── hooks/                    # Project-only hooks
│
└── .claude-config/               # Credentials (gitignored, org subscription)
    ├── settings.json             # Hooks configuration (merged from template)
    └── gh/                       # gh CLI OAuth credentials (per-project)
        └── hosts.yml             # gh auth tokens (managed by configure_gh_cli)
```

### .claude/ vs .claude-local/ vs .claude-config/

| Aspect | .claude/ | .claude-local/ | .claude-config/ |
|--------|----------|----------------|-----------------|
| Source | Synced from ~/ali-ai | Created locally | Created by setup script |
| Git | Gitignored | Git-tracked | Gitignored |
| Purpose | Org-wide resources | Project-specific resources | Credentials isolation |
| Lifecycle | Overwritten each launch | Persistent, overlayed | Persistent, per-session |
| Naming | ali- prefix | No prefix required | N/A |

---

## Troubleshooting (Quick Checks)

### Migration Failed - Recovery

1. Check logs: `cat .claude/logs/migrations.log`
2. Identify what failed and why
3. Fix the issue (e.g., missing dependency, permission error)
4. Re-run claude-ali (migration will retry automatically)
5. If migration is broken, fix in ~/ali-ai/migrations/ and git pull

### Resources Not Syncing

**Checklist:**
1. Is .aliunde-project marker present?
2. Is ~/.ali-ai git repo up to date?
3. Run with --force-sync: `claude-ali --force-sync`
4. Check sync hash: `cat .claude/.last-sync`
5. Check rsync errors in terminal output

### Subscription Billing Wrong Account

**Fix:**
1. Check CLAUDE_CONFIG_DIR: `echo $CLAUDE_CONFIG_DIR`
2. For personal work: `claude-ali --personal`
3. For company work: `claude-ali` (from Aliunde project)
4. Verify .claude-config/ vs ~/.config/claude-personal/

For detailed troubleshooting, consult **references/troubleshooting-detailed.md** covering:
- Hook blocking operations incorrectly (debugging procedure)
- Hooks not executing (complete diagnostic checklist)
- Docker testing vs host machine (credential patterns)

---

## Common Tasks (Step-by-Step)

### Setting Up New Team Member

1. Clone repository
2. Run `~/.claude/bin/setup_dev_environment.sh`
3. Follow prompts (Claude config, Git, services, Python)
4. Activate environment: `source .venv/bin/activate`
5. Launch Claude: `claude-ali`

### Creating a New Migration

1. Determine number: `ls ~/ali-ai/migrations/*.sh | tail -1`
2. Copy template: `cp MIGRATION_TEMPLATE.sh NNN-slug.sh`
3. Fill in details (ID, description, idempotency check)
4. Test locally: `./migrations/NNN-slug.sh ~/ali-ai`
5. Test idempotency (run twice)
6. Commit: `git add migrations/NNN-slug.sh && git commit`

### Creating a New Hook

1. Choose type (PreToolUse, PostToolUse, SessionStart, etc.)
2. Create file: `touch ~/ali-ai/hooks/ali-name.sh && chmod +x`
3. Implement logic (validation, blocking, formatting)
4. Test manually: `export TOOL_INPUT='...' && ./hooks/ali-name.sh`
5. Update templates/settings.json (add hook, bump version)
6. Create migration to deploy hook
7. Test end-to-end in project

For complete procedures, consult **references/common-tasks.md** covering:
- Setting up new team member environment (full walkthrough)
- Creating a new migration (10-step procedure with testing checklist)
- Creating a new hook (10-step procedure with integration)
- Adding a new script (12-step procedure with documentation)

---

## Anti-Patterns

| Anti-Pattern | Why Bad | Do Instead |
|--------------|---------|------------|
| Edit .claude/ files directly | Overwritten on next sync | Edit source in ~/ali-ai/ or create in .claude-local/ |
| Non-idempotent migrations | Fails on retry, corrupts state | Always check-before-act, exit 0 if already done |
| Hook exits 1 for blocked operation | Claude treats as error, confuses user | Exit 0 with {"allow": false, "reason": "..."} |
| Missing executable permissions | Hook fails silently | Always chmod +x after creating |
| Hardcoded paths in scripts | Breaks on other machines | Use variables, accept arguments |
| Skipping migration testing | Breaks projects in production | Test on 2+ projects before commit |
| Nested skill directories | Claude won't find them | Use flat structure: skill-name/SKILL.md |
| No version bump after hook add | Projects don't get new hook | Bump template version, create migration |
| Forgetting to rebuild manifest | Expert changes not visible | Run build_agent_manifest.py after agent edits |
| Use global ~/.claude/ for all projects | Can't run multiple accounts simultaneously | Use .claude-config/ per project |
| Manually editing .claude-config/gh/hosts.yml | Bypasses account verification, may corrupt credentials | Re-run setup_dev_environment.sh |

---

## Quick Reference

### Directory Quick Paths

```bash
# Source of truth
~/ali-ai/agents/              # Agent definitions
~/ali-ai/skills/              # Skill definitions
~/ali-ai/hooks/               # Hook implementations
~/ali-ai/migrations/          # Migration scripts
~/ali-ai/templates/           # settings.json template

# Project runtime
.claude/                      # Auto-synced org resources
.claude-local/                # Project-specific resources
.claude-config/               # Subscription credentials (isolated)
.claude-config/gh/            # gh CLI OAuth credentials (per-project)
.claude/logs/                 # Migration and hook logs
.env                          # Service credentials (gitignored)
```

### Common Commands

<!-- source-doc: RUNBOOK.md — Test Infrastructure, Common Commands -->

```bash
# Launch Claude Code
claude-ali                              # Org mode
claude-ali --personal                   # Personal isolated
claude-ali --personal --sync-org        # Personal + org tools
claude-ali --force-sync                 # Force re-sync

# Setup environment
~/.claude/bin/setup_dev_environment.sh

# Initialize project
~/.claude/bin/init-aliunde-project.sh --dry-run

# Test migration
~/ali-ai/migrations/NNN-slug.sh /path/to/project

# Rebuild agent manifest
cd ~/ali-ai && python3 scripts/build_agent_manifest.py

# Run tests (no Docker, no auth required — fast feedback during development)
make test-unit

# Run all Docker structural tests (no auth needed)
make test-structural

# Check migration status
cat .claude/.migration-registry.json | jq '.migrations'
tail -50 .claude/logs/migrations.log

# View recent log files
tail -50 ~/.claude/logs/session-protocol.log
tail -50 ~/.claude/logs/pre-push-review.log
tail -50 .claude/logs/migrations.log

# Retire a skill (remove from org source)
cd ~/ali-ai/skills && rm -rf old-skill-name/
# Then: git add -A && git commit -m "chore(skills): Remove old-skill (replaced by new-skill)"
```

---

## Detailed Reference

For comprehensive documentation, see:

### Launcher Internals
**File:** `references/launcher-internals.md`

Detailed execution flow, credential isolation implementation, OAuth patterns, multi-account setup, performance optimizations, security considerations, gh CLI credential isolation (configure_gh_cli three-path decision tree).

### Migration Framework
**File:** `references/migration-framework.md`

Full migration template, idempotency patterns (6 types), registry operations, testing procedures, example migrations (skill cleanup, settings update, directory restructure). Includes migration status check commands.

### Hook System
**File:** `references/hook-system.md`

Complete environment variable reference, advanced JSON output patterns, production hook examples (git-guard, file-protect, auto-format, session-protocol), comprehensive creation template, testing and debugging. Includes inline pipe pattern for quick hook testing.

### Scripts Inventory
**File:** `references/scripts-inventory.md`

Full documentation for all scripts with usage, options, examples, exit codes, implementation notes, dependencies, performance characteristics, when to use.

### Common Tasks
**File:** `references/common-tasks.md`

Step-by-step procedures for setting up new team members, creating migrations, creating hooks, adding scripts. Includes testing checklists and troubleshooting. Also covers colleague onboarding via setup-colleague.sh and docker-login interactivity requirements.

### Detailed Troubleshooting
**File:** `references/troubleshooting-detailed.md`

In-depth debugging for hook issues (blocking incorrectly, not executing), Docker vs host machine patterns, credential management, security notes. Covers skill-not-triggering, agent-not-found, hook-not-firing, and emergency procedures.

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| Launcher | ~/.claude/bin/claude-ali |
| Project setup | ~/.claude/bin/init-aliunde-project.sh |
| Environment setup | ~/.claude/bin/setup_dev_environment.sh |
| Colleague setup | ~/.claude/bin/setup-colleague.sh |
| Migration runner | ~/.claude/bin/run-pending-migrations.sh |
| Agent manifest builder | ~/.claude/bin/build_agent_manifest.py |
| Expert scorer | ~/.claude/bin/list_experts.py |
| Hooks template | ~/ali-ai/templates/settings.json |
| Git guard | ~/ali-ai/hooks/ali-git-guard.sh |
| Migration template | ~/ali-ai/migrations/MIGRATION_TEMPLATE.sh |

---

## References

- [Copy-on-Launch Architecture](~/ali-ai/docs/plans/copy-on-launch-architecture.md)
- [Migration Developer Guide](~/ali-ai/migrations/README.md)
- [Agent Creation Guide](~/.claude/docs/AGENT_CREATION_GUIDE.md)
- [Claude Code Hooks Documentation](https://docs.anthropic.com/claude-code/hooks)
- [Claude Code Settings Reference](https://docs.anthropic.com/claude-code/settings)

---

**Document Version:** 2.2 (Session 19 — gh CLI credential isolation, configure_gh_cli three-path flow)
**Last Updated:** 2026-02-28
**Maintained By:** ALI AI Team
