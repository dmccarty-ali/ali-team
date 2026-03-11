# Session Management

## Session Persistence

```bash
# Sessions are saved automatically
claude -c                    # Continue latest
claude -r "session-name"     # Resume by name
/resume                      # Open session picker
/rename <name>               # Rename current session
```

## Course Correction

| Action | When | How |
|--------|------|-----|
| Stop mid-action | Claude doing wrong thing | Press Esc |
| Restore previous | Claude made bad change | Esc+Esc or /rewind |
| Clear context | Unrelated tasks piling up | /clear |

## Checkpoints

Checkpoints persist across sessions. /rewind restores to previous checkpoint.

## Persistent Sessions

```bash
# Continue where you left off
claude --continue

# Resume by name or ID
claude --resume "feature-work"
```

---

# Automation & Scaling

## Headless Mode

Run Claude non-interactively:

```bash
# Print mode (exit after)
claude -p "query"

# Start with prompt
claude "analyze this codebase"
```

## Output Formats

```bash
# JSON output
claude -p --output-format json "list all API endpoints"

# Streaming JSON (one object per line)
claude -p --output-format stream-json "review each file"
```

## Parallel Sessions

Run multiple Claude sessions simultaneously:

**Writer/Reviewer Pattern:**
```bash
# Terminal 1: Writer
claude --append-system-prompt "You are implementing features"

# Terminal 2: Reviewer
claude --append-system-prompt "You are reviewing code for security"
```

## Fan-Out Pattern

Loop over items with separate Claude invocations:

```bash
# Review each file independently
for file in src/api/*.py; do
  claude -p "Review $file for security issues" >> results.txt
done
```

## Scoped Permissions

Use --allowedTools to restrict capabilities:

```bash
# Read-only analysis
claude --allowedTools "Read,Grep,Glob" --permission-mode plan

# Only allow specific bash commands
claude --allowedTools "Bash(npm test:*),Read,Write"
```

## Safe Autonomous Mode

**DANGER:** Only use in sandboxed environments:

```bash
# Skip all permission prompts (DANGEROUS)
claude --dangerously-skip-permissions --allowedTools "Read,Write"
```

Use cases:
- CI/CD pipelines in containers
- Sandboxed test environments
- Automated code generation (with review)

**Never use in production environments or with access to sensitive data.**

---

# Settings Configuration

## File Locations

| File | Scope | Precedence |
|------|-------|------------|
| /Library/Application Support/ClaudeCode/ | Managed | Highest (cannot override) |
| ~/.claude/settings.json | User | Global preferences |
| .claude/settings.json | Project | Team shared |
| .claude/settings.local.json | Local | Personal, gitignored |

## Key Settings

```json
{
  "model": "sonnet",
  "alwaysThinkingEnabled": true,
  "statusLine": {
    "type": "command",
    "command": "~/.claude/statusline.sh"
  },
  "permissions": {
    "allow": [
      "Bash(npm run:*)",
      "Bash(git diff:*)",
      "Read"
    ],
    "ask": [
      "Bash(git push:*)"
    ],
    "deny": [
      "WebFetch",
      "Read(.env)",
      "Read(./secrets/**)"
    ],
    "additionalDirectories": ["../docs/"],
    "defaultMode": "default"
  },
  "cleanupPeriodDays": 30,
  "respectGitignore": true
}
```

## Permission Patterns

```
# Tool-specific syntax
Bash(npm run test:*)          # Prefix matching for bash
WebFetch(domain:github.com)   # Domain control
Edit(/src/**/*.ts)            # Gitignore-style paths
mcp__github__*                # All tools from MCP server
Task(Explore)                 # Specific subagent
Read(.env)                    # Deny sensitive files
```

## Permission Modes

| Mode | Behavior |
|------|----------|
| default | Standard prompts for approval |
| acceptEdits | Auto-accept file edits |
| dontAsk | Auto-deny unpre-approved |
| bypassPermissions | Skip all prompts (dangerous) |
| plan | Read-only analysis mode |

---

# Keyboard Shortcuts

## General

| Shortcut | Action |
|----------|--------|
| Ctrl+C | Cancel input/generation |
| Ctrl+D | Exit session |
| Ctrl+L | Clear screen |
| Ctrl+R | Reverse search history |
| Ctrl+V | Paste image from clipboard |
| Shift+Tab | Toggle permission modes |
| Alt+P | Switch model |
| Alt+T | Toggle extended thinking |
| Esc | Stop mid-action |
| Esc+Esc | Rewind conversation |

## Multiline Input

| Shortcut | Works In |
|----------|----------|
| \+Enter | All terminals |
| Option+Enter | macOS default |
| Shift+Enter | iTerm2, WezTerm, Ghostty |

---

# Slash Commands

## Built-In Commands

| Command | Purpose |
|---------|---------|
| /help | Get help |
| /clear | Clear conversation |
| /compact [hint] | Compact context with focus |
| /context | Visualize token usage |
| /cost | Token usage stats |
| /memory | Edit CLAUDE.md files |
| /model | Change model |
| /permissions | View/update permissions |
| /plan | Enter plan mode |
| /resume | Resume previous session |
| /status | Version/status info |
| /vim | Enable vim mode |

## Custom Commands

```markdown
# .claude/commands/review.md
---
allowed-tools: Read, Grep, Glob
description: Run comprehensive code review
argument-hint: [file-pattern]
model: sonnet
---

Review code in $1 (or current directory if not specified).
Focus on: security, performance, maintainability.
Output structured findings with severity levels.
```

## Argument Syntax

| Syntax | Meaning |
|--------|---------|
| $ARGUMENTS | All arguments as one string |
| $1, $2, $3 | Positional arguments |
| !\<command\> | Bash execution with output |
| @file | File reference |

---

# CLI Flags

## Common Options

```bash
claude                           # Interactive mode
claude "query"                   # Start with prompt
claude -p "query"                # Print mode (exit after)
claude -c                        # Continue latest session
claude -r "name"                 # Resume by name/ID

# Model selection
claude --model sonnet
claude --model opus
claude --model haiku

# Permissions
claude --permission-mode plan
claude --dangerously-skip-permissions

# Working directories
claude --add-dir ../docs ../lib

# Output
claude -p --output-format json "query"
claude -p --output-format stream-json "query"

# System prompt
claude --append-system-prompt "Always use TypeScript"
claude --system-prompt-file ./prompt.txt

# Debug
claude --debug "api,mcp"
claude --verbose
```

---

# IDE Integrations

## VS Code

- **Install**: Command Palette > "Claude Code" > Install
- **Launch**: Click Spark icon or Cmd+Shift+Esc
- **Features**: Inline diffs, @-mentions with line numbers, conversation tabs
- **Focus Input**: Cmd+Esc / Ctrl+Esc
- **Insert @-Mention**: Alt+K

## JetBrains

- **Install**: Settings > Plugins > Marketplace > "Claude Code"
- **Launch**: Cmd+Esc / Ctrl+Esc
- **File Reference**: Cmd+Option+K / Alt+Ctrl+K
- **Configuration**: Settings > Tools > Claude Code

---

# Quick Reference

## File Locations Cheatsheet

```
Configuration:
~/.claude/settings.json          # User settings
.claude/settings.json            # Project settings (git)
.claude/settings.local.json      # Local settings (gitignored)

Instructions:
~/.claude/CLAUDE.md              # User memory
./CLAUDE.md                      # Project memory (git)
./CLAUDE.local.md                # Local memory (gitignored)
./.claude/rules/*.md             # Path-specific rules

Customization:
~/.claude/skills/*/SKILL.md      # User skills
.claude/skills/*/SKILL.md        # Project skills
~/.claude/agents/*.md            # User agents
.claude/agents/*.md              # Project agents
~/.claude/commands/*.md          # User commands
.claude/commands/*.md            # Project commands

MCP:
~/.claude.json                   # User MCP servers
.mcp.json                        # Project MCP servers
```

## Common Commands

```bash
# Session management
claude -c                        # Continue latest
claude -r "name"                 # Resume session
/clear                           # Clear conversation
/compact                         # Reduce context

# Configuration
/memory                          # Edit CLAUDE.md
/permissions                     # Manage permissions
/model sonnet                    # Switch model

# MCP
claude mcp list                  # List servers
claude mcp add <name> <url>      # Add server

# Debug
/context                         # View token usage
/cost                            # View costs
/status                          # Version info
```

---

# Your Codebase

## Aliunde Skills Location

| Purpose | Location |
|---------|----------|
| Shared skills repo | ~/.claude/skills/ (symlink to ali-ai/skills/) |
| Shared agents | ~/.claude/agents/ (symlink to skills/agents/) |
| Skill template | ~/.claude/skills/SKILL_TEMPLATE.md |
| Agent template | ~/.claude/skills/PROJECT_EXPERT_TEMPLATE.md |
| Expert panel | ~/.claude/skills/agents/expert-panel.md |

## Project-Specific Setup

| Project | CLAUDE.md Location |
|---------|-------------------|
| ingestion-engine | projects/ingestion-engine/CLAUDE.md |
| tax-practice | projects/tax-practice/CLAUDE.md |

---

# Batch Mode (claude -p) — Non-Interactive Execution

<!-- source-doc: RUNBOOK.md — "Batch Claude CLI" section -->

The proven pattern for running Claude CLI non-interactively (batch mode):

```bash
claude -p \
  --max-turns N \
  --output-format text \
  --system-prompt-file path/to/agent.txt \
  < input.md \
  > output.md \
  2> stderr.log
```

**Key flags:**

| Flag | Purpose |
|------|---------|
| `--max-turns N` | Limit API round-trips (use 1-2 for tests, higher for real work) |
| `--output-format text` | Required for stdout capture — streaming JSON does not flush to non-TTY |
| `--system-prompt-file` | Replaces CLAUDE.md entirely, bypasses CoS/role loading. Required for batch/test invocations. |
| `< input.md` | stdin redirect for prompt |
| `> output.md` | stdout redirect for output |
| `2> stderr.log` | stderr redirect for errors/debug |

## Critical Gotchas (Must Not Ignore)

These are hard behavioral constraints — not preference notes. Each one causes silent failures if ignored.

**Gotcha 1: --output-format text is required**

Without `--output-format text`, streaming output does not flush into non-TTY context. The command will appear to hang or produce empty output.

```bash
# WRONG — streaming JSON does not flush to non-TTY stdout
claude -p "query" > output.md

# CORRECT
claude -p --output-format text "query" > output.md
```

**Gotcha 2: File redirect required — $() capture does not work**

`$()` subshell capture returns empty because claude writes to TTY directly, not stdout.

```bash
# WRONG — capture returns empty
result=$(claude -p --output-format text "query")

# CORRECT — use file redirect
claude -p --output-format text "query" > output.md
result=$(cat output.md)
```

**Gotcha 3: Do not run claude -p inside an active Claude Code session**

Nested sessions block each other. Running `claude -p` from inside an active Claude Code session (e.g., from a hook or agent tool call) will deadlock or fail.

Run batch invocations from a separate terminal or a separate process outside the active Claude Code session.

For the full analysis and claude-agent-sdk solution, see `references/nested-sessions.md`.

**Gotcha 4: --system-prompt-file bypasses CLAUDE.md (required for batch)**

Without `--system-prompt-file`, CLAUDE.md loads the CoS role which consumes turns asking protocol questions instead of answering your prompt. Always supply a system prompt file for batch invocations.

```bash
# WRONG — loads CoS role, wastes turns
claude -p --output-format text --max-turns 3 < prompt.md > output.md

# CORRECT — bypasses CoS, uses targeted system prompt
claude -p --output-format text --max-turns 3 \
  --system-prompt-file /path/to/agent-prompt.txt \
  < prompt.md > output.md
```

## Fan-Out Pattern (Loop + Batch)

For processing multiple items with separate Claude invocations:

```bash
# Review each file independently
for file in src/api/*.py; do
  claude -p --output-format text --max-turns 2 \
    --system-prompt-file prompts/reviewer.txt \
    "Review $file for security issues" >> results.txt
done
```


