# ali-infrastructure - Launcher Internals Reference

This document provides detailed information about the claude-ali launcher's internal execution flow and credential isolation implementation.

---

## claude-ali Launcher - Detailed Execution Flow

The claude-ali launcher follows a six-step execution sequence with extensive validation and error handling at each stage.

### Step 1: Parse Arguments

```bash
# Supported arguments
--personal              # Use personal subscription (~/.config/claude-personal/)
--sync-org              # Sync org resources even in personal mode
--force-sync            # Force resource sync (ignore hash check)
--help                  # Display usage information
```

### Step 2: Validate ali-ai Environment

```bash
# Check ~/ali-ai exists and is a git repository
if [[ ! -d "$HOME/ali-ai/.git" ]]; then
    echo "ERROR: ~/ali-ai not found or not a git repository"
    exit 1
fi

# Check for remote updates (cached for 30 minutes)
CACHE_FILE="$HOME/.cache/claude-ali/remote-check"
if [[ ! -f "$CACHE_FILE" ]] || [[ $(find "$CACHE_FILE" -mmin +30) ]]; then
    cd ~/ali-ai
    git fetch origin main --quiet
    LOCAL=$(git rev-parse HEAD)
    REMOTE=$(git rev-parse origin/main)

    if [[ "$LOCAL" != "$REMOTE" ]]; then
        echo "WARNING: ~/ali-ai is behind remote. Run: cd ~/ali-ai && git pull"
    fi

    mkdir -p "$(dirname "$CACHE_FILE")"
    touch "$CACHE_FILE"
fi
```

### Step 3: Detect Project Context

```bash
# Look for .aliunde-project marker
PROJECT_DIR=$(pwd)
while [[ "$PROJECT_DIR" != "/" ]]; do
    if [[ -f "$PROJECT_DIR/.aliunde-project" ]]; then
        break
    fi
    PROJECT_DIR=$(dirname "$PROJECT_DIR")
done

# Org mode requires marker (unless --personal)
if [[ "$PROJECT_DIR" == "/" ]] && [[ "$PERSONAL_MODE" != "true" ]]; then
    echo "ERROR: Not in an Aliunde project (no .aliunde-project marker found)"
    echo "Run 'init-aliunde-project.sh' to initialize this project"
    exit 1
fi
```

### Step 4: Sync Organization Resources (if needed)

```bash
# Determine if sync is needed
LAST_SYNC=$(cat "$PROJECT_DIR/.claude/.last-sync" 2>/dev/null || echo "")
CURRENT_HASH=$(cd ~/ali-ai && git rev-parse HEAD)

if [[ "$LAST_SYNC" != "$CURRENT_HASH" ]] || [[ "$FORCE_SYNC" == "true" ]]; then
    echo "Syncing org resources from ~/ali-ai..."

    # Validate agents (regenerate-agents.sh)
    ~/.claude/bin/regenerate-agents.sh || {
        echo "ERROR: Agent validation failed"
        exit 1
    }

    # rsync org resources to .claude/
    rsync -a --delete \
        ~/ali-ai/agents/ \
        "$PROJECT_DIR/.claude/agents/"

    rsync -a --delete \
        ~/ali-ai/skills/ \
        "$PROJECT_DIR/.claude/skills/"

    rsync -a --delete \
        ~/ali-ai/hooks/ \
        "$PROJECT_DIR/.claude/hooks/"

    # Build agent manifest
    cd ~/ali-ai
    python3 scripts/build_agent_manifest.py \
        --agents-dir agents \
        --output "$PROJECT_DIR/.claude/AGENT_MANIFEST.json" || {
        echo "ERROR: Agent manifest build failed"
        exit 1
    }

    # Run pending migrations
    ~/.claude/bin/run-pending-migrations.sh "$PROJECT_DIR" || {
        echo "ERROR: Migrations failed"
        exit 1
    }

    # Overlay .claude-local/ resources (if they exist)
    if [[ -d "$PROJECT_DIR/.claude-local" ]]; then
        rsync -a \
            "$PROJECT_DIR/.claude-local/" \
            "$PROJECT_DIR/.claude/"
    fi

    # Update sync hash
    echo "$CURRENT_HASH" > "$PROJECT_DIR/.claude/.last-sync"

    echo "Sync complete ($(date +%H:%M:%S))"
else
    echo "Resources up to date (last sync: $(stat -f %Sm -t '%Y-%m-%d %H:%M' "$PROJECT_DIR/.claude/.last-sync"))"
fi
```

### Step 5: Setup Hooks Configuration

```bash
# Merge hooks from template into project settings.json
TEMPLATE="$HOME/ali-ai/templates/settings.json"
TARGET="$PROJECT_DIR/.claude-config/settings.json"

if command -v jq &> /dev/null; then
    # Use jq for proper JSON merge
    if [[ -f "$TARGET" ]]; then
        # Create backup
        cp "$TARGET" "$TARGET.bak"

        # Merge hooks from template
        jq -s '.[0] * .[1]' "$TARGET" "$TEMPLATE" > "$TARGET.tmp"
        mv "$TARGET.tmp" "$TARGET"
    else
        # No existing settings, copy template
        cp "$TEMPLATE" "$TARGET"
    fi
else
    # Fallback: replace entire file (no merge)
    echo "WARNING: jq not found, replacing settings.json (no merge)"
    cp "$TEMPLATE" "$TARGET"
fi
```

### Step 6: Launch Claude Code

```bash
# Set environment variables
export NODE_OPTIONS="--max-old-space-size=8192"  # 8GB heap for multi-agent

if [[ "$PERSONAL_MODE" == "true" ]]; then
    export CLAUDE_CONFIG_DIR="$HOME/.config/claude-personal"
else
    export CLAUDE_CONFIG_DIR="$PROJECT_DIR/.claude-config"
fi

# Explicitly unset API key to force subscription login
unset ANTHROPIC_API_KEY

# Prepare system prompt append (if Aliunde project)
APPEND_PROMPT=""
if [[ -f "$PROJECT_DIR/.aliunde-project" ]]; then
    APPEND_PROMPT="Follow session protocol: Read ARCHITECTURE.md, RUNBOOK.md, CLAUDE.md before starting work."
fi

# Launch Claude Code
if [[ -n "$APPEND_PROMPT" ]]; then
    claude --append-system-prompt "$APPEND_PROMPT"
else
    claude
fi
```

---

## Credential Isolation Implementation Details

### OAuth Token Storage

```
CLAUDE_CONFIG_DIR structure:
.claude-config/                    # Per-project credentials
├── settings.json                  # Hooks and permissions
├── credentials.json               # OAuth tokens (encrypted)
├── session.json                   # Active session state
└── cache/                         # Response cache
    ├── conversations/
    └── completions/
```

### First Launch Flow

```
1. User runs: claude-ali
   ↓
2. CLAUDE_CONFIG_DIR set to .claude-config/
   ↓
3. Claude Code checks for credentials.json
   ↓
4. Not found → Launch OAuth flow
   ↓
5. Browser opens: anthropic.com/oauth/authorize
   ↓
6. User logs in with organization account
   ↓
7. OAuth tokens stored in .claude-config/credentials.json
   ↓
8. Subsequent launches use stored tokens (no re-login)
```

### Subsequent Launch Flow

```
1. User runs: claude-ali
   ↓
2. CLAUDE_CONFIG_DIR set to .claude-config/
   ↓
3. Claude Code finds credentials.json
   ↓
4. Validates token expiry
   ↓
5. Token valid → Launch session immediately
   ↓
   Token expired → Refresh token automatically
   ↓
6. Session starts with organization subscription billing
```

### Multi-Account Pattern

```
# Work project 1 (org account)
cd ~/ali-ai/projects/tax-practice
claude-ali
# Uses: .claude-config/ → Org subscription

# Work project 2 (org account)
cd ~/ali-ai/projects/ingestion-engine
claude-ali
# Uses: .claude-config/ → Org subscription

# Personal project (personal account)
cd ~/personal/my-project
claude-ali --personal
# Uses: ~/.config/claude-personal/ → Personal subscription
```

### Environment Variable Precedence

```
Priority (highest to lowest):
1. ANTHROPIC_API_KEY (if set) → Direct API access (bypasses subscription)
2. CLAUDE_CONFIG_DIR + credentials.json → Subscription-based login
3. ~/.claude/credentials.json → Default subscription (not recommended)

claude-ali enforces: unset ANTHROPIC_API_KEY
This ensures subscription isolation works correctly.
```

### Docker Credential Context

```
Host machine:
- Has .claude-config/ directory
- OAuth tokens available
- No ANTHROPIC_API_KEY needed
- Command: claude-ali

Docker container:
- No .claude-config/ directory (not mounted)
- No OAuth tokens available
- ANTHROPIC_API_KEY REQUIRED
- Command: docker run -e ANTHROPIC_API_KEY=$KEY ...

Why: Docker containers are ephemeral, mounting .claude-config/
would share credentials across containers (security risk).
Solution: Use API key for Docker, OAuth for host.
```

### Troubleshooting Credential Issues

**Problem: "Please log in" prompt on every launch**

```bash
# Check CLAUDE_CONFIG_DIR is set
echo $CLAUDE_CONFIG_DIR
# Should be: /path/to/project/.claude-config

# Check credentials file exists
ls -la .claude-config/credentials.json

# Check file permissions
chmod 600 .claude-config/credentials.json

# Check for API key override
echo $ANTHROPIC_API_KEY
# Should be: (empty)

# Re-initialize config directory
rm -rf .claude-config/
~/.claude/bin/setup_dev_environment.sh
```

**Problem: Wrong account billing (personal work on org subscription)**

```bash
# Verify current config directory
echo $CLAUDE_CONFIG_DIR

# For personal work:
claude-ali --personal
# Should set: ~/.config/claude-personal/

# For org work:
claude-ali
# Should set: .claude-config/

# Check active subscription in Claude Code
# Settings → Account → Active Subscription
```

---

## gh CLI Credential Isolation (configure_gh_cli)

gh CLI credentials are isolated per-project using GH_CONFIG_DIR=.claude-config/gh/. This is separate from Claude Code subscription credentials stored in .claude-config/credentials.json.

**CRITICAL DISTINCTION:**
- Claude Code subscription credentials: .claude-config/credentials.json (launcher-managed)
- gh CLI OAuth credentials: .claude-config/gh/hosts.yml (configure_gh_cli()-managed)

Never conflate these two systems.

### Three-Path Decision Tree

configure_gh_cli() in scripts/setup_dev_environment.sh:198-302 follows three ordered paths. The paths are evaluated top to bottom; the first matching path wins.

```
configure_gh_cli(account, gh_config_dir_local)
│
├── [GATE] gh not installed?
│     └── YES → return 0 (clean skip, not an error)
│
├── [GATE] .claude-config/ in .gitignore?
│     └── NO → auto-append ".claude-config/" to .gitignore
│              FAIL to append → return 1 (abort to prevent credential exposure)
│
├── Path 1: hosts.yml exists?
│     └── YES → gh auth status --hostname github.com valid?
│                 YES → return 0 (fast path, skip login)
│                 NO  → remove hosts.yml, fall through to Path 2/3
│
├── Path 2: PAT token file exists at ~/.config/aliunde/tokens/{account}.github.token?
│     └── YES → validate_gh_token() succeeds?
│                 YES → gh auth login --with-token --git-protocol https
│                       FAIL → return 1
│                 NO  → fall through to Path 3
│
├── Path 3: hosts.yml still absent? (always true here unless Path 2 wrote it)
│     └── YES → gh auth login --web --git-protocol https
│               FAIL or cancelled → return 1
│
└── [POST-LOGIN] Account verification + git setup (applies to Path 2 and Path 3)
      gh api user → .login field
      Match expected account? (case-insensitive)
        YES → chmod 600 hosts.yml
              gh auth setup-git  (registers gh credential helper for HTTPS transport)
              return 0
        NO  → rm -rf .claude-config/gh/, return 1
              (user must re-run setup_dev_environment.sh)
```

### GH_CONFIG_DIR as the Isolation Mechanism

GH_CONFIG_DIR is set to the project-local path before every gh invocation inside configure_gh_cli():

```bash
GH_CONFIG_DIR=".claude-config/gh/" gh auth login --web --git-protocol https
GH_CONFIG_DIR=".claude-config/gh/" gh auth status --hostname github.com
GH_CONFIG_DIR=".claude-config/gh/" gh api user --jq '.login'
GH_CONFIG_DIR=".claude-config/gh/" gh auth setup-git
```

This means each project directory has an independent gh credential store. Two projects can authenticate as different GitHub accounts simultaneously.

Token storage location: .claude-config/gh/hosts.yml (gitignored).

### gitignore Gate (Automatic)

Before any credential write, configure_gh_cli() checks whether .claude-config/ is present in the project's .gitignore. If not, it appends:

```
# gh CLI credential isolation
.claude-config/
```

This gate is a hard gate: if the append fails (e.g., permission error), configure_gh_cli() returns 1 and no credentials are written. This prevents accidental exposure of OAuth tokens in git history.

Do not rely on .claude-config/ being in .gitignore before running setup — the gate handles it automatically.

### Account Verification (Automatic, Post-Login)

After Path 2 or Path 3 login completes, configure_gh_cli() calls:

```bash
GH_CONFIG_DIR=".claude-config/gh/" gh api user --jq '.login'
```

The returned login is compared (case-insensitive) against the expected account. On mismatch:
- .claude-config/gh/ is removed entirely
- configure_gh_cli() returns 1
- The user must re-run setup_dev_environment.sh

### Path 2 Deprecation

PAT token files at ~/.config/aliunde/tokens/{account}.github.token are deprecated. Path 2 (gh auth login --with-token) will be removed in v0.4.0. Teams should migrate to the web login flow (Path 3).

### Remediation: Auth Failure

If gh auth fails for any reason (wrong account, cancelled browser login, network error):

```bash
# Do NOT manually edit .claude-config/gh/hosts.yml
# Re-run setup — configure_gh_cli() will detect the invalid state and re-authenticate
~/.claude/bin/setup_dev_environment.sh
```

If the wrong account authenticated and you need to fix it immediately:

```bash
# Remove the gh credential directory (configure_gh_cli() also does this on mismatch)
rm -rf .claude-config/gh/

# Re-run setup to go through the login flow again
~/.claude/bin/setup_dev_environment.sh
```

---

## Performance Optimizations

### Hash-Based Change Detection

```bash
# Fast path: Hash check only (~6ms)
LAST_SYNC=$(cat .claude/.last-sync 2>/dev/null || echo "")
CURRENT_HASH=$(cd ~/ali-ai && git rev-parse HEAD)

if [[ "$LAST_SYNC" == "$CURRENT_HASH" ]]; then
    echo "Resources up to date"
    exit 0  # Skip sync entirely
fi

# Slow path: Full sync (~200ms)
# Only executed when ~/ali-ai has changed
```

### Incremental rsync

```bash
# rsync with --delete flag removes obsolete files
# But only transfers changed files (incremental)
rsync -a --delete ~/ali-ai/skills/ .claude/skills/

# Performance:
# - First sync: ~200ms (full transfer)
# - Subsequent sync: ~50ms (changed files only)
# - No changes: ~10ms (directory scan only)
```

### Remote Update Check Caching

```bash
# Cache remote check for 30 minutes
CACHE_FILE="$HOME/.cache/claude-ali/remote-check"

if [[ -f "$CACHE_FILE" ]] && [[ ! $(find "$CACHE_FILE" -mmin +30) ]]; then
    # Cache valid, skip git fetch
    exit 0
fi

# Cache expired, check remote
git fetch origin main --quiet
# Update cache timestamp
touch "$CACHE_FILE"
```

---

## Security Considerations

### Credential File Permissions

```bash
# .claude-config/ should not be world-readable
chmod 700 .claude-config/
chmod 600 .claude-config/credentials.json

# gh CLI credential permissions (enforced by configure_gh_cli)
chmod 600 .claude-config/gh/hosts.yml

# Verify permissions
ls -la .claude-config/
# Should show: drwx------ (700)
# credentials.json: -rw------- (600)
# gh/hosts.yml: -rw------- (600)
```

### .env File Handling

```bash
# .env contains service credentials (not Claude credentials)
# Should be in .gitignore
# Permissions should be restrictive
chmod 600 .env

# Never commit to git
git ls-files | grep -q '\.env$' && {
    echo "ERROR: .env file is tracked by git!"
    echo "Run: git rm --cached .env"
    exit 1
}
```

### Separation of Concerns

```
Credential Types:

1. Claude OAuth tokens (.claude-config/credentials.json)
   - Stored per-project
   - Managed by Claude Code
   - Not shared across projects
   - Never committed to git

2. gh CLI OAuth tokens (.claude-config/gh/hosts.yml)
   - Stored per-project
   - Managed by configure_gh_cli() in setup_dev_environment.sh
   - Not shared across projects
   - Never committed to git (gitignore gate enforces this)
   - git protocol: HTTPS (--git-protocol https passed explicitly)
   - git credential helper registered via gh auth setup-git (enables HTTPS push/pull)

3. Service credentials (.env)
   - Stored per-project
   - Managed by developers
   - Contains: Snowflake, AWS, CloudFlare credentials
   - Never committed to git

4. SSH keys (.git/config: core.sshCommand)
   - Stored per-project git config
   - Optional: SSH key setup is available but not required for git transport
   - HTTPS + gh credential helper is the default transport method
```

---

**Last Updated:** 2026-02-28 (Phase 6 — HTTPS transport, gh auth setup-git post-login step, update three-path decision tree)
**Maintained By:** ALI AI Team
