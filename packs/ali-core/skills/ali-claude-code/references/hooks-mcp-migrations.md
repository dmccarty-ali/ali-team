# Migration Framework

ali-ai includes an automated migration framework for zero-touch organizational updates.

## When to Create Migrations

Create migrations when ali-ai changes require transformation of existing project resources:
- Resource renames (skills, hooks, agents, directories)
- Deprecation cleanup (removing obsolete files)
- Configuration schema evolution (updating file formats)
- New required files (adding mandatory resources)
- Permission corrections (fixing file modes)

**Don't create migrations for:**
- New features that sync automatically via rsync
- Documentation updates with no file impact
- Optional enhancements

## How Migrations Work

1. Migrations stored in ~/ali-ai/migrations/
2. Launcher runs pending migrations automatically on claude-ali launch
3. Registry tracks completed migrations per-project (.claude/.migration-registry.json)
4. Idempotent execution (safe to run multiple times)
5. Logged to .claude/logs/migrations.log
6. Stop-on-failure behavior (subsequent migrations wait until fix)

## Developer Guide

See ~/ali-ai/migrations/README.md for:
- Migration template (MIGRATION_TEMPLATE.sh)
- Naming conventions (NNN-descriptive-slug.sh)
- Idempotency requirements
- Testing procedures
- Example migrations

## Example: Adding New Hook to All Projects

When Plan 004 (SOP Enforcement) adds a new hook:
1. Add hook to ~/ali-ai/templates/settings.json
2. Bump version: "1.0" → "1.1"
3. Migration 003 automatically runs on next sync
4. New hook merged into all project settings

## Example: Renaming a Skill

1. Rename skill in ~/ali-ai/skills/ (e.g., old-name/ → ali-new-name/)
2. Create migration to remove old skill from projects:
   ```bash
   # migrations/005-remove-old-skill.sh
   if [[ -d "$PROJECT_DIR/.claude/skills/old-name" ]]; then
       rm -rf "$PROJECT_DIR/.claude/skills/old-name"
       echo "Removed: old-name"
   fi
   ```
3. rsync adds new skill, migration removes old

---

# Hooks System

## Hook Events

| Event | When | Use Case |
|-------|------|----------|
| PreToolUse | Before tool execution | Validation, blocking |
| PostToolUse | After tool completion | Formatting, logging |
| UserPromptSubmit | Before processing prompt | Pre-processing |
| Stop | When Claude finishes | Cleanup, notifications |
| SessionStart | Session begins | Setup |

## Hook Configuration

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "./scripts/validate-command.sh $TOOL_INPUT"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "ruff format $FILE_PATH"
          }
        ]
      }
    ]
  }
}
```

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Allow execution |
| 2 | Block execution (show error to Claude) |
| 1 | Error in hook itself |

## JSON Output for Advanced Control

```json
{
  "allow": true,
  "deny": false,
  "explanation": "Why allowing/blocking",
  "stop": false,
  "appendMessage": "Additional info for Claude"
}
```

---

# MCP Servers in Claude Code

**For comprehensive MCP development patterns, see the ali-mcp skill.**

This section covers Claude Code-specific MCP configuration commands.

## Claude Code MCP Commands

```bash
# Add HTTP server
claude mcp add --transport http github https://mcp.github.com

# Add local process
claude mcp add --transport stdio db -- npx -y database-mcp-server

# List configured servers
claude mcp list

# Remove server
claude mcp remove github
```

## MCP Scopes

| Scope | Location | Purpose |
|-------|----------|---------|
| local | ~/.claude.json | Current project, private |
| project | .mcp.json | Team shared (git) |
| user | ~/.claude.json | All projects, private |

## Related

- **ali-mcp skill**: MCP protocol, server development, tool schemas, MCP Apps (UI extensions)
