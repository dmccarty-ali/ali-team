# ali-infrastructure - Scripts Inventory Reference

This document provides complete documentation for all ali-ai infrastructure scripts, including detailed usage, options, implementation notes, and examples.

---

## Core Launcher & Sync Scripts

### claude-ali

**Location:** ~/.claude/bin/claude-ali

**Purpose:** Unified Claude Code launcher with subscription and resource isolation.

**Full Usage:**
```bash
claude-ali [OPTIONS]

OPTIONS:
  --personal          Use personal subscription (~/.config/claude-personal/)
  --sync-org          Sync org resources even in personal mode
  --force-sync        Force resource sync (ignore hash check)
  --help              Display this help message
  --version           Show version information
  --debug             Enable debug output

EXAMPLES:
  # Org mode (default) - company subscription + org resources
  cd ~/ali-ai/projects/my-project
  claude-ali

  # Personal mode - personal subscription, no org resources
  claude-ali --personal

  # Personal mode with org resources
  claude-ali --personal --sync-org

  # Force re-sync
  claude-ali --force-sync

  # Debug mode
  claude-ali --debug
```

**Features:**
- Automatic org resource sync from ~/ali-ai
- Per-project credential isolation (.claude-config/)
- Hash-based change detection (fast ~6ms check)
- Subscription isolation (org vs personal)
- Session protocol enforcement
- Remote update check (cached 30min)
- Hooks configuration merge
- Migration execution

**Requirements:**
- .aliunde-project marker in project directory (for org mode)
- ~/ali-ai cloned and up to date
- jq (optional, for settings.json merge)

**Exit codes:**
- 0: Successful launch
- 1: Error (environment validation, sync failure)

**Implementation notes:**
- Size: 412 lines, bash
- Uses rsync for efficient file syncing
- Implements cache for remote update checks
- Falls back to template replacement if jq not available
- Sets NODE_OPTIONS for 8GB heap (multi-agent orchestration)

**Environment variables set:**
- NODE_OPTIONS="--max-old-space-size=8192"
- CLAUDE_CONFIG_DIR (varies by mode)
- ANTHROPIC_API_KEY (explicitly unset)

---

### sync-org-resources.sh

**Location:** ~/.claude/bin/sync-org-resources.sh

**Purpose:** Sync org resources from ~/ali-ai to project .claude/ directory.

**Note:** Usually not called directly - claude-ali handles sync automatically.

**Full Usage:**
```bash
sync-org-resources.sh [OPTIONS] [PROJECT_DIR]

OPTIONS:
  --dry-run           Preview what would sync (no changes)
  --force             Force sync (ignore hash check)
  --verbose           Verbose output (show all rsync operations)
  --help              Display this help message

ARGUMENTS:
  PROJECT_DIR         Target project directory (default: current directory)

EXAMPLES:
  # Normal sync (usually done by claude-ali)
  ~/.claude/bin/sync-org-resources.sh

  # Preview what would sync
  ~/.claude/bin/sync-org-resources.sh --dry-run

  # Force sync (ignore hash check)
  ~/.claude/bin/sync-org-resources.sh --force

  # Verbose output
  ~/.claude/bin/sync-org-resources.sh --verbose

  # Sync specific project
  ~/.claude/bin/sync-org-resources.sh ~/ali-ai/projects/tax-practice
```

**What it syncs:**
- ~/ali-ai/agents/ → .claude/agents/
- ~/ali-ai/skills/ → .claude/skills/
- ~/ali-ai/hooks/ → .claude/hooks/
- .claude-local/* → .claude/* (overlay, project-specific resources)

**Exit codes:**
- 0: Success
- 1: Error (validation or rsync failure)
- 2: Skipped (already in sync)

**Performance:**
- Hash check only: ~6ms
- Full sync: ~200ms
- Incremental sync: ~50ms

**Implementation notes:**
- Size: 287 lines, bash
- Uses rsync --delete flag (removes obsolete files)
- Creates .last-sync hash file for change detection
- Overlays .claude-local/ resources after org sync
- Validates ~/ali-ai is a git repository

**Dependencies:**
- rsync (required)
- git (required)

---

## Project Setup & Initialization Scripts

### init-aliunde-project.sh

**Location:** ~/.claude/bin/init-aliunde-project.sh

**Purpose:** Initialize or convert project for ali-ai infrastructure.

**Full Usage:**
```bash
init-aliunde-project.sh [OPTIONS] [PROJECT_DIR]

OPTIONS:
  --dry-run           Preview changes without writing files
  --force             Overwrite existing files
  --no-gitignore      Skip .gitignore updates
  --no-claude-md      Skip CLAUDE.md creation
  --help              Display this help message
  --verbose           Verbose output

ARGUMENTS:
  PROJECT_DIR         Project directory to initialize (default: current directory)

EXAMPLES:
  # Initialize current directory as Aliunde project
  cd /path/to/project
  ~/.claude/bin/init-aliunde-project.sh

  # Preview what would change
  ~/.claude/bin/init-aliunde-project.sh --dry-run

  # Force overwrite existing configuration
  ~/.claude/bin/init-aliunde-project.sh --force

  # Initialize specific project
  ~/.claude/bin/init-aliunde-project.sh ~/ali-ai/projects/new-project
```

**What it does:**
- Detects project state (new vs existing Claude Code project)
- Migrates custom agents/skills/hooks to .claude-local/
- Creates .aliunde-project marker
- Updates .gitignore with Claude entries
- Creates CLAUDE.md from template
- Sets up directory structure (.claude-local/)
- Creates README.md in .claude-local/ explaining purpose

**Key flags:**
- --dry-run: Preview changes without writing files (safe to run)
- --force: Overwrite existing files (use with caution)

**Exit codes:**
- 0: Success
- 1: Error (validation or file operation failure)

**Implementation notes:**
- Size: 783 lines, bash
- Detects existing Claude Code project (has .claude/ or custom agents/skills)
- Preserves custom resources by moving to .claude-local/
- Uses template from ~/ali-ai/templates/CLAUDE.md
- Creates backups before overwriting files

**Migration logic:**
```bash
# If .claude/ exists with custom resources:
1. Create .claude-local/ directory
2. Move custom agents → .claude-local/agents/
3. Move custom skills → .claude-local/skills/
4. Move custom hooks → .claude-local/hooks/
5. Delete .claude/ (will be recreated by sync)
6. Create .aliunde-project marker
```

---

### setup_dev_environment.sh

**Location:** ~/.claude/bin/setup_dev_environment.sh

**Purpose:** Interactive setup wizard for new team members.

**Full Usage:**
```bash
setup_dev_environment.sh [OPTIONS] [PROJECT_DIR]

OPTIONS:
  --non-interactive   Non-interactive mode (use defaults)
  --skip-git          Skip Git/GitHub configuration
  --skip-services     Skip service credential configuration
  --skip-python       Skip Python venv creation
  --help              Display this help message

ARGUMENTS:
  PROJECT_DIR         Project directory (default: current directory)

EXAMPLES:
  # Interactive setup
  cd /path/to/project
  ~/.claude/bin/setup_dev_environment.sh

  # Non-interactive with defaults
  ~/.claude/bin/setup_dev_environment.sh --non-interactive

  # Skip Git configuration
  ~/.claude/bin/setup_dev_environment.sh --skip-git
```

**When to use:**
- New team member joins project
- Fresh clone of project repository
- Setting up development environment on new machine
- Configuring multiple Claude accounts (personal + work)

**What it configures:**

1. **Claude credential isolation (always)**
   - Creates .claude-config/ directory
   - Enables multiple simultaneous Claude sessions with different accounts
   - Isolates OAuth tokens per-project
   - Merges hooks configuration from template

2. **Git/GitHub SSH (optional)**
   - Prompts for git user.name and git user.email
   - Configures locally in .git/config
   - Lists existing SSH keys or creates new ed25519 key
   - Sets up core.sshCommand for per-project key usage
   - Tests GitHub connection with selected key
   - Provides instructions for adding key to GitHub
   - Calls configure_gh_cli() to isolate gh CLI credentials per project (three-path flow):
     - Path 1 (fast path): hosts.yml exists and gh auth status valid — skip login
     - Path 2 (backward compat, deprecated): PAT token file found — gh auth login --with-token (removed in v0.4.0)
     - Path 3 (primary): gh auth login --web (browser OAuth)
   - Sets GH_CONFIG_DIR=.claude-config/gh/ for per-project gh CLI isolation
   - gitignore hard gate: auto-appends .claude-config/ to .gitignore before any credential write
   - Post-login account verification via gh api user — mismatch removes .claude-config/gh/ and returns error

3. **Service credentials (optional)**
   - Snowflake: account, user, database, warehouse, role
   - AWS: profile name, region
   - CloudFlare: account ID, API token
   - Writes credentials to .env file (gitignored)
   - Sets restrictive permissions (chmod 600)

4. **Python virtual environment (optional)**
   - Creates .venv/ directory
   - Ready for pip install -r requirements.txt
   - Provides activation instructions

**Exit codes:**
- 0: Success
- 1: Error (configuration failure)

**Implementation notes:**
- Size: 461 lines, bash
- Interactive prompts with default values
- Validates inputs (email format, directory existence)
- Creates backups before modifying files
- Adds .env to .gitignore if not present
- SSH key generation uses ed25519 (modern, secure)

**Output files:**
- .claude-config/settings.json (hooks configuration)
- .claude-config/gh/ (gh CLI credential store, per-project)
- .env (service credentials, gitignored)
- .git/config (updated with user info and SSH key)
- .venv/ (Python virtual environment, if selected)

---

### setup-colleague.sh

**Location:** ~/.claude/bin/setup-colleague.sh

**Purpose:** One-liner setup for new Aliunde colleagues.

**Full Usage:**
```bash
# Remote installation (recommended for new colleagues)
curl -sSL https://raw.githubusercontent.com/dmccarty-ali/ali-ai/main/scripts/setup-colleague.sh | bash

# Or if you've already cloned ali-ai
~/.claude/bin/setup-colleague.sh [OPTIONS]

OPTIONS:
  --skip-backups      Skip creating backups of existing installation
  --force             Force installation (overwrite without prompts)
  --help              Display this help message
```

**What it does:**
1. Checks prerequisites (git, Claude CLI, SSH key)
2. Installs missing prerequisites automatically
3. Clones or updates ~/ali-ai repository
4. Makes scripts executable (chmod +x)
5. Sets up shell alias (claude-ali) in ~/.zshrc or ~/.bashrc
6. Creates backups before modifying existing installations
7. Provides next steps instructions

**Prerequisites installed automatically:**
- **git:** via Homebrew (if available) or Xcode CLI Tools on macOS
- **Claude Code CLI:** via npm (requires Node.js)
- **SSH key:** ed25519 key generated if missing

**Interactive prompts:**
- GitHub email (for SSH key generation)
- Add SSH key to GitHub (opens browser automatically)
- GitHub collaborator invite acceptance (if applicable)

**Exit codes:**
- 0: Success
- 1: Error (prerequisite installation or git clone failure)

**Implementation notes:**
- Size: 356 lines, bash
- Detects OS (macOS, Linux)
- Checks for Homebrew, installs if needed (macOS)
- Uses apt-get on Linux
- Tests GitHub connection after SSH key setup
- Creates timestamped backups in ~/.ali-ai-backups/

**Output:**
- ~/ali-ai/ cloned and up to date
- claude-ali alias added to shell profile
- Backups in ~/.ali-ai-backups/YYYY-MM-DD-HHMMSS/ (if updating)

**Shell alias added:**
```bash
# Added to ~/.zshrc or ~/.bashrc
alias claude-ali="~/.claude/bin/claude-ali"
```

---

## Agent & Manifest Scripts

### build_agent_manifest.py

**Location:** ~/.claude/bin/build_agent_manifest.py

**Purpose:** Build AGENT_MANIFEST.json from expert agent frontmatter.

**Full Usage:**
```python
python scripts/build_agent_manifest.py [OPTIONS]

OPTIONS:
  --agents-dir DIR    Agents directory (default: agents/)
  --output FILE       Output file path (default: agents/AGENT_MANIFEST.json)
  --verbose           Verbose output
  --validate-only     Validate metadata without writing manifest
  --help              Display this help message

EXAMPLES:
  # Extract from default location
  cd ~/ali-ai
  python scripts/build_agent_manifest.py

  # Specify custom agents directory
  python scripts/build_agent_manifest.py --agents-dir ~/.claude/agents

  # Custom output path
  python scripts/build_agent_manifest.py --output /path/to/manifest.json

  # Validate metadata coverage
  python scripts/build_agent_manifest.py --validate-only --verbose
```

**When to use:**
- After creating new expert agent
- After updating expert-metadata in agent file
- After changing agent domains, keywords, or file-patterns
- Automatically called by claude-ali during sync

**What it does:**
1. Scans agents directory for *.md files
2. Extracts YAML frontmatter from each file
3. Parses expert-metadata section:
   - domains (list of expertise domains)
   - keywords (list of trigger keywords)
   - file-patterns (list of glob patterns)
   - anti-keywords (list of negative keywords)
4. Writes consolidated manifest to AGENT_MANIFEST.json
5. Reports metadata coverage statistics

**Output format:**
```json
{
  "generated_at": "2026-01-25T10:30:00Z",
  "source_dir": "/Users/user/ali-ai/agents",
  "total_agents": 42,
  "expert_agents": 27,
  "agents": [
    {
      "file": "/path/to/ali-security-expert.md",
      "name": "ali-security-expert",
      "description": "Security expert for...",
      "domains": ["security", "compliance"],
      "keywords": ["authentication", "OWASP", "encryption"],
      "file_patterns": ["**/auth/**", "**/security/**"],
      "anti_keywords": ["frontend", "ui"]
    }
  ],
  "coverage": {
    "with_domains": 27,
    "with_keywords": 27,
    "with_file_patterns": 24,
    "with_anti_keywords": 18
  }
}
```

**Exit codes:**
- 0: Success
- 1: Error (file parsing or writing failure)

**Implementation notes:**
- Size: 206 lines, Python
- Uses PyYAML for frontmatter parsing
- Validates YAML structure
- Reports agents missing expert-metadata
- Handles malformed frontmatter gracefully

**Dependencies:**
- Python 3.7+
- PyYAML (pip install pyyaml)

**Example output:**
```
Building agent manifest...
Found 42 agent files
Extracting metadata...
  ✓ ali-security-expert (4 domains, 8 keywords, 3 patterns)
  ✓ ali-database-expert (3 domains, 12 keywords, 5 patterns)
  ⚠ ali-generic-expert (no expert-metadata)
...
Manifest written to: agents/AGENT_MANIFEST.json
Coverage: 27/42 agents have expert metadata (64%)
```

---

### list_experts.py

**Location:** ~/.claude/bin/list_experts.py

**Purpose:** Score expert agents against plan files and architecture stack.

**Full Usage:**
```python
python scripts/list_experts.py [OPTIONS] [PLAN_FILE]

OPTIONS:
  --manifest FILE     Path to AGENT_MANIFEST.json (default: ./.claude/AGENT_MANIFEST.json)
  --stack FILE        Path to architecture_stack.txt (for stack boosts)
  --threshold N       Minimum score threshold (default: 0)
  --top N             Return only top N experts (default: all)
  --format json|text  Output format (default: json)
  --verbose           Verbose output
  --help              Display this help message

ARGUMENTS:
  PLAN_FILE           Path to plan file to score against (optional)

EXAMPLES:
  # List all experts from manifest
  python scripts/list_experts.py

  # Score experts against a plan file
  python scripts/list_experts.py path/to/plan.md

  # Apply stack boosts from architecture_stack.txt
  python scripts/list_experts.py --stack architecture_stack.txt path/to/plan.md

  # Filter by threshold (score >= 5)
  python scripts/list_experts.py --threshold 5 plan.md

  # Return only top 3 experts
  python scripts/list_experts.py --top 3 plan.md

  # Text output format
  python scripts/list_experts.py --format text plan.md
```

**Scoring rules:**
- Each keyword match: +1 point
- Each anti-keyword match: -2 points
- Stack boosts: +3 to +5 points (when stack file provided)
- File pattern match: +2 points (if plan mentions matching files)

**Stack boost mappings:**
```python
STACK_BOOSTS = {
    'postgresql': {'database-expert': 5, 'backend-expert': 2},
    'aurora': {'database-expert': 5, 'aws-expert': 3},
    'langgraph': {'ai-expert': 5, 'backend-expert': 2},
    'terraform': {'infrastructure-expert': 5, 'aws-expert': 3},
    'fastapi': {'backend-expert': 3, 'api-expert': 2},
    'react': {'frontend-expert': 3, 'ui-expert': 2},
    'hipaa': {'security-expert': 5, 'compliance-expert': 5},
    'wisp': {'security-expert': 5, 'compliance-expert': 5},
}
```

**Output (JSON):**
```json
[
  {
    "name": "ali-database-expert",
    "score": 12,
    "reasons": [
      "Keyword match: postgresql (+1)",
      "Keyword match: migrations (+1)",
      "Stack boost: postgresql→database-expert (+5)",
      "Stack boost: aurora→database-expert (+5)"
    ]
  },
  {
    "name": "ali-backend-expert",
    "score": 5,
    "reasons": [
      "Keyword match: api (+1)",
      "Stack boost: fastapi→backend-expert (+3)",
      "Anti-keyword match: frontend (-2)"
    ]
  }
]
```

**Output (Text):**
```
Experts ranked by relevance:

1. ali-database-expert (score: 12)
   - Keyword match: postgresql (+1)
   - Keyword match: migrations (+1)
   - Stack boost: postgresql→database-expert (+5)
   - Stack boost: aurora→database-expert (+5)

2. ali-backend-expert (score: 5)
   - Keyword match: api (+1)
   - Stack boost: fastapi→backend-expert (+3)
   - Anti-keyword match: frontend (-2)
```

**Exit codes:**
- 0: Success
- 1: Error (file not found, parsing failure)

**Implementation notes:**
- Size: 215 lines, Python
- Case-insensitive keyword matching
- Partial keyword matching (e.g., "auth" matches "authentication")
- Handles missing manifest gracefully

**Use cases:**
- expert-panel agent selection
- Automated expert routing based on plan content
- Plan file analysis and expert recommendations

---

### regenerate-agents.sh

**Location:** ~/.claude/bin/regenerate-agents.sh

**Purpose:** Validate agents directory exists and is ready for sync.

**Full Usage:**
```bash
regenerate-agents.sh [OPTIONS] [AGENTS_DIR]

OPTIONS:
  --verbose           Verbose output (list all agents found)
  --help              Display this help message

ARGUMENTS:
  AGENTS_DIR          Agents directory to validate (default: ~/ali-ai/agents)

EXAMPLES:
  # Validate agents (normal use)
  ~/.claude/bin/regenerate-agents.sh

  # Verbose output
  ~/.claude/bin/regenerate-agents.sh --verbose

  # Validate custom agents directory
  ~/.claude/bin/regenerate-agents.sh ~/.claude/agents
```

**What it does:**
- Validates agents directory exists
- Counts ali-*.md agent files
- Previously compiled agents (legacy), now just validates
- Reports count and status

**Exit codes:**
- 0: Success (agents validated)
- 1: Error (agents directory missing or no agent files)

**Implementation notes:**
- Size: 217 lines, bash
- Called automatically by claude-ali during sync
- Backward compatibility: Used to compile agents, now validates only
- Reports warning if no ali-prefixed agents found

**Example output:**
```
Validating agents directory...
Found 42 agent files (ali-*.md)
Validation complete
```

---

## Migration Scripts

### run-pending-migrations.sh

**Location:** ~/.claude/bin/run-pending-migrations.sh

**Purpose:** Execute pending migrations for a project.

**Full Usage:**
```bash
run-pending-migrations.sh [OPTIONS] [PROJECT_DIR]

OPTIONS:
  --verbose           Verbose output (show migration details)
  --dry-run           List pending migrations without executing
  --force             Force re-run all migrations (ignore registry)
  --help              Display this help message

ARGUMENTS:
  PROJECT_DIR         Project directory (default: current directory)

EXAMPLES:
  # Run pending migrations for current project
  ~/.claude/bin/run-pending-migrations.sh

  # Run for specific project
  ~/.claude/bin/run-pending-migrations.sh /path/to/project

  # Verbose output
  ~/.claude/bin/run-pending-migrations.sh --verbose

  # List pending migrations without executing
  ~/.claude/bin/run-pending-migrations.sh --dry-run
```

**How it works:**
1. Compares available migrations (~/ali-ai/migrations/*.sh) to registry (.claude/.migration-registry.json)
2. Executes pending migrations sequentially (sorted by number)
3. Stops on first failure (doesn't proceed to next migration)
4. Updates registry with results (success/failure, duration, output)
5. Logs to .claude/logs/migrations.log

**Registry tracking:**
- Per-project registry: .claude/.migration-registry.json
- Tracks: migration ID, execution time, status, duration, ali-ai hash
- Prevents duplicate execution

**Exit codes:**
- 0: Success (all migrations completed or already migrated)
- 1: Failure (migration failed, subsequent migrations skipped)

**Implementation notes:**
- Size: 113 lines, bash
- Uses jq for registry management
- Captures stdout and stderr from migrations
- Measures execution time in milliseconds
- Creates registry if missing

**Logging:**
- Log file: .claude/logs/migrations.log
- Includes: timestamp, migration ID, status, duration, errors
- Appends to existing log (never truncates)

**Example output:**
```
Running pending migrations...
[2026-01-26 18:45:32] Running migration: 001-skill-prefix-cleanup
[2026-01-26 18:45:32]   SUCCESS (127ms)
[2026-01-26 18:45:32] Running migration: 002-hook-rename-cleanup
[2026-01-26 18:45:32]   SUCCESS (89ms)
[2026-01-26 18:45:33] Running migration: 003-settings-version-update
[2026-01-26 18:45:34]   SUCCESS (1023ms)
All migrations complete
```

---

## Cleanup & Validation Scripts

### cleanup-old-skills.sh

**Location:** ~/.claude/bin/cleanup-old-skills.sh

**Purpose:** Remove old unprefixed org skills from project .claude/skills/

**Full Usage:**
```bash
cleanup-old-skills.sh [OPTIONS] [PROJECT_DIR]

OPTIONS:
  --dry-run           Preview what would be removed (no changes)
  --verbose           Verbose output (list all skills checked)
  --help              Display this help message

ARGUMENTS:
  PROJECT_DIR         Project directory (default: current directory)

EXAMPLES:
  # Run from Aliunde project directory
  cd /path/to/project
  ~/.claude/bin/cleanup-old-skills.sh

  # Preview what would be removed
  ~/.claude/bin/cleanup-old-skills.sh --dry-run

  # Verbose output
  ~/.claude/bin/cleanup-old-skills.sh --verbose
```

**What it does:**
- Checks for old unprefixed skill directories (agent-operations, backend-developer, etc.)
- Removes only KNOWN org skills (preserves project-specific skills)
- Reports count of removed skills
- Lists remaining skills after cleanup

**Old skills removed:**
```bash
OLD_SKILLS=(
    "agent-operations"      # → ali-agent-operations
    "backend-developer"     # → ali-backend-developer
    "claude-code"           # → ali-claude-code
    "database-developer"    # → ali-database-developer
    "frontend-developer"    # → ali-frontend-developer
    "infrastructure"        # → ali-infrastructure
    "security-review"       # → ali-security-review
    # ... 40+ total
)
```

**Exit codes:**
- 0: Success
- 1: Error (directory not found)

**Implementation notes:**
- Size: 96 lines, bash
- Hardcoded list of known org skill names
- Does NOT remove project-specific skills
- No --force flag (safe to run multiple times)
- Idempotent (safe to run repeatedly)

**Safety:**
- Only removes skills in hardcoded list
- Checks if directory exists before removal
- Does NOT use wildcards or pattern matching

**Example output:**
```
Cleaning up old org skills...
Removing: agent-operations
Removing: backend-developer
Removing: claude-code
Removed 3 old skill(s)

Remaining skills:
- ali-agent-operations
- ali-backend-developer
- ali-claude-code
- tax-practice-expert (project-specific)
```

---

### verify-expert-evidence.sh

**Location:** ~/.claude/bin/verify-expert-evidence.sh

**Purpose:** Verify all expert agents have Operating Procedure section with evidence requirements.

**Full Usage:**
```bash
verify-expert-evidence.sh [OPTIONS] [AGENTS_DIR]

OPTIONS:
  --verbose           Verbose output (list all checks)
  --help              Display this help message

ARGUMENTS:
  AGENTS_DIR          Agents directory (default: ~/ali-ai/agents)

EXAMPLES:
  # Verify experts in default location
  ~/.claude/bin/verify-expert-evidence.sh

  # Verify custom agents directory
  ~/.claude/bin/verify-expert-evidence.sh /path/to/agents

  # Verbose output
  ~/.claude/bin/verify-expert-evidence.sh --verbose
```

**What it checks:**
- Presence of ## Operating Procedure section in expert agents
- All expected expert agents exist (hardcoded list of 27 experts)

**Expected expert agents:**
```bash
EXPECTED_EXPERTS=(
    "ali-ai-expert"
    "ali-airflow-expert"
    "ali-aws-expert"
    "ali-backend-expert"
    "ali-database-expert"
    "ali-frontend-expert"
    "ali-infrastructure-expert"
    "ali-security-expert"
    # ... 27 total
)
```

**Reports:**
- Compliant experts (have Operating Procedure section)
- Missing section (expert file exists but missing section)
- Missing file (expected expert file not found)
- Compliance percentage

**Exit codes:**
- 0: All experts compliant (100%)
- 1: Compliance incomplete (<100%)

**Implementation notes:**
- Size: 90 lines, bash
- Uses grep to search for section header
- Hardcoded list of expected experts (update when adding new experts)

**Example output:**
```
Verifying expert agent evidence requirements...

Compliant: ali-security-expert
Compliant: ali-database-expert
Compliant: ali-backend-expert
...
Missing section: ali-frontend-expert
Missing file: ali-new-expert

Results:
- Compliant: 24/27 (89%)
- Missing section: 2
- Missing file: 1

FAILED: Not all experts have evidence requirements
```

---

**Last Updated:** 2026-02-28
**Maintained By:** ALI AI Team
