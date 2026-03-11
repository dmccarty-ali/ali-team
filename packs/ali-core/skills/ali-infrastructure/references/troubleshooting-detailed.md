# ali-infrastructure - Detailed Troubleshooting Reference

This document provides in-depth troubleshooting procedures for ali-ai infrastructure issues.

---

## Hook Blocking Operations Incorrectly

**Scenario:** Hook is blocking safe operations that should be allowed.

### Symptoms
- Safe git commands being blocked (e.g., git status, git log)
- Read-only operations being blocked
- Hook returning unexpected error messages

### Debugging Procedure

**Step 1: Check hook logs**
```bash
# Locate log file
cat ~/.claude/logs/git-operations.log

# Or project-specific logs
cat .claude/logs/git-operations.log

# Look for recent entries
tail -20 ~/.claude/logs/git-operations.log
```

**Step 2: Test hook manually**
```bash
# Set environment variables to simulate tool use
export TOOL_NAME="Bash"
export TOOL_INPUT='{"command":"git status"}'

# Run hook directly
.claude/hooks/ali-git-guard.sh

# Check output
# Expected for safe operation: {"allow": true}
# If getting: {"allow": false, ...} then hook logic is wrong
```

**Step 3: Check JSON parsing**

Hook may be failing to parse TOOL_INPUT:
```bash
# Test jq parsing manually
echo '{"command":"git status"}' | jq -r '.command // empty'
# Should output: git status

# If jq not installed or command fails:
which jq
# If missing: brew install jq (macOS) or apt-get install jq (Linux)
```

**Step 4: Check pattern matching**

Hook may have incorrect regex patterns:
```bash
# Add debug output to hook
vim .claude/hooks/ali-git-guard.sh

# Add after command extraction:
echo "DEBUG: COMMAND=$COMMAND" >&2

# Run hook again with debug
export TOOL_INPUT='{"command":"git status"}'
.claude/hooks/ali-git-guard.sh 2>&1
# Should see: DEBUG: COMMAND=git status
```

**Step 5: Review hook logic**

Common pattern matching errors:
```bash
# WRONG: Too broad pattern
if [[ "$COMMAND" =~ git ]]; then
    # This matches ANY command containing "git" (e.g., "digit")
fi

# RIGHT: Specific pattern
if [[ "$COMMAND" =~ ^git\  ]]; then
    # This matches only commands STARTING with "git "
fi
```

**Step 6: Fix hook in source**

If hook logic is wrong:
```bash
# Edit source hook (not .claude/ copy)
cd ~/ali-ai
vim hooks/ali-git-guard.sh

# Fix pattern matching or logic
# Test manually (see Step 2)
# Commit fix

git add hooks/ali-git-guard.sh
git commit -m "fix(hooks): correct git-guard pattern matching for safe operations"
git push origin main
```

**Step 7: Re-sync to deploy fix**
```bash
# In affected project
claude-ali --force-sync

# Verify fix deployed
cat .claude/hooks/ali-git-guard.sh
# Should contain fix

# Test again
claude-ali
# Try previously-blocked operation
# Should now be allowed
```

### Common Causes

| Cause | Symptom | Fix |
|-------|---------|-----|
| jq parse failure | All operations blocked | Install jq or fix fallback logic |
| Overly broad regex | Too many operations blocked | Tighten regex patterns (use ^, $, \s) |
| Missing fallback | Hook crashes on invalid input | Add fallback: `\|\| echo ""` |
| Exit code 1 | Claude shows error, not block | Change to exit 0 with {"allow": false} |
| Empty TOOL_INPUT | Hook blocks all operations | Check for empty/missing input before parsing |

---

## Hooks Not Executing

**Scenario:** Hooks are configured but not triggering when expected.

### Symptoms
- No log entries in hook log files
- Operations proceeding without validation
- No block/allow messages shown to Claude

### Debugging Procedure

**Step 1: Check settings.json has hooks**
```bash
# Check project settings
cat .claude-config/settings.json | jq '.hooks'

# Should show hooks configuration like:
# {
#   "PreToolUse": [
#     {
#       "matcher": "Bash",
#       "hooks": [{"type": "command", "command": ".claude/hooks/ali-git-guard.sh"}]
#     }
#   ]
# }

# If missing or empty: hooks not configured
```

**Step 2: Verify hooks are executable**
```bash
# Check permissions
ls -la .claude/hooks/

# Should show: -rwxr-xr-x (executable)
# If not executable: -rw-r--r--

# Fix permissions
chmod +x .claude/hooks/*.sh
```

**Step 3: Check hook matcher**

Matcher must match tool being used:
```bash
# Check settings.json matcher
cat .claude-config/settings.json | jq '.hooks.PreToolUse[].matcher'

# Common matchers:
# - "Bash" (for git, npm, etc.)
# - "Edit" (for file edits)
# - "Write" (for file writes)
# - "Read" (for file reads)

# If matcher is "Bash" but you're using Edit tool, hook won't trigger
```

**Step 4: Test hook manually**

Verify hook works independently:
```bash
# Set sample TOOL_INPUT
export TOOL_NAME="Bash"
export TOOL_INPUT='{"command":"git push origin main"}'

# Run hook
.claude/hooks/ali-git-guard.sh

# Should output JSON
# If no output or error: hook has syntax error
```

**Step 5: Check for syntax errors**
```bash
# Validate bash syntax
bash -n .claude/hooks/ali-git-guard.sh

# Should output nothing (no errors)
# If errors shown: fix syntax

# Common syntax errors:
# - Missing quotes around variables
# - Unmatched brackets
# - Missing `then` after `if`
# - Missing `fi` to close `if`
```

**Step 6: Check Claude Code version**

Hooks require Claude Code CLI (not Claude Desktop):
```bash
# Check version
claude --version

# Should show: Claude Code CLI version X.X.X
# If shows Claude Desktop: hooks not supported

# Install Claude Code CLI
npm install -g @anthropic-ai/claude-code
```

**Step 7: Re-sync to get latest template**

Settings may be outdated:
```bash
# Force re-sync
claude-ali --force-sync

# This will:
# - Update .claude-config/settings.json from template
# - Restore hook configurations
# - Verify hook files exist and are executable
```

**Step 8: Enable hook debugging**

Add debug logging to hook:
```bash
vim .claude/hooks/ali-git-guard.sh

# Add at top (after set -euo pipefail):
LOG_FILE="$HOME/.claude/logs/git-guard-debug.log"
echo "[$(date)] Hook triggered: TOOL_NAME=$TOOL_NAME" >> "$LOG_FILE"
echo "[$(date)] TOOL_INPUT=$TOOL_INPUT" >> "$LOG_FILE"

# Launch Claude and trigger hook
claude-ali
# Run operation that should trigger hook

# Check debug log
cat ~/.claude/logs/git-guard-debug.log
# Should show entries if hook is triggering
# If empty: hook not being called by Claude Code
```

### Common Causes

| Cause | Symptom | Fix |
|-------|---------|-----|
| Missing settings.json | No hooks executed | Run claude-ali --force-sync |
| Wrong matcher | Hooks don't trigger for specific tools | Update matcher in settings.json |
| Not executable | Silent failure | chmod +x .claude/hooks/*.sh |
| Syntax error | Hook crashes | bash -n to validate, fix errors |
| Wrong path | Hook not found | Verify path in settings.json matches actual location |
| Claude Desktop vs CLI | Hooks not supported | Use Claude Code CLI (not Desktop) |

---

## Docker Testing vs Host Machine

**Scenario:** Need to test in Docker but container has no .claude-config/ directory.

### Context

**Host machine:**
- Has .claude-config/ directory with OAuth tokens
- ANTHROPIC_API_KEY not needed (uses OAuth)
- Command: `claude-ali`

**Docker container:**
- No .claude-config/ directory (not mounted, by design)
- No OAuth tokens available
- ANTHROPIC_API_KEY REQUIRED
- Command: `docker run -e ANTHROPIC_API_KEY=$KEY ...`

### Why This Happens

```
Security design: .claude-config/ contains OAuth tokens
- Mounting would share credentials across containers (bad)
- Ephemeral containers shouldn't persist credentials
- Solution: Use API key for Docker, OAuth for host
```

### Testing Pattern

**On host (development):**
```bash
# Uses OAuth from .claude-config/
cd ~/ali-ai/projects/myproject
claude-ali

# OAuth tokens in: .claude-config/credentials.json
# No API key needed
```

**In Docker (CI/CD, testing):**
```bash
# Must use API key
docker run \
  -e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY \
  -v $(pwd):/workspace \
  -w /workspace \
  claude-code-image \
  claude "run tests"

# Or with claude-ali wrapper:
docker run \
  -e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY \
  -v $(pwd):/workspace \
  -w /workspace \
  claude-code-image \
  bash -c "source ~/.bashrc && claude-ali"
```

### Getting API Key

**For testing purposes:**
```bash
# Generate API key from Anthropic Console
# https://console.anthropic.com/settings/keys

# Set in environment
export ANTHROPIC_API_KEY="sk-ant-..."

# Use in Docker
docker run -e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY ...
```

**For CI/CD:**
```bash
# Store as secret in CI system
# GitHub Actions: Settings → Secrets → ANTHROPIC_API_KEY
# GitLab CI: Settings → CI/CD → Variables → ANTHROPIC_API_KEY

# Reference in workflow
env:
  ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

### Dockerfile Pattern

```dockerfile
FROM node:18

# Install Claude Code CLI
RUN npm install -g @anthropic-ai/claude-code

# Install dependencies
RUN apt-get update && apt-get install -y \
    git \
    jq \
    rsync

# Copy ali-ai infrastructure
COPY --from=ali-ai /ali-ai /root/ali-ai
RUN chmod +x /root/ali-ai/scripts/*.sh

# Setup shell alias
RUN echo 'alias claude-ali="~/.claude/bin/claude-ali"' >> ~/.bashrc

WORKDIR /workspace

# Note: ANTHROPIC_API_KEY must be provided at runtime
# Not baked into image (security)
```

### Docker Compose Pattern

```yaml
version: '3.8'
services:
  claude-dev:
    image: claude-code-dev:latest
    environment:
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
    volumes:
      - ./:/workspace
      - ~/.ssh:/root/.ssh:ro  # Mount SSH keys (read-only)
    working_dir: /workspace
    command: bash
```

### Testing Procedure

**Step 1: Build Docker image**
```bash
docker build -t claude-code-dev .
```

**Step 2: Run with API key**
```bash
# Interactive shell
docker run -it \
  -e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY \
  -v $(pwd):/workspace \
  claude-code-dev

# Inside container:
cd /workspace
claude "list all files"
```

**Step 3: Verify Claude works**
```bash
# Test Claude CLI
claude --version

# Test simple query
claude "echo hello"

# Should see Claude response
# If error about authentication: check API key
```

**Step 4: Test with claude-ali**
```bash
# Inside container
cd /workspace

# Run claude-ali (will skip OAuth, use API key)
claude-ali

# Should launch Claude with project context
```

### Troubleshooting Docker Issues

**Problem: "Authentication failed"**
```bash
# Check API key is set
echo $ANTHROPIC_API_KEY
# Should show: sk-ant-...

# If empty:
export ANTHROPIC_API_KEY="sk-ant-..."
# Re-run docker command
```

**Problem: "ANTHROPIC_API_KEY not set"**
```bash
# Verify -e flag in docker run
docker run -e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY ...

# Or use --env-file
docker run --env-file .env.docker ...

# .env.docker contents:
ANTHROPIC_API_KEY=sk-ant-...
```

**Problem: "claude-ali not found"**
```bash
# Inside container, check if ali-ai copied
ls -la ~/ali-ai/scripts/

# If missing, check Dockerfile COPY command
# Or mount ali-ai:
docker run -v ~/ali-ai:/root/ali-ai ...
```

### Security Notes

**DO NOT:**
- Bake ANTHROPIC_API_KEY into Docker image
- Commit .env files with API keys to git
- Share API keys in logs or console output

**DO:**
- Use environment variables at runtime
- Use secrets management (GitHub Secrets, AWS Secrets Manager)
- Rotate API keys regularly
- Use separate API keys for dev/staging/prod


---

## Skill Not Triggering

<!-- source-doc: RUNBOOK.md — Troubleshooting section, "Skill Not Triggering" -->

**Symptoms:** Skill that should activate based on context is not appearing or being used.

**Diagnostic checklist:**

1. **Check sync completed:**
   ```bash
   ls .claude/skills/
   # Should show skill directories including the expected skill
   ```

2. **Check flat structure:** Skill must be at `skills/name/SKILL.md` (not nested). Nested directories are invisible to Claude Code.
   ```bash
   ls ~/ali-ai/skills/your-skill-name/
   # Should show SKILL.md directly (not another subdirectory)
   ```

3. **Check YAML frontmatter:** Description should cover all four modes (planning, implementation, guidance, review). Skills missing modes only trigger ~25% of the time they should.
   ```bash
   head -20 ~/ali-ai/skills/your-skill-name/SKILL.md
   # Check for PLANNING:, IMPLEMENTATION:, GUIDANCE:, REVIEW: in description
   ```

4. **Force re-sync:**
   ```bash
   ~/ali-ai/scripts/sync-org-resources.sh --force
   ```

---

## Agent Not Found

<!-- source-doc: RUNBOOK.md — Troubleshooting section, "Agent Not Found" -->

**Symptoms:** Agent not available when invoked with "use ali-[agent-name] to..."

**Diagnostic checklist:**

1. **Check sync completed:**
   ```bash
   ls .claude/agents/
   # Should show ali-*.md files
   ```

2. **Check file exists in source:**
   ```bash
   ls ~/ali-ai/agents/ali-[agent-name].md
   ```

3. **Force re-sync:**
   ```bash
   ~/ali-ai/scripts/sync-org-resources.sh --force
   ```

**Note:** Agents are distributed to project `.claude/agents/` by the claude-ali launcher. They are NOT at `~/.claude/agents/`. Always check the project's `.claude/agents/` directory.

---

## Hook Not Firing

<!-- source-doc: RUNBOOK.md — Troubleshooting section, "Hook Not Firing" -->

**Symptoms:** Hook that should intercept operations is not running.

**Diagnostic checklist:**

1. **Check sync completed:**
   ```bash
   ls .claude/hooks/
   # Should show hook scripts
   ```

2. **Check executable permissions:**
   ```bash
   ls -la ~/ali-ai/hooks/[hook].sh
   # Should show -rwxr-xr-x (executable bit set)
   # If not: chmod +x ~/ali-ai/hooks/[hook].sh
   ```

3. **Check settings.json:** Hook must be registered under correct event and matcher:
   ```bash
   cat .claude-config/settings.json | jq '.hooks'
   ```

4. **Relaunch:** Hooks load at session start. A hook registered after session start will not fire until next launch:
   ```bash
   claude-ali
   ```

---

## Emergency Procedures

<!-- source-doc: RUNBOOK.md — Emergency Procedures section -->

### Bypass Pre-Push Review (Emergency Only)

Use only in genuine emergencies where the pre-push review hook is blocking a critical fix:

```bash
git config hooks.skipPrePushReview true
git push
git config --unset hooks.skipPrePushReview
```

### Disable All Hooks Temporarily

Remove or rename the hooks section in `~/.claude/settings.json`, then relaunch claude-ali. Restore after the emergency.

### Reset Project .claude/ Directory

If `.claude/` is corrupted or has merge conflicts:

```bash
rm -rf .claude
claude-ali  # Fresh sync from ~/ali-ai
```

### Rollout Validation Violations — Recovery

If `make rollout` exits with violations (exit code 1), a project's `.claude/skills/` contains a stale skill not backed by org source:

```bash
# Option 1: Run the appropriate migration
# Option 2: Remove the stale directory manually
rm -rf {project}/.claude/skills/{stale-skill-name}

# Option 3: Re-sync the project
cd {project} && bash scripts/sync-org-resources.sh --force

# Check rollout migration log
cat {project}/.claude/logs/migrations.log
```

---

**Last Updated:** 2026-02-25 (W2 Restructure — added skill-not-triggering, agent-not-found, hook-not-firing, emergency procedures)
**Maintained By:** ALI AI Team
