# ali-infrastructure - Hook System Reference

This document provides comprehensive documentation for the ali-ai hook system, including detailed environment variables, advanced JSON output patterns, complete hook examples, and creation templates.

---

## Hook System Overview

Hooks enable validation, automation, and enforcement at specific points in the Claude Code lifecycle. The ali-ai infrastructure provides standardized hooks and patterns for organizational consistency.

### Hook Lifecycle

```
User Action → Claude Decision → Hook Intercept → Allow/Block → Execution
```

---

## Hook Environment Variables (Detailed)

### PreToolUse and PostToolUse Hooks

| Variable | Available | Type | Example | Notes |
|----------|-----------|------|---------|-------|
| TOOL_NAME | Both | string | "Bash", "Edit", "Write", "Read" | Tool being invoked |
| TOOL_INPUT | Both | JSON string | '{"command":"git push"}' | Tool parameters as JSON |
| FILE_PATH | PostToolUse only | string | "/path/to/file.py" | File modified (Edit/Write only) |
| COMMAND | Both (if extracted) | string | "git push origin main" | Bash command (if extracted from TOOL_INPUT) |

### Extracting Data from TOOL_INPUT

```bash
#!/bin/bash
# Hook template with TOOL_INPUT extraction

TOOL_NAME="${TOOL_NAME:-unknown}"
TOOL_INPUT="${TOOL_INPUT:-{}}"

# Extract command (for Bash tool)
COMMAND=$(echo "$TOOL_INPUT" | jq -r '.command // empty' 2>/dev/null || echo "")

# Extract file path (for Edit/Write tools)
FILE_PATH=$(echo "$TOOL_INPUT" | jq -r '.file_path // empty' 2>/dev/null || echo "")

# Extract content (for Write tool)
CONTENT=$(echo "$TOOL_INPUT" | jq -r '.content // empty' 2>/dev/null || echo "")

# Handle jq parse failures gracefully
if [[ $? -ne 0 ]] || [[ -z "$COMMAND" ]]; then
    # Not applicable or parse failed - allow by default
    echo '{"allow": true}'
    exit 0
fi
```

### SessionStart Hook

| Variable | Available | Type | Example | Notes |
|----------|-----------|------|---------|-------|
| PWD | Yes | string | "/Users/user/project" | Current working directory |
| CLAUDE_CONFIG_DIR | Yes | string | ".claude-config" | Config directory path |
| ANTHROPIC_API_KEY | Yes | string | "" or "sk-..." | API key (if set) |

### UserPromptSubmit Hook

| Variable | Available | Type | Example | Notes |
|----------|-----------|------|---------|-------|
| USER_PROMPT | Yes | string | "Fix the bug in auth.py" | User's input text |

### Stop Hook

| Variable | Available | Type | Example | Notes |
|----------|-----------|------|---------|-------|
| SESSION_ID | Yes | string | "sess_abc123..." | Session identifier |
| TOOL_USE_COUNT | Yes | integer | "42" | Number of tools used |

---

## Advanced JSON Output Patterns

### Basic Allow/Deny

```bash
# Allow execution
echo '{"allow": true}'

# Block execution
echo '{"allow": false}'

# Explicit deny (same as allow: false)
echo '{"deny": true}'
```

### With Explanation

```bash
# Block with reason
echo '{
  "allow": false,
  "explanation": "Git force push to main branch is not allowed. Use feature branches."
}'

# Allow with warning
echo '{
  "allow": true,
  "explanation": "Proceeding, but note: This file is auto-generated."
}'
```

### With Append Message (Shown to Claude)

```bash
# Block and suggest alternative
echo '{
  "allow": false,
  "explanation": "Direct git operations not allowed",
  "appendMessage": "Use /github-admin skill instead:\n\n  Ask github-admin to push your changes\n  Example: \"GitHub admin, please push these changes to feature-branch\""
}'

# Allow with context
echo '{
  "allow": true,
  "appendMessage": "FYI: This file is reviewed by security team before deployment."
}'
```

### Stop Conversation

```bash
# Block and stop entire conversation
echo '{
  "allow": false,
  "stop": true,
  "explanation": "Critical security violation detected. Session terminated.",
  "appendMessage": "SECURITY ALERT: Attempted to modify production secrets.\n\nSession ended. Please contact security team."
}'
```

### Conditional Logic

```bash
# Complex decision tree
if [[ "$COMMAND" =~ ^git\ push.*--force ]]; then
    if [[ "$COMMAND" =~ main|master ]]; then
        # Force push to main - ALWAYS block
        echo '{
          "allow": false,
          "explanation": "Force push to main branch prohibited",
          "appendMessage": "This operation is NEVER allowed. Use feature branches."
        }'
    else
        # Force push to feature branch - warn but allow
        echo '{
          "allow": true,
          "explanation": "Force push to feature branch allowed",
          "appendMessage": "⚠️ WARNING: Force push detected. Ensure no one else is working on this branch."
        }'
    fi
else
    # Not a force push - allow
    echo '{"allow": true}'
fi
```

---

## Complete Hook Examples

### Example 1: ali-git-guard.sh (Production Hook)

> **Credential system clarification:** ali-git-guard.sh enforces git and gh CLI operation safety (blocking force pushes, hard resets, unreviewed write operations, etc.). It does NOT manage gh CLI authentication or OAuth credentials. gh CLI credential isolation is handled separately by configure_gh_cli() in setup_dev_environment.sh, which stores per-project credentials in .claude-config/gh/ using GH_CONFIG_DIR. These are two distinct systems — do not conflate them. ali-git-guard.sh scope: git/gh command safety. .claude-config/gh/ scope: gh CLI authentication state.

```bash
#!/bin/bash
# Hook: ali-git-guard
# Purpose: Enforce git/gh command safety and delegate to github-admin skill
# Event: PreToolUse
# Matcher: Bash

set -euo pipefail

# Environment
TOOL_NAME="${TOOL_NAME:-unknown}"
TOOL_INPUT="${TOOL_INPUT:-{}}"
LOG_FILE="${HOME}/.claude/logs/git-operations.log"

# Logging helper
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" >> "$LOG_FILE"
}

# Extract command from TOOL_INPUT
COMMAND=$(echo "$TOOL_INPUT" | jq -r '.command // empty' 2>/dev/null || echo "")

# Not a bash command or parse failed - allow
if [[ -z "$COMMAND" ]]; then
    echo '{"allow": true}'
    exit 0
fi

# Check if command is git/gh related
if [[ ! "$COMMAND" =~ ^(git|gh)\ ]]; then
    echo '{"allow": true}'
    exit 0
fi

log "Git command: $COMMAND"

# ─────────────────────────────────────────────────────────
# CRITICAL VIOLATIONS (ALWAYS BLOCKED)
# ─────────────────────────────────────────────────────────

# Force push to main/master
if [[ "$COMMAND" =~ git\ push.*--force.*(main|master) ]]; then
    log "BLOCKED: Force push to main/master"
    echo '{
      "allow": false,
      "explanation": "Force push to main/master is prohibited",
      "appendMessage": "🚫 BLOCKED: Force push to main/master branch is NEVER allowed.\n\nUse feature branches for your work."
    }'
    exit 0
fi

# Hard reset
if [[ "$COMMAND" =~ git\ reset\ --hard ]]; then
    log "BLOCKED: Hard reset"
    echo '{
      "allow": false,
      "explanation": "git reset --hard can destroy work",
      "appendMessage": "🚫 BLOCKED: git reset --hard destroys uncommitted work.\n\nUse git stash or git checkout instead."
    }'
    exit 0
fi

# Clean with force
if [[ "$COMMAND" =~ git\ clean\ -.*f ]]; then
    log "BLOCKED: git clean -f"
    echo '{
      "allow": false,
      "explanation": "git clean -f deletes untracked files permanently",
      "appendMessage": "🚫 BLOCKED: git clean -f permanently deletes files.\n\nReview with git clean -n first, then ask for approval."
    }'
    exit 0
fi

# Rebase on main/master
if [[ "$COMMAND" =~ git\ rebase.*(main|master) ]]; then
    log "BLOCKED: Rebase on main/master"
    echo '{
      "allow": false,
      "explanation": "Rebasing on main/master can rewrite shared history",
      "appendMessage": "🚫 BLOCKED: Rebasing on main/master rewrites shared history.\n\nUse git merge instead, or ask github-admin for guidance."
    }'
    exit 0
fi

# ─────────────────────────────────────────────────────────
# DANGEROUS OPERATIONS (REQUIRE CONFIRMATION)
# ─────────────────────────────────────────────────────────

# Force push to feature branches
if [[ "$COMMAND" =~ git\ push.*--force ]]; then
    log "WARN: Force push to feature branch"
    echo '{
      "allow": false,
      "explanation": "Force push requires confirmation",
      "appendMessage": "⚠️ Force push detected.\n\nThis will rewrite remote history. Are you sure?\n- If working alone on this branch: OK\n- If others have pulled this branch: NOT OK\n\nReply YES to proceed, or ask github-admin for guidance."
    }'
    exit 0
fi

# Delete branch (local or remote)
if [[ "$COMMAND" =~ git\ branch\ -D ]] || [[ "$COMMAND" =~ git\ push.*--delete ]]; then
    log "WARN: Branch deletion"
    echo '{
      "allow": false,
      "explanation": "Branch deletion requires confirmation",
      "appendMessage": "⚠️ Branch deletion detected.\n\nEnsure:\n- Branch is merged or no longer needed\n- No one else depends on this branch\n\nReply YES to proceed."
    }'
    exit 0
fi

# ─────────────────────────────────────────────────────────
# SAFE READ-ONLY OPERATIONS (ALWAYS ALLOWED)
# ─────────────────────────────────────────────────────────

if [[ "$COMMAND" =~ ^git\ (status|log|diff|show|fetch|branch\ --list|remote|config\ --get) ]]; then
    log "ALLOWED: Safe read-only operation"
    echo '{"allow": true}'
    exit 0
fi

if [[ "$COMMAND" =~ ^gh\ (repo\ view|issue\ list|pr\ list|release\ list) ]]; then
    log "ALLOWED: Safe gh read-only operation"
    echo '{"allow": true}'
    exit 0
fi

# ─────────────────────────────────────────────────────────
# WRITE OPERATIONS (SUGGEST DELEGATION TO github-admin)
# ─────────────────────────────────────────────────────────

# Git write operations
if [[ "$COMMAND" =~ ^git\ (add|commit|push|pull|merge|rebase|cherry-pick|stash) ]]; then
    log "SUGGEST DELEGATION: Git write operation"
    echo '{
      "allow": false,
      "explanation": "Git write operations should use github-admin skill",
      "appendMessage": "Use /github-admin skill for git operations:\n\n  Ask github-admin to: '"$COMMAND"'\n  Example: \"GitHub admin, please commit these changes with message: feat: add new feature\""
    }'
    exit 0
fi

# GitHub write operations (gh CLI)
if [[ "$COMMAND" =~ ^gh\ (pr\ create|issue\ create|release\ create) ]]; then
    log "SUGGEST DELEGATION: GitHub write operation"
    echo '{
      "allow": false,
      "explanation": "GitHub operations should use github-admin skill",
      "appendMessage": "Use /github-admin skill for GitHub operations:\n\n  Ask github-admin to create PR/issue/release"
    }'
    exit 0
fi

# ─────────────────────────────────────────────────────────
# DEFAULT: BLOCK UNKNOWN GIT/GH OPERATIONS
# ─────────────────────────────────────────────────────────

log "BLOCKED: Unknown git/gh operation"
echo '{
  "allow": false,
  "explanation": "Unknown git/gh operation",
  "appendMessage": "Git/gh operation not recognized:\n\n  '"$COMMAND"'\n\nFor safety, this operation is blocked. Use /github-admin skill for git operations."
}'
exit 0
```

### Example 2: File Protection Hook

```bash
#!/bin/bash
# Hook: ali-protect-files
# Purpose: Block writes to protected files and directories
# Event: PreToolUse
# Matcher: Write|Edit

set -euo pipefail

TOOL_NAME="${TOOL_NAME:-unknown}"
TOOL_INPUT="${TOOL_INPUT:-{}}"

# Extract file path
FILE_PATH=$(echo "$TOOL_INPUT" | jq -r '.file_path // empty' 2>/dev/null || echo "")

if [[ -z "$FILE_PATH" ]]; then
    echo '{"allow": true}'
    exit 0
fi

# Protected patterns
PROTECTED_PATTERNS=(
    "^config/production\."
    "^\.env$"
    "^\.env\."
    "^secrets/"
    "^\.aws/credentials"
    "^\.ssh/id_"
)

# Check against protected patterns
for pattern in "${PROTECTED_PATTERNS[@]}"; do
    if [[ "$FILE_PATH" =~ $pattern ]]; then
        echo '{
          "allow": false,
          "explanation": "Protected file: '"$FILE_PATH"'",
          "appendMessage": "🚫 BLOCKED: Cannot modify protected file:\n\n  '"$FILE_PATH"'\n\nProtected files/directories:\n- config/production.* (production configs)\n- .env* (environment files)\n- secrets/ (secret storage)\n- .aws/credentials (AWS credentials)\n- .ssh/id_* (SSH keys)\n\nUse configuration management tools instead."
        }'
        exit 0
    fi
done

# Allow all other files
echo '{"allow": true}'
exit 0
```

### Example 3: Auto-Formatter Hook (PostToolUse)

```bash
#!/bin/bash
# Hook: ali-auto-format
# Purpose: Auto-format files after edit based on file type
# Event: PostToolUse
# Matcher: Edit|Write

set -euo pipefail

TOOL_NAME="${TOOL_NAME:-unknown}"
FILE_PATH="${FILE_PATH:-}"

# No file path - skip
if [[ -z "$FILE_PATH" ]]; then
    echo '{"allow": true}'
    exit 0
fi

# File doesn't exist - skip
if [[ ! -f "$FILE_PATH" ]]; then
    echo '{"allow": true}'
    exit 0
fi

# Python files → black
if [[ "$FILE_PATH" =~ \.py$ ]]; then
    if command -v black &> /dev/null; then
        black "$FILE_PATH" &> /dev/null || true
    fi
fi

# JavaScript/TypeScript → prettier
if [[ "$FILE_PATH" =~ \.(js|ts|jsx|tsx)$ ]]; then
    if command -v prettier &> /dev/null; then
        prettier --write "$FILE_PATH" &> /dev/null || true
    fi
fi

# Shell scripts → shfmt
if [[ "$FILE_PATH" =~ \.sh$ ]]; then
    if command -v shfmt &> /dev/null; then
        shfmt -w "$FILE_PATH" &> /dev/null || true
    fi
fi

# Go files → gofmt
if [[ "$FILE_PATH" =~ \.go$ ]]; then
    if command -v gofmt &> /dev/null; then
        gofmt -w "$FILE_PATH" &> /dev/null || true
    fi
fi

# Always allow (formatting is best-effort)
echo '{"allow": true}'
exit 0
```

### Example 4: Session Protocol Hook

```bash
#!/bin/bash
# Hook: ali-session-protocol
# Purpose: Enforce session start protocol for Aliunde projects
# Event: SessionStart

set -euo pipefail

# Check if we're in an Aliunde project
if [[ ! -f ".aliunde-project" ]]; then
    echo '{"allow": true}'
    exit 0
fi

# Check for required documentation
MISSING_FILES=()

if [[ ! -f "ARCHITECTURE.md" ]]; then
    MISSING_FILES+=("ARCHITECTURE.md")
fi

if [[ ! -f "RUNBOOK.md" ]]; then
    MISSING_FILES+=("RUNBOOK.md")
fi

if [[ ! -f "CLAUDE.md" ]]; then
    MISSING_FILES+=("CLAUDE.md")
fi

# If files missing, warn but allow
if [[ ${#MISSING_FILES[@]} -gt 0 ]]; then
    FILES_LIST=$(printf '\n- %s' "${MISSING_FILES[@]}")
    echo '{
      "allow": true,
      "appendMessage": "⚠️ SESSION PROTOCOL WARNING\n\nMissing required documentation files:'"$FILES_LIST"'\n\nYou must still follow session protocol:\n1. Read available context files\n2. Understand project architecture\n3. Ask clarifying questions before changes\n\nCreate missing files before starting work."
    }'
    exit 0
fi

# All files present - remind about protocol
echo '{
  "allow": true,
  "appendMessage": "📋 SESSION PROTOCOL\n\nBefore starting work, please:\n1. Read ARCHITECTURE.md (system design)\n2. Read RUNBOOK.md (operational procedures)\n3. Read CLAUDE.md (project-specific rules)\n4. Ask clarifying questions if anything is unclear\n\nType \"ready\" when you have completed session protocol."
}'
exit 0
```

---

## Hook Creation Template (Comprehensive)

```bash
#!/bin/bash
# Hook: ali-[descriptive-name]
# Purpose: [One-line description of what this hook does]
# Event: PreToolUse|PostToolUse|SessionStart|UserPromptSubmit|Stop
# Matcher: Bash|Edit|Write|Task|Read|Grep|Glob (if PreToolUse/PostToolUse)
# Created: YYYY-MM-DD
# Author: Your Name

set -euo pipefail

# ─────────────────────────────────────────────────────────
# Configuration
# ─────────────────────────────────────────────────────────

HOOK_NAME="ali-[name]"
LOG_FILE="${HOME}/.claude/logs/${HOOK_NAME}.log"

# ─────────────────────────────────────────────────────────
# Helper Functions
# ─────────────────────────────────────────────────────────

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" >> "$LOG_FILE"
}

error() {
    log "ERROR: $*"
}

allow() {
    echo '{"allow": true}'
    exit 0
}

block() {
    local reason="$1"
    local message="${2:-}"

    log "BLOCKED: $reason"

    if [[ -n "$message" ]]; then
        echo '{
          "allow": false,
          "explanation": "'"$reason"'",
          "appendMessage": "'"$message"'"
        }'
    else
        echo '{
          "allow": false,
          "explanation": "'"$reason"'"
        }'
    fi

    exit 0
}

# ─────────────────────────────────────────────────────────
# Environment Variables
# ─────────────────────────────────────────────────────────

TOOL_NAME="${TOOL_NAME:-unknown}"
TOOL_INPUT="${TOOL_INPUT:-{}}"
FILE_PATH="${FILE_PATH:-}"

# ─────────────────────────────────────────────────────────
# Extract Data from TOOL_INPUT
# ─────────────────────────────────────────────────────────

# For Bash tools
COMMAND=$(echo "$TOOL_INPUT" | jq -r '.command // empty' 2>/dev/null || echo "")

# For Edit/Write tools (alternative to FILE_PATH env var)
# FILE_PATH_FROM_INPUT=$(echo "$TOOL_INPUT" | jq -r '.file_path // empty' 2>/dev/null || echo "")

# For Read/Grep tools
# READ_PATH=$(echo "$TOOL_INPUT" | jq -r '.file_path // empty' 2>/dev/null || echo "")

# ─────────────────────────────────────────────────────────
# Validation: Should this hook process this tool use?
# ─────────────────────────────────────────────────────────

# Example: Only process Bash commands
if [[ "$TOOL_NAME" != "Bash" ]] || [[ -z "$COMMAND" ]]; then
    allow
fi

# Example: Only process specific file types
# if [[ ! "$FILE_PATH" =~ \.py$ ]]; then
#     allow
# fi

# ─────────────────────────────────────────────────────────
# Hook Logic
# ─────────────────────────────────────────────────────────

log "Processing: $TOOL_NAME - $COMMAND"

# Example: Block dangerous operations
if [[ "$COMMAND" =~ dangerous-pattern ]]; then
    block "Dangerous operation detected" "🚫 This operation is not allowed because..."
fi

# Example: Warning but allow
if [[ "$COMMAND" =~ warning-pattern ]]; then
    log "WARN: Risky operation"
    echo '{
      "allow": true,
      "appendMessage": "⚠️ Warning: This operation is risky. Proceed with caution."
    }'
    exit 0
fi

# Default: Allow
log "ALLOWED"
allow
```

---

## Hook Testing Patterns

### Manual Testing

```bash
# Set environment variables to simulate tool use
export TOOL_NAME="Bash"
export TOOL_INPUT='{"command":"git push origin main"}'

# Run hook
.claude/hooks/ali-git-guard.sh

# Expected output: JSON response
# {"allow": false, "explanation": "...", "appendMessage": "..."}

# Unset for next test
unset TOOL_NAME TOOL_INPUT
```

### Automated Testing Script

```bash
#!/bin/bash
# test-hook.sh
# Automated hook testing script

HOOK_PATH="$1"
TEST_CASES_FILE="$2"

# Test case format (JSON):
# {
#   "name": "test name",
#   "tool_name": "Bash",
#   "tool_input": {"command": "git push"},
#   "file_path": "/path/to/file",
#   "expected_allow": false,
#   "expected_reason": "reason string"
# }

while IFS= read -r test_case; do
    TEST_NAME=$(echo "$test_case" | jq -r '.name')
    echo "Running test: $TEST_NAME"

    export TOOL_NAME=$(echo "$test_case" | jq -r '.tool_name')
    export TOOL_INPUT=$(echo "$test_case" | jq -c '.tool_input')
    export FILE_PATH=$(echo "$test_case" | jq -r '.file_path // empty')

    # Run hook
    RESULT=$("$HOOK_PATH")

    # Parse result
    ACTUAL_ALLOW=$(echo "$RESULT" | jq -r '.allow')
    EXPECTED_ALLOW=$(echo "$test_case" | jq -r '.expected_allow')

    if [[ "$ACTUAL_ALLOW" == "$EXPECTED_ALLOW" ]]; then
        echo "  ✓ PASS"
    else
        echo "  ✗ FAIL: Expected allow=$EXPECTED_ALLOW, got $ACTUAL_ALLOW"
        exit 1
    fi

    unset TOOL_NAME TOOL_INPUT FILE_PATH
done < <(jq -c '.[]' "$TEST_CASES_FILE")

echo "All tests passed"
```

---

## Hook Debugging

### Enable Verbose Logging

```bash
#!/bin/bash
# Add to hook for verbose debugging

DEBUG="${DEBUG:-false}"

debug() {
    if [[ "$DEBUG" == "true" ]]; then
        echo "[DEBUG] $*" >> "$LOG_FILE"
    fi
}

debug "TOOL_NAME: $TOOL_NAME"
debug "TOOL_INPUT: $TOOL_INPUT"
debug "FILE_PATH: $FILE_PATH"
debug "COMMAND: $COMMAND"
```

### Test with Debug Output

```bash
DEBUG=true .claude/hooks/ali-git-guard.sh
tail -f ~/.claude/logs/git-operations.log
```


---

## Inline Pipe Testing Pattern

<!-- source-doc: RUNBOOK.md — Testing section, "Test a Hook Locally" -->

**When to use:** Quick one-liner tests during hook development. More compact than the export-then-run pattern above. Especially useful in scripts and CI.

The inline environment variable + stdin pattern lets you test a hook without polluting your shell environment:

```bash
cd ~/ali-ai/hooks

# Test with sample input — inline env vars, pipe the JSON payload to stdin
echo '{"command": "git status"}' | \
  TOOL_NAME=Bash \
  TOOL_INPUT='{"command":"git status"}' \
  ./git-guard.sh

# Expected output: {"allow": true} or {"allow": false, "reason": "..."}
```

**How it works:**
- Env vars (`TOOL_NAME=Bash`) are set inline for this command only (no export needed, no cleanup needed)
- The stdin pipe (`echo ... |`) feeds TOOL_INPUT as if Claude called the hook
- Hook reads from its environment and TOOL_INPUT, outputs JSON to stdout

**Testing a blocked operation:**

```bash
echo '{"command": "git push --force origin main"}' | \
  TOOL_NAME=Bash \
  TOOL_INPUT='{"command":"git push --force origin main"}' \
  ./git-guard.sh

# Expected: {"allow": false, "explanation": "Force push to main/master is prohibited", ...}
```

**Note:** Both the inline pattern (this section) and the export pattern (Manual Testing section above) work. Use whichever is cleaner for your context.

---

## settings.json Hooks Template

<!-- source-doc: RUNBOOK.md — Configuration section, "Hook Registration" -->

The hooks template at `~/ali-ai/templates/settings.json` is the canonical source for hook registration. It is automatically merged into each project's `.claude-config/settings.json` on first launch.

**Template structure:**

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/git-guard.sh"
          },
          {
            "type": "command",
            "command": ".claude/hooks/aws-guard.sh"
          }
        ]
      }
    ]
  }
}
```

**Note on hook paths:** Hook paths in settings.json reference the project's `.claude/hooks/` directory (not `~/ali-ai/hooks/` directly). The `.claude/hooks/` directory is populated by sync-org-resources.sh on launch.

**Config locations:**
- Org mode: `.claude-config/settings.json` (per-project)
- Personal mode: `~/.config/claude-personal/settings.json` (user-level)

**Registration is automatic:** The claude-ali launcher handles hook registration on first launch. Manual edits to settings.json are only needed when customizing beyond the org defaults.

---

**Last Updated:** 2026-02-28 (Phase 3 credential remediation — ali-git-guard.sh credential system clarification)
**Maintained By:** ALI AI Team
