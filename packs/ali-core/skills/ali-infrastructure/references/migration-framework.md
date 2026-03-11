# ali-infrastructure - Migration Framework Reference

This document provides comprehensive documentation for the ali-ai migration framework, including detailed structure requirements, idempotency patterns, and example implementations.

---

## Migration Framework Overview

The migration framework provides automated, zero-touch organizational updates when ali-ai changes require transformation of existing project resources.

### Core Design Principles

1. **Idempotent execution:** All migrations must be safe to run multiple times
2. **Per-project tracking:** Each project maintains its own migration registry
3. **Stop-on-failure:** First migration failure halts remaining migrations
4. **Automatic execution:** Migrations run automatically during claude-ali launch
5. **Comprehensive logging:** All migration activity logged for debugging

---

## Migration Structure (Detailed)

### Full Template with Comments

```bash
#!/bin/bash
# Migration: NNN-descriptive-slug
# Description: What transformation this performs (one-line summary)
# Created: YYYY-MM-DD
# Author: Your Name
# Idempotent: Yes (MUST be Yes)

set -euo pipefail

# Migration metadata
MIGRATION_ID="NNN-descriptive-slug"
PROJECT_DIR="${1:-.}"  # Accept project dir as argument, default to current

# Logging helper
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" >&2
}

# Error handler
error() {
    log "ERROR: $*"
    exit 1
}

# ─────────────────────────────────────────────────────────
# Step 1: Validate Environment
# ─────────────────────────────────────────────────────────
# Check that we're in a valid Aliunde project
if [[ ! -f "$PROJECT_DIR/.aliunde-project" ]]; then
    log "SKIP: Not an Aliunde project (no .aliunde-project marker)"
    exit 0
fi

# Check that target directory exists
TARGET_DIR="$PROJECT_DIR/.claude/skills"
if [[ ! -d "$TARGET_DIR" ]]; then
    log "SKIP: No .claude/skills/ directory found"
    exit 0
fi

# ─────────────────────────────────────────────────────────
# Step 2: Check Idempotency (Already Migrated?)
# ─────────────────────────────────────────────────────────
# Detect if migration already complete
# Strategy 1: Check if old resource no longer exists
if [[ ! -d "$TARGET_DIR/old-skill-name" ]]; then
    log "SKIP: Migration already complete (old resource not found)"
    exit 0
fi

# Strategy 2: Check if new resource already exists
if [[ -f "$TARGET_DIR/new-marker-file" ]]; then
    log "SKIP: Migration already complete (new marker found)"
    exit 0
fi

# Strategy 3: Check a version marker
if grep -q "version: 2.0" "$TARGET_DIR/config.yaml" 2>/dev/null; then
    log "SKIP: Migration already complete (version marker found)"
    exit 0
fi

# ─────────────────────────────────────────────────────────
# Step 3: Perform Migration
# ─────────────────────────────────────────────────────────
log "Starting migration: $MIGRATION_ID"

# Example: Remove old skill directory
if [[ -d "$TARGET_DIR/old-skill-name" ]]; then
    log "  Removing: $TARGET_DIR/old-skill-name"
    rm -rf "$TARGET_DIR/old-skill-name" || error "Failed to remove old skill"
fi

# Example: Rename files
if [[ -f "$TARGET_DIR/old-name.md" ]]; then
    log "  Renaming: old-name.md → new-name.md"
    mv "$TARGET_DIR/old-name.md" "$TARGET_DIR/new-name.md" || error "Failed to rename"
fi

# Example: Update configuration
if [[ -f "$TARGET_DIR/config.yaml" ]]; then
    log "  Updating: config.yaml version"
    sed -i.bak 's/version: 1.0/version: 2.0/' "$TARGET_DIR/config.yaml" || error "Failed to update config"
    rm -f "$TARGET_DIR/config.yaml.bak"  # Clean up backup
fi

# ─────────────────────────────────────────────────────────
# Step 4: Verify Success
# ─────────────────────────────────────────────────────────
# Verify expected post-migration state
if [[ -d "$TARGET_DIR/old-skill-name" ]]; then
    error "Migration verification failed: old skill still exists"
fi

log "Migration complete: $MIGRATION_ID"
exit 0
```

### Migration Metadata Fields

| Field | Required | Purpose | Example |
|-------|----------|---------|---------|
| Migration | Yes | Filename (NNN-slug) | 001-skill-prefix-cleanup |
| Description | Yes | One-line summary | Remove old unprefixed org skills |
| Created | Yes | Creation date | 2026-01-26 |
| Author | No | Creator name | Don McCarty |
| Idempotent | Yes | Must be "Yes" | Yes |

---

## Idempotency Patterns (Detailed)

### Pattern 1: Check-Before-Act (File/Directory Operations)

```bash
# BAD: Non-idempotent (fails on second run)
rm -rf "$DIR/old-skill"  # Fails if already removed

# GOOD: Idempotent (safe to run multiple times)
if [[ -d "$DIR/old-skill" ]]; then
    rm -rf "$DIR/old-skill"
fi
# Or use -f flag:
rm -rf "$DIR/old-skill"  # -r flag makes rm not fail if missing
```

### Pattern 2: Skip-If-Complete (State Detection)

```bash
# Detect already-migrated state at the START
if [[ ! -d "$DIR/old-resource" ]]; then
    echo "SKIP: Migration already complete"
    exit 0  # Exit early, no work needed
fi

# Proceed with migration...
rm -rf "$DIR/old-resource"
```

### Pattern 3: Conditional Updates (Partial State)

```bash
# Update configuration only if old value present
if grep -q "old_value" "$CONFIG_FILE"; then
    sed -i 's/old_value/new_value/' "$CONFIG_FILE"
    echo "Updated: $CONFIG_FILE"
else
    echo "SKIP: $CONFIG_FILE already updated"
fi
```

### Pattern 4: Force Flags (Overwrite-Safe)

```bash
# Copy files with force flag (no fail if exists)
cp -f "$SOURCE" "$DEST"

# Remove with force (no fail if missing)
rm -f "$FILE"

# Move with force (overwrites if exists)
mv -f "$OLD" "$NEW"
```

### Pattern 5: Version Markers (Configuration Evolution)

```bash
# Check current version
CURRENT_VERSION=$(grep "version:" "$CONFIG" | awk '{print $2}')

if [[ "$CURRENT_VERSION" == "1.0" ]]; then
    # Apply 1.0 → 2.0 migration
    sed -i 's/version: 1.0/version: 2.0/' "$CONFIG"
    # Add new fields for 2.0
    echo "new_field: default_value" >> "$CONFIG"
elif [[ "$CURRENT_VERSION" == "2.0" ]]; then
    echo "SKIP: Already at version 2.0"
    exit 0
else
    echo "ERROR: Unknown version: $CURRENT_VERSION"
    exit 1
fi
```

### Pattern 6: Atomic Operations (All-or-Nothing)

```bash
# Create temp directory for atomic operations
TEMP_DIR=$(mktemp -d)
trap "rm -rf '$TEMP_DIR'" EXIT  # Cleanup on exit

# Perform operations in temp directory
cp -r "$SOURCE_DIR" "$TEMP_DIR/staged"
# ... modify files in $TEMP_DIR/staged ...

# Atomic swap (all-or-nothing)
mv "$TARGET_DIR" "$TARGET_DIR.backup"
mv "$TEMP_DIR/staged" "$TARGET_DIR"
rm -rf "$TARGET_DIR.backup"
```

---

## Registry Format (Detailed)

### Registry File Structure

```json
{
  "registry_version": "1.0",
  "migrations": [
    {
      "id": "001-skill-prefix-cleanup",
      "executed_at": "2026-01-26T18:45:32Z",
      "status": "success",
      "duration_ms": 127,
      "ali_ai_hash": "abc123f",
      "exit_code": 0,
      "stdout": "Removed 3 old skills",
      "stderr": ""
    },
    {
      "id": "002-hook-rename-cleanup",
      "executed_at": "2026-01-26T18:45:33Z",
      "status": "success",
      "duration_ms": 89,
      "ali_ai_hash": "abc123f",
      "exit_code": 0,
      "stdout": "Renamed ali-git-validate.sh → ali-git-guard.sh",
      "stderr": ""
    },
    {
      "id": "003-settings-version-update",
      "executed_at": "2026-01-26T18:45:34Z",
      "status": "failed",
      "duration_ms": 1023,
      "ali_ai_hash": "abc123f",
      "exit_code": 1,
      "stdout": "Merging settings.json...",
      "stderr": "ERROR: jq parse failed"
    }
  ]
}
```

### Registry Fields

| Field | Type | Purpose | Example |
|-------|------|---------|---------|
| registry_version | string | Schema version | "1.0" |
| migrations | array | List of executed migrations | [...] |
| id | string | Migration identifier | "001-skill-prefix-cleanup" |
| executed_at | string | ISO 8601 timestamp | "2026-01-26T18:45:32Z" |
| status | string | success / failed / skipped | "success" |
| duration_ms | integer | Execution time in milliseconds | 127 |
| ali_ai_hash | string | Git commit hash of ~/ali-ai | "abc123f" |
| exit_code | integer | Migration exit code | 0 |
| stdout | string | Migration output | "Removed 3 old skills" |
| stderr | string | Migration errors | "" |

### Registry Operations

**Read registry:**
```bash
REGISTRY="$PROJECT_DIR/.claude/.migration-registry.json"
if [[ -f "$REGISTRY" ]]; then
    COMPLETED=$(jq -r '.migrations[].id' "$REGISTRY")
else
    COMPLETED=""
fi
```

**Check if migration completed:**
```bash
if echo "$COMPLETED" | grep -q "^$MIGRATION_ID$"; then
    echo "Already executed"
    exit 0
fi
```

**Add successful migration:**
```bash
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
HASH=$(cd ~/ali-ai && git rev-parse HEAD)

jq --arg id "$MIGRATION_ID" \
   --arg ts "$TIMESTAMP" \
   --arg hash "$HASH" \
   --argjson duration "$DURATION_MS" \
   '.migrations += [{
     id: $id,
     executed_at: $ts,
     status: "success",
     duration_ms: $duration,
     ali_ai_hash: $hash,
     exit_code: 0
   }]' "$REGISTRY" > "$REGISTRY.tmp"

mv "$REGISTRY.tmp" "$REGISTRY"
```

---

## Migration Execution Flow (Detailed)

### run-pending-migrations.sh Logic

```bash
#!/bin/bash
# Location: ~/.claude/bin/run-pending-migrations.sh

set -euo pipefail

PROJECT_DIR="${1:-.}"
MIGRATIONS_DIR="$HOME/ali-ai/migrations"
REGISTRY="$PROJECT_DIR/.claude/.migration-registry.json"
LOG_FILE="$PROJECT_DIR/.claude/logs/migrations.log"

# Initialize registry if missing
if [[ ! -f "$REGISTRY" ]]; then
    mkdir -p "$(dirname "$REGISTRY")"
    echo '{"registry_version":"1.0","migrations":[]}' > "$REGISTRY"
fi

# Initialize log file
mkdir -p "$(dirname "$LOG_FILE")"

# Get completed migration IDs
COMPLETED=$(jq -r '.migrations[].id' "$REGISTRY" 2>/dev/null || echo "")

# Find all available migrations (sorted by number)
AVAILABLE=$(ls -1 "$MIGRATIONS_DIR"/*.sh 2>/dev/null | sort -V || echo "")

# Filter to pending migrations
PENDING=""
for migration_path in $AVAILABLE; do
    migration_id=$(basename "$migration_path" .sh)

    if ! echo "$COMPLETED" | grep -q "^$migration_id$"; then
        PENDING="$PENDING $migration_path"
    fi
done

# Execute pending migrations
for migration_path in $PENDING; do
    migration_id=$(basename "$migration_path" .sh)

    echo "Running migration: $migration_id" | tee -a "$LOG_FILE"

    START_TIME=$(date +%s%3N)  # Milliseconds

    # Execute migration, capture output
    if OUTPUT=$("$migration_path" "$PROJECT_DIR" 2>&1); then
        END_TIME=$(date +%s%3N)
        DURATION=$((END_TIME - START_TIME))
        EXIT_CODE=0

        # Log success
        echo "  SUCCESS (${DURATION}ms)" | tee -a "$LOG_FILE"

        # Update registry
        TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
        HASH=$(cd ~/ali-ai && git rev-parse HEAD)

        jq --arg id "$migration_id" \
           --arg ts "$TIMESTAMP" \
           --arg hash "$HASH" \
           --argjson duration "$DURATION" \
           --arg stdout "$OUTPUT" \
           '.migrations += [{
             id: $id,
             executed_at: $ts,
             status: "success",
             duration_ms: $duration,
             ali_ai_hash: $hash,
             exit_code: 0,
             stdout: $stdout,
             stderr: ""
           }]' "$REGISTRY" > "$REGISTRY.tmp"

        mv "$REGISTRY.tmp" "$REGISTRY"
    else
        END_TIME=$(date +%s%3N)
        DURATION=$((END_TIME - START_TIME))
        EXIT_CODE=$?

        # Log failure
        echo "  FAILED (exit code: $EXIT_CODE)" | tee -a "$LOG_FILE"
        echo "$OUTPUT" | tee -a "$LOG_FILE"

        # Update registry with failure
        TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
        HASH=$(cd ~/ali-ai && git rev-parse HEAD)

        jq --arg id "$migration_id" \
           --arg ts "$TIMESTAMP" \
           --arg hash "$HASH" \
           --argjson duration "$DURATION" \
           --argjson exit_code "$EXIT_CODE" \
           --arg stderr "$OUTPUT" \
           '.migrations += [{
             id: $id,
             executed_at: $ts,
             status: "failed",
             duration_ms: $duration,
             ali_ai_hash: $hash,
             exit_code: $exit_code,
             stdout: "",
             stderr: $stderr
           }]' "$REGISTRY" > "$REGISTRY.tmp"

        mv "$REGISTRY.tmp" "$REGISTRY"

        # STOP: Don't run further migrations
        echo "ERROR: Migration $migration_id failed. Stopping." | tee -a "$LOG_FILE"
        exit 1
    fi
done

echo "All migrations complete" | tee -a "$LOG_FILE"
exit 0
```

---

## Example Migrations (Detailed)

### Example 1: Skill Prefix Cleanup (Resource Removal)

```bash
#!/bin/bash
# Migration: 001-skill-prefix-cleanup
# Description: Remove old unprefixed org skills (agent-operations → ali-agent-operations)
# Created: 2026-01-26
# Idempotent: Yes

set -euo pipefail

MIGRATION_ID="001-skill-prefix-cleanup"
PROJECT_DIR="${1:-.}"

# Validate environment
SKILLS_DIR="$PROJECT_DIR/.claude/skills"
if [[ ! -d "$SKILLS_DIR" ]]; then
    echo "SKIP: No .claude/skills/ directory found"
    exit 0
fi

# List of old skills to remove (now prefixed with ali-)
OLD_SKILLS=(
    "agent-operations"
    "backend-developer"
    "claude-code"
    "database-developer"
    "frontend-developer"
    "infrastructure"
    "security-review"
)

REMOVED_COUNT=0

# Remove each old skill if it exists
for skill in "${OLD_SKILLS[@]}"; do
    if [[ -d "$SKILLS_DIR/$skill" ]]; then
        echo "Removing old skill: $skill"
        rm -rf "$SKILLS_DIR/$skill"
        ((REMOVED_COUNT++))
    fi
done

# Report results
if [[ $REMOVED_COUNT -eq 0 ]]; then
    echo "SKIP: No old skills found (already cleaned up)"
else
    echo "Removed $REMOVED_COUNT old skill(s)"
fi

exit 0
```

### Example 2: Settings Version Update (Configuration Evolution)

```bash
#!/bin/bash
# Migration: 003-settings-version-update
# Description: Update settings.json version and merge new hooks
# Created: 2026-01-26
# Idempotent: Yes

set -euo pipefail

MIGRATION_ID="003-settings-version-update"
PROJECT_DIR="${1:-.}"

# Paths
SETTINGS="$PROJECT_DIR/.claude-config/settings.json"
TEMPLATE="$HOME/ali-ai/templates/settings.json"

# Validate environment
if [[ ! -f "$SETTINGS" ]]; then
    echo "SKIP: No settings.json found"
    exit 0
fi

# Check current version
CURRENT_VERSION=$(jq -r '.version // "1.0"' "$SETTINGS")

if [[ "$CURRENT_VERSION" == "1.1" ]]; then
    echo "SKIP: Already at version 1.1"
    exit 0
fi

# Backup existing settings
cp "$SETTINGS" "$SETTINGS.backup"

# Merge template (adds new hooks)
if jq -s '.[0] * .[1]' "$SETTINGS" "$TEMPLATE" > "$SETTINGS.tmp"; then
    # Update version
    jq '.version = "1.1"' "$SETTINGS.tmp" > "$SETTINGS"
    rm "$SETTINGS.tmp"
    echo "Updated settings.json to version 1.1"
else
    # Restore backup on failure
    mv "$SETTINGS.backup" "$SETTINGS"
    echo "ERROR: Failed to merge settings"
    exit 1
fi

# Clean up backup
rm "$SETTINGS.backup"

exit 0
```

### Example 3: Directory Restructure (Atomic Migration)

```bash
#!/bin/bash
# Migration: 004-project-structure-migration
# Description: Restructure .claude/ directory layout
# Created: 2026-01-27
# Idempotent: Yes

set -euo pipefail

MIGRATION_ID="004-project-structure-migration"
PROJECT_DIR="${1:-.}"

# Check if already migrated
if [[ -f "$PROJECT_DIR/.claude/.restructure-complete" ]]; then
    echo "SKIP: Already restructured"
    exit 0
fi

# Create temp directory for atomic operation
TEMP_DIR=$(mktemp -d)
trap "rm -rf '$TEMP_DIR'" EXIT

# New structure
mkdir -p "$TEMP_DIR/new-structure/agents"
mkdir -p "$TEMP_DIR/new-structure/skills"
mkdir -p "$TEMP_DIR/new-structure/hooks"
mkdir -p "$TEMP_DIR/new-structure/logs"

# Migrate agents
if [[ -d "$PROJECT_DIR/.claude/agents" ]]; then
    cp -r "$PROJECT_DIR/.claude/agents/"* "$TEMP_DIR/new-structure/agents/" 2>/dev/null || true
fi

# Migrate skills
if [[ -d "$PROJECT_DIR/.claude/skills" ]]; then
    cp -r "$PROJECT_DIR/.claude/skills/"* "$TEMP_DIR/new-structure/skills/" 2>/dev/null || true
fi

# Migrate hooks
if [[ -d "$PROJECT_DIR/.claude/hooks" ]]; then
    cp -r "$PROJECT_DIR/.claude/hooks/"* "$TEMP_DIR/new-structure/hooks/" 2>/dev/null || true
fi

# Atomic swap
mv "$PROJECT_DIR/.claude" "$PROJECT_DIR/.claude.old"
mv "$TEMP_DIR/new-structure" "$PROJECT_DIR/.claude"

# Mark migration complete
touch "$PROJECT_DIR/.claude/.restructure-complete"

# Clean up old directory
rm -rf "$PROJECT_DIR/.claude.old"

echo "Restructure complete"
exit 0
```

---

## Testing Migrations

### Manual Testing Procedure

```bash
# 1. Test in ali-ai itself (safe test environment)
cd ~/ali-ai
./migrations/NNN-your-migration.sh ~/ali-ai

# 2. Test in a real project
cd ~/ali-ai/projects/test-project
~/ali-ai/migrations/NNN-your-migration.sh .

# 3. Test idempotency (run twice)
~/ali-ai/migrations/NNN-your-migration.sh .
~/ali-ai/migrations/NNN-your-migration.sh .
# Second run should SKIP or succeed without changes

# 4. Check registry
cat .claude/.migration-registry.json | jq '.migrations[] | select(.id=="NNN-your-migration")'

# 5. Check logs
tail -f .claude/logs/migrations.log
```

### Automated Testing

```bash
#!/bin/bash
# test-migration.sh
# Automated migration testing script

MIGRATION_PATH="$1"
MIGRATION_ID=$(basename "$MIGRATION_PATH" .sh)

# Create test project
TEST_DIR=$(mktemp -d)
trap "rm -rf '$TEST_DIR'" EXIT

# Setup test environment
mkdir -p "$TEST_DIR/.claude/skills"
touch "$TEST_DIR/.aliunde-project"

# Run migration (first time)
echo "Test 1: First execution"
if "$MIGRATION_PATH" "$TEST_DIR"; then
    echo "  PASS: Migration succeeded"
else
    echo "  FAIL: Migration failed (exit code: $?)"
    exit 1
fi

# Run migration (second time - idempotency test)
echo "Test 2: Idempotency check"
if "$MIGRATION_PATH" "$TEST_DIR"; then
    echo "  PASS: Migration idempotent"
else
    echo "  FAIL: Migration not idempotent (exit code: $?)"
    exit 1
fi

# Check registry
echo "Test 3: Registry validation"
REGISTRY="$TEST_DIR/.claude/.migration-registry.json"
if [[ -f "$REGISTRY" ]]; then
    COUNT=$(jq '[.migrations[] | select(.id=="'"$MIGRATION_ID"'")] | length' "$REGISTRY")
    if [[ "$COUNT" -eq 1 ]]; then
        echo "  PASS: Registry contains exactly 1 entry"
    else
        echo "  FAIL: Registry contains $COUNT entries (expected 1)"
        exit 1
    fi
else
    echo "  WARN: No registry file created"
fi

echo "All tests passed: $MIGRATION_ID"
```


---

## Checking Migration Status

<!-- source-doc: RUNBOOK.md — Migration Framework section, "Checking Migration Status" -->

To inspect the state of migrations for a project:

```bash
# View the migration registry (shows all run migrations with timestamps)
cat .claude/.migration-registry.json | jq '.migrations'

# View the migration execution log (last 50 entries)
tail -50 .claude/logs/migrations.log
```

**Registry format example:**

```json
{
  "migrations": [
    {"id": "001-remove-unprefixed-skills", "status": "complete", "timestamp": "2026-01-15T10:30:00Z"},
    {"id": "002-remove-unprefixed-hooks", "status": "complete", "timestamp": "2026-01-15T10:30:01Z"},
    {"id": "003-merge-settings-template", "status": "complete", "timestamp": "2026-01-15T10:30:02Z"}
  ]
}
```

**Troubleshooting a failed migration:**

1. Check `.claude/logs/migrations.log` for error details
2. Fix the underlying issue (missing dependency, permission error, etc.)
3. Re-run `claude-ali` — the migration will retry automatically on next launch
4. If the migration script itself is broken, fix it in `~/ali-ai/migrations/` and run `git pull`
5. For rollback options, see `~/ali-ai/migrations/README.md`

---

**Last Updated:** 2026-02-25 (W2 Restructure — added migration status check commands)
**Maintained By:** ALI AI Team
