# ali-infrastructure - Common Tasks Reference

This document provides step-by-step procedures for common ali-ai infrastructure tasks.

---

## Setting Up New Team Member Environment

**Context:** New team member joining the project needs development environment configured.

### Prerequisites
- Access to project repository
- GitHub account with collaborator access
- Internet connection

### Step-by-Step Procedure

**Step 1: Clone repository**
```bash
# Get repository URL from team lead
git clone <repo-url>
cd <project-name>
```

**Step 2: Run setup script**
```bash
~/.claude/bin/setup_dev_environment.sh
```

**Step 3: Follow interactive prompts**

The script will guide you through:

1. **Claude credential isolation (always configured)**
   - Creates .claude-config/ directory
   - Answer: Press Enter to accept defaults

2. **Git/GitHub configuration (optional)**
   - Prompt: "Configure Git for this project? (y/n)"
   - Answer: `y` if you need per-project Git identity
   - Enter your git user.name: "Your Name"
   - Enter your git user.email: "your.email@company.com"
   - SSH key generation is available but optional — HTTPS transport via gh credential helper is the default

3. **gh CLI authentication (automatic when Git is configured)**

   If you configured Git above, setup_dev_environment.sh will call configure_gh_cli() to establish gh credentials. This runs automatically — you do not invoke it separately.

   What happens:
   - A browser window opens for GitHub OAuth login (`gh auth login --web`)
   - Sign in as your GitHub account when prompted
   - After login, setup verifies the authenticated account matches the expected account
   - `gh auth setup-git` is called automatically, registering the gh credential helper for HTTPS git transport
   - `git push` and `git pull` will work without SSH keys — authentication uses the OAuth token
   - Credentials are stored at .claude-config/gh/hosts.yml (per-project, gitignored)
   - .claude-config/ is automatically added to .gitignore if not already present

   If the wrong account authenticates:
   - Setup will detect the mismatch automatically
   - .claude-config/gh/ is removed
   - Re-run setup_dev_environment.sh — it will prompt for login again

   On subsequent launches:
   - Path 1 (fast path): If .claude-config/gh/hosts.yml exists and gh auth status passes, login is skipped entirely — no browser needed

4. **Service credentials (optional)**
   - Prompt: "Configure service credentials? (y/n)"
   - Answer: `y` if project needs Snowflake, AWS, or CloudFlare
   - Enter credentials when prompted (get from team lead)

5. **Python virtual environment (optional)**
   - Prompt: "Create Python virtual environment? (y/n)"
   - Answer: `y` if project uses Python
   - Creates .venv/ directory

**Step 4: Activate environment (if Python)**
```bash
source .venv/bin/activate
pip install -r requirements.txt
```

**Step 5: Verify setup**
```bash
# Check Claude config exists
ls -la .claude-config/

# Check gh credentials (if Git was configured)
GH_CONFIG_DIR=.claude-config/gh/ gh auth status --hostname github.com
# Should show: Logged in to github.com as <your-username>

# Check Git config (if configured)
git config --local user.name
git config --local user.email

# Check service credentials (if configured)
cat .env  # Should contain credentials
```

**Step 6: Launch Claude**
```bash
claude-ali
```

On first launch:
- Claude will prompt for OAuth login
- Use organization account credentials
- OAuth tokens stored in .claude-config/credentials.json
- Subsequent launches won't require login

**Result:**
- ✓ .claude-config/ created for isolated credentials
- ✓ .claude-config/gh/hosts.yml created for gh CLI (per-project)
- ✓ .claude-config/ added to .gitignore (credential exposure prevention)
- ✓ gh credential helper registered for HTTPS git transport (git push/pull work without SSH keys)
- ✓ .env file created with service credentials
- ✓ Git configured with user info (SSH key optional)
- ✓ Ready to start work

### Troubleshooting

**Problem: setup_dev_environment.sh not found**
```bash
# Ensure ali-ai is set up
curl -sSL https://raw.githubusercontent.com/dmccarty-ali/ali-ai/main/scripts/setup-colleague.sh | bash
```

**Problem: SSH key not working with GitHub**
```bash
# Test SSH connection
ssh -T git@github.com -i ~/.ssh/id_ed25519

# Should see: "Hi username! You've successfully authenticated"
# If not, check key is added to GitHub
```

**Problem: Claude prompts for login every time**
```bash
# Check CLAUDE_CONFIG_DIR is set correctly
echo $CLAUDE_CONFIG_DIR
# Should be: /path/to/project/.claude-config

# Check credentials file exists
ls -la .claude-config/credentials.json

# If missing, OAuth didn't complete - re-launch and complete login
```

**Problem: gh auth fails or wrong account authenticated**
```bash
# Do NOT manually edit .claude-config/gh/hosts.yml
# Re-run setup — configure_gh_cli() will detect the invalid state and re-authenticate
~/.claude/bin/setup_dev_environment.sh

# If you need to force a fresh login immediately:
rm -rf .claude-config/gh/
~/.claude/bin/setup_dev_environment.sh
```

---

## Creating a New Migration

**Context:** ali-ai infrastructure change requires transformation of existing project resources.

### When to Create Migrations
- Resource renames (skills, hooks, agents)
- Deprecation cleanup (removing obsolete files)
- Configuration schema evolution (updating file formats)
- New required files (adding mandatory resources)
- Permission corrections (fixing file modes)

### Step-by-Step Procedure

**Step 1: Determine migration number**
```bash
cd ~/ali-ai/migrations
ls -1 *.sh | tail -1
# Example output: 007-some-migration.sh
# Next number: 008
```

**Step 2: Copy template**
```bash
cp MIGRATION_TEMPLATE.sh 008-your-descriptive-slug.sh
chmod +x 008-your-descriptive-slug.sh
```

**Step 3: Edit migration metadata**
```bash
vim 008-your-descriptive-slug.sh
```

Update:
```bash
# Migration: 008-your-descriptive-slug
MIGRATION_ID="008-your-descriptive-slug"
# Description: One-line description of transformation
# Created: 2026-02-16
# Idempotent: Yes  # MUST be Yes
```

**Step 4: Implement validation logic**

Check environment is suitable for migration:
```bash
# Example: Validate target directory exists
SKILLS_DIR="$PROJECT_DIR/.claude/skills"
if [[ ! -d "$SKILLS_DIR" ]]; then
    echo "SKIP: No .claude/skills/ directory found"
    exit 0
fi
```

**Step 5: Implement idempotency check**

Detect if already migrated:
```bash
# Strategy 1: Check if old resource no longer exists
if [[ ! -d "$SKILLS_DIR/old-skill-name" ]]; then
    echo "SKIP: Migration already complete"
    exit 0
fi

# Strategy 2: Check if new marker exists
if [[ -f "$PROJECT_DIR/.claude/.migration-marker-008" ]]; then
    echo "SKIP: Migration already complete"
    exit 0
fi
```

**Step 6: Implement migration logic**

Perform the transformation:
```bash
echo "Starting migration: $MIGRATION_ID"

# Remove old skill
if [[ -d "$SKILLS_DIR/old-skill-name" ]]; then
    echo "  Removing: old-skill-name"
    rm -rf "$SKILLS_DIR/old-skill-name"
fi

# Create marker (for future idempotency checks)
touch "$PROJECT_DIR/.claude/.migration-marker-008"
```

**Step 7: Implement verification**

Verify expected post-migration state:
```bash
# Verify old resource removed
if [[ -d "$SKILLS_DIR/old-skill-name" ]]; then
    echo "ERROR: Migration verification failed"
    exit 1
fi

echo "Migration complete: $MIGRATION_ID"
exit 0
```

**Step 8: Test locally**

Test in safe environment first:
```bash
# Test in ali-ai itself (safe)
./migrations/008-your-slug.sh ~/ali-ai
# Should see: "SKIP: Migration already complete" or "Migration complete"

# Test in real project
cd ~/ali-ai/projects/test-project
~/ali-ai/migrations/008-your-slug.sh .
# Should see: "Migration complete"

# Test idempotency (run twice)
~/ali-ai/migrations/008-your-slug.sh .
# Second run should SKIP

# Check registry was updated
cat .claude/.migration-registry.json | jq '.migrations[] | select(.id=="008-your-slug")'
```

**Step 9: Test in multiple projects**

Ensure migration works across different project states:
```bash
# Test in project with old state
cd ~/ali-ai/projects/project-with-old-skills
~/ali-ai/migrations/008-your-slug.sh .

# Test in project already migrated
cd ~/ali-ai/projects/project-already-migrated
~/ali-ai/migrations/008-your-slug.sh .
# Should SKIP

# Test in project without .claude/ directory
cd ~/ali-ai/projects/project-not-initialized
~/ali-ai/migrations/008-your-slug.sh .
# Should SKIP gracefully
```

**Step 10: Document migration**

Add entry to ~/ali-ai/migrations/README.md:
```markdown
### 008-your-descriptive-slug
**Purpose:** Brief description
**Created:** 2026-02-16
**Affects:** .claude/skills/
**Idempotent:** Yes
```

**Step 11: Commit**
```bash
cd ~/ali-ai
git add migrations/008-your-slug.sh
git add migrations/README.md  # If updated
git commit -m "feat(migrations): add 008-your-slug migration

Removes old unprefixed skill directories from .claude/skills/
Idempotent: checks for existence before removal"
git push origin main
```

**Step 12: Communicate to team**

Notify team in Slack/email:
```
New migration deployed: 008-your-slug

What it does: [description]
When it runs: Automatically on next claude-ali launch
Action required: None (automatic)

Details: https://github.com/dmccarty-ali/ali-ai/blob/main/migrations/008-your-slug.sh
```

### Testing Checklist

Before committing migration:
- [ ] Runs successfully on clean project
- [ ] Runs successfully on already-migrated project (SKIP)
- [ ] Runs successfully on project without target resources (SKIP)
- [ ] Updates registry correctly
- [ ] Logs to migrations.log
- [ ] Idempotent (safe to run multiple times)
- [ ] Exit code 0 on success
- [ ] Exit code 1 only on actual failure (not SKIP)

---

## Creating a New Hook

**Context:** Need to add validation, automation, or enforcement to Claude Code lifecycle.

### Hook Types
- **PreToolUse:** Validation before tool execution (blocking)
- **PostToolUse:** Formatting/logging after completion (non-blocking)
- **SessionStart:** Context loading at session start
- **UserPromptSubmit:** Pre-processing user input
- **Stop:** Cleanup at session end

### Step-by-Step Procedure

**Step 1: Choose hook type**

Based on use case:
- Need to **block** dangerous operations? → PreToolUse
- Need to **format** files after edit? → PostToolUse
- Need to **enforce** session protocol? → SessionStart
- Need to **filter** user input? → UserPromptSubmit
- Need to **cleanup** at end? → Stop

**Step 2: Create hook file**
```bash
cd ~/ali-ai/hooks
touch ali-your-descriptive-name.sh
chmod +x ali-your-descriptive-name.sh
```

**Step 3: Copy hook template**
```bash
# From references/hook-system.md
# Copy "Hook Creation Template (Comprehensive)" section
```

**Step 4: Implement hook logic**

Set bash options and environment:
```bash
#!/bin/bash
# Hook: ali-your-name
# Purpose: What this hook does
# Event: PreToolUse
# Matcher: Bash

set -euo pipefail

TOOL_NAME="${TOOL_NAME:-unknown}"
TOOL_INPUT="${TOOL_INPUT:-{}}"
```

Extract data from TOOL_INPUT:
```bash
# For Bash commands
COMMAND=$(echo "$TOOL_INPUT" | jq -r '.command // empty' 2>/dev/null || echo "")

# For file operations
FILE_PATH=$(echo "$TOOL_INPUT" | jq -r '.file_path // empty' 2>/dev/null || echo "")
```

Implement validation/transformation:
```bash
# Example: Block dangerous commands
if [[ "$COMMAND" =~ dangerous-pattern ]]; then
    echo '{
      "allow": false,
      "explanation": "Blocked because...",
      "appendMessage": "🚫 This operation is not allowed..."
    }'
    exit 0
fi

# Allow safe operations
echo '{"allow": true}'
exit 0
```

**Step 5: Test hook manually**
```bash
# Set environment variables
export TOOL_NAME="Bash"
export TOOL_INPUT='{"command":"git push --force origin main"}'

# Run hook
./hooks/ali-your-hook.sh

# Expected output: JSON response
# {"allow": false, "explanation": "...", "appendMessage": "..."}

# Test safe operation
export TOOL_INPUT='{"command":"git status"}'
./hooks/ali-your-hook.sh
# Expected: {"allow": true}

# Unset
unset TOOL_NAME TOOL_INPUT
```

**Step 6: Update settings template**
```bash
cd ~/ali-ai/templates
vim settings.json
```

Add hook to appropriate event:
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/ali-your-hook.sh"
          }
        ]
      }
    ]
  }
}
```

**Step 7: Bump template version**
```json
{
  "version": "1.2",  // Increment minor version (was 1.1)
  "hooks": {
    // ... hooks including your new one
  }
}
```

**Step 8: Create migration to deploy hook**

Create migration to merge new hook into existing projects:
```bash
cd ~/ali-ai/migrations
cp MIGRATION_TEMPLATE.sh 009-add-your-hook.sh
chmod +x 009-add-your-hook.sh
```

Implement migration (similar to 003-settings-version-update.sh):
```bash
# Check current version
CURRENT_VERSION=$(jq -r '.version // "1.1"' "$SETTINGS")

if [[ "$CURRENT_VERSION" == "1.2" ]]; then
    echo "SKIP: Already at version 1.2"
    exit 0
fi

# Merge template
jq -s '.[0] * .[1]' "$SETTINGS" "$TEMPLATE" > "$SETTINGS.tmp"
jq '.version = "1.2"' "$SETTINGS.tmp" > "$SETTINGS"
```

**Step 9: Test end-to-end**
```bash
# In test project
cd ~/ali-ai/projects/test-project

# Force sync to get new hook
~/.claude/bin/claude-ali --force-sync

# Verify hook in settings
cat .claude-config/settings.json | jq '.hooks.PreToolUse'

# Launch Claude and test hook triggers
claude-ali
# Try operation that should trigger hook
# Verify hook blocks/allows as expected
```

**Step 10: Commit**
```bash
cd ~/ali-ai

git add hooks/ali-your-hook.sh
git add templates/settings.json
git add migrations/009-add-your-hook.sh
git commit -m "feat(hooks): add ali-your-hook for [purpose]

- Blocks dangerous [operation type]
- Template version: 1.1 → 1.2
- Migration 009 deploys to existing projects"

git push origin main
```

### Hook Testing Checklist

Before committing:
- [ ] Manual test with sample TOOL_INPUT
- [ ] Handles missing TOOL_INPUT gracefully
- [ ] Exits 0 for all outcomes (even when blocking)
- [ ] Returns valid JSON
- [ ] Logs to appropriate log file (if applicable)
- [ ] Is executable (chmod +x)
- [ ] Template updated with correct matcher
- [ ] Template version bumped
- [ ] Migration created to deploy
- [ ] End-to-end test in real project

---

## Adding a New Script

**Context:** Need to create a new infrastructure script for automation or tooling.

### Step-by-Step Procedure

**Step 1: Determine script purpose and location**

Script types:
- **Launcher/Sync:** scripts/ directory (e.g., claude-ali, sync-org-resources.sh)
- **Project Setup:** scripts/ directory (e.g., init-aliunde-project.sh)
- **Agent/Manifest:** scripts/ directory (e.g., build_agent_manifest.py)
- **Migration:** migrations/ directory (see "Creating a New Migration")
- **Cleanup/Validation:** scripts/ directory (e.g., verify-expert-evidence.sh)

**Step 2: Create script**
```bash
cd ~/ali-ai/scripts
touch your-descriptive-script.sh
chmod +x your-descriptive-script.sh
```

**Step 3: Add script header**
```bash
#!/bin/bash
# your-script.sh
# Brief one-line description
#
# Location: ~/.claude/bin/your-script.sh
# Purpose: Detailed purpose (2-3 sentences)
#
# Usage:
#   your-script.sh [OPTIONS] [ARGUMENTS]
#
# Options:
#   --option1    Description
#   --help       Display help message
#
# Examples:
#   your-script.sh --option1 value
#   your-script.sh /path/to/project
```

**Step 4: Set bash options**
```bash
set -euo pipefail

# Error handling
error() {
    echo "ERROR: $*" >&2
    exit 1
}
```

**Step 5: Define constants**
```bash
# Constants
SCRIPT_NAME="your-script"
DEFAULT_VALUE="default"
LOG_FILE="$HOME/.claude/logs/${SCRIPT_NAME}.log"
```

**Step 6: Define helper functions**
```bash
# Logging helper
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

# Validation helper
validate_directory() {
    local dir="$1"
    if [[ ! -d "$dir" ]]; then
        error "Directory not found: $dir"
    fi
}
```

**Step 7: Implement argument parsing**
```bash
# Parse arguments
while [[ $# -gt 0 ]]; do
    case "$1" in
        --option1)
            OPTION1="$2"
            shift 2
            ;;
        --help)
            show_help
            exit 0
            ;;
        *)
            PROJECT_DIR="$1"
            shift
            ;;
    esac
done
```

**Step 8: Implement main logic**
```bash
# Main function
main() {
    log "Starting: $SCRIPT_NAME"

    # Validate inputs
    validate_directory "$PROJECT_DIR"

    # Perform operations
    # ...

    log "Complete: $SCRIPT_NAME"
}

# Run main
main "$@"
```

**Step 9: Document in skills**

Update skills/ali-infrastructure/references/scripts-inventory.md:
```markdown
### your-script.sh

**Location:** ~/.claude/bin/your-script.sh

**Purpose:** What this script does

**Usage:**
```bash
your-script.sh [OPTIONS] [ARGUMENTS]
```

**When to use:**
- Scenario 1
- Scenario 2
```

**Step 10: Test script**
```bash
# Test with sample inputs
~/ali-ai/scripts/your-script.sh --option1 value /path/to/test

# Test error handling
~/ali-ai/scripts/your-script.sh --option1 invalid
# Should error gracefully

# Test help message
~/ali-ai/scripts/your-script.sh --help
```

**Step 11: Integrate (if needed)**

If script needs to run automatically:

Add to claude-ali launcher:
```bash
# In claude-ali script
~/.claude/bin/your-script.sh "$PROJECT_DIR"
```

Or add to init-aliunde-project.sh:
```bash
# In init-aliunde-project.sh
~/.claude/bin/your-script.sh "$PROJECT_DIR"
```

**Step 12: Commit**
```bash
cd ~/ali-ai
git add scripts/your-script.sh
git add skills/ali-infrastructure/references/scripts-inventory.md
git commit -m "feat(scripts): add your-script for [purpose]

- Does X
- Usage: your-script.sh [options]
- Documented in scripts inventory"
git push origin main
```

### Script Development Checklist

Before committing:
- [ ] Has descriptive header with usage examples
- [ ] Sets bash options (set -euo pipefail)
- [ ] Has error handling (error function)
- [ ] Has constants section
- [ ] Has helper functions (if applicable)
- [ ] Has main function
- [ ] Accepts arguments via command line
- [ ] Has --help flag
- [ ] Logs to appropriate log file (if applicable)
- [ ] Is executable (chmod +x)
- [ ] Tested with valid inputs
- [ ] Tested with invalid inputs (error handling)
- [ ] Documented in scripts inventory

---

## Docker Login — Interactive Requirement

<!-- source-doc: RUNBOOK.md — Docker Credential Isolation section -->

**Critical constraint: `make docker-login` is interactive only.**

`docker-login` requires an active terminal with a human present. It cannot be scripted or run as part of an automated sequence.

**Why:** The authentication flow requires interactive input — it prompts for credentials and may require browser-based OAuth. The `-it` (interactive + TTY) flags are required in the underlying Docker command.

**Do this (in an active terminal):**

```bash
make docker-login
```

**Do NOT do this:**
- Running `make docker-login` from a script without a TTY
- Piping to `make docker-login`
- Running from CI/CD without interactive mode
- Running from inside a non-TTY subprocess

**After docker-login completes:**
- Credentials are stored in the `ali-ai-functional-creds` Docker volume
- The volume persists across runs — no re-auth needed for subsequent `make test-functional` runs
- To force re-auth: `make clean-docker-creds && make docker-login`

---

## Viewing Log Files

<!-- source-doc: RUNBOOK.md — Log Files section -->

Log files are written to `~/.claude/logs/` (global) and `.claude/logs/` (per-project).

| Log | Location | Purpose |
|-----|----------|---------|
| Session protocol | `~/.claude/logs/session-protocol.log` | Session start activity |
| Pre-push review | `~/.claude/logs/pre-push-review.log` | Review decisions |
| Pre-push audit | `~/.claude/logs/pre-push-audit.log` | Audit trail |
| Migration log | `.claude/logs/migrations.log` | Migration execution history |

**Viewing recent log entries:**

```bash
# Session protocol
tail -50 ~/.claude/logs/session-protocol.log

# Pre-push decisions
tail -50 ~/.claude/logs/pre-push-review.log

# Migrations (per-project)
tail -50 .claude/logs/migrations.log
```

---

## Retiring a Skill

<!-- source-doc: RUNBOOK.md — Maintenance section, "Retire a Skill" -->

When a skill is obsolete or replaced by a newer skill, remove it from the org source:

```bash
cd ~/ali-ai/skills
rm -rf old-skill-name/
git add -A
git commit -m "chore(skills): Remove old-skill-name (replaced by new-skill-name)"
git push
```

**What happens next:** At each team member's next `claude-ali` launch, the stale skill copy in `.claude/skills/` is removed by the sync process (rsync with --delete flag for skills).

**If a migration is needed** (e.g., skill was renamed and projects have both old and new copies), create a migration script instead of relying on rsync alone. See "Creating a New Migration" above.

---

## Colleague Onboarding — Quick Setup

<!-- source-doc: RUNBOOK.md — Colleague Onboarding section -->

For a new team member joining with the one-liner automated setup:

```bash
# One-liner: checks prerequisites, clones ali-ai, makes scripts executable, adds alias
curl -sSL https://raw.githubusercontent.com/dmccarty-ali/ali-ai/main/scripts/setup-colleague.sh | bash

# After running, activate the alias
source ~/.zshrc  # or ~/.bashrc

# Launch in any Aliunde project
claude-ali
```

**First launch behavior:**
- Syncs all org resources (agents, skills, hooks) to project `.claude/`
- Configures hooks in settings.json automatically
- Prompts for subscription OAuth login

**Operation modes:**

| Command | Subscription | Resources | Use Case |
|---------|-------------|-----------|----------|
| `claude-ali` | Org | Org resources | Company work |
| `claude-ali --personal` | Personal | None (isolated) | Personal projects |
| `claude-ali --personal --sync-org` | Personal | Org resources | Personal with org tools |

**Manual setup** (if curl one-liner is not preferred): see RUNBOOK.md Colleague Onboarding section for step-by-step instructions.

---

**Last Updated:** 2026-02-28 (Phase 6 — HTTPS transport default, gh auth setup-git, SSH key registration no longer required)
**Maintained By:** ALI AI Team
