---
name: ali-claude-code
description: |
  Claude Code CLI configuration, customization, and best practices. Use when:

  PLANNING: Designing Claude Code setup, planning skills/agents architecture,
  considering permission strategies, evaluating MCP server integrations

  IMPLEMENTATION: Creating CLAUDE.md files, writing skills, defining agents,
  configuring hooks, setting up MCP servers, writing slash commands,
  creating or editing agent files (agents/*.md), setting agent frontmatter
  fields (tools:, skills:, name:, model:, description:), referencing
  AGENT_CREATION_GUIDE.md, specifying tools format or skills format in YAML

  GUIDANCE: Asking about Claude Code features, best practices for configuration,
  how to structure skills, what hooks to use, permission patterns

  REVIEW: Auditing Claude Code setup, checking skill quality, reviewing
  agent definitions, validating permission configurations
---

# Claude Code Configuration

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing Claude Code setup for projects or teams
- Planning skills and agents architecture
- Evaluating MCP server integrations
- Architecting permission strategies

**Implementation:**
- Creating or editing CLAUDE.md files
- Writing skills (SKILL.md files)
- Defining custom agents
- Configuring hooks for automation
- Setting up MCP servers
- Writing custom slash commands
- Writing or creating agent files (agents/*.md)
- Setting agent frontmatter fields (tools:, skills:, name:, model:, description:)
- Referencing AGENT_CREATION_GUIDE.md
- Specifying tools format or skills format in YAML frontmatter

**Guidance/Best Practices:**
- Asking about Claude Code features
- Asking how to structure skills effectively
- Asking about permission patterns
- Asking about context management

**Review/Validation:**
- Auditing Claude Code configuration
- Reviewing skill quality and trigger patterns
- Validating agent definitions
- Checking permission security

## When NOT to Use This Skill

This skill does NOT activate for:
- Writing MCP servers (use ali-mcp skill if available)
- Project-specific development work (use project expert agents)
- General CLI usage questions (basic terminal/shell commands)
- IDE troubleshooting (VS Code, JetBrains, Cursor specific issues)
- Writing or debugging application code (use language-specific skills)
- Git operations beyond Claude Code context (use git documentation)

---

## Key Principles

- **CLAUDE.md is your primary interface**: Project-specific instructions go in CLAUDE.md, not repeated in every conversation
- **Skills trigger automatically**: Write good descriptions with trigger keywords - Claude selects skills based on context
- **Agents are for formal reviews**: Use agents when you need isolated context, specific tool restrictions, or structured output
- **Permissions follow least-privilege**: Only allow what's needed, deny dangerous operations by default
- **Context is precious**: Manage context window carefully - skills should be concise, use progressive disclosure
- **Hooks enable automation**: Use hooks for validation, formatting, and compliance - not core logic
- **Flat structure required**: Skills must be one level deep (skill-name/SKILL.md), no nesting
- **@import paths must be absolute**: Claude Code does NOT expand tilde in @import paths. Use `/Users/username/path/file.md`, not `@~/path/file.md` (silently fails)

---

## Creating Agents in ali-ai

Quick-reference for ali-ai-specific agent creation rules. Full guide: `docs/AGENT_CREATION_GUIDE.md`.

### CRITICAL: tools and skills field format

Tools and skills MUST use comma-separated inline format. Multi-line YAML list format does NOT work — Claude Code does not parse it.

```yaml
# CORRECT
tools: Read, Write(.tmp/**), Grep, Glob
skills: ali-agent-operations, ali-bash

# WRONG — will not work
tools:
  - Read
  - Grep
```

### Naming rules

- Filename MUST use ali- prefix: `ali-{name}.md`
- `name:` field in frontmatter must match filename without .md
- Skill references in `skills:` field use ali- prefix (e.g., `ali-agent-operations`)
- Exception: `plan-builder` is the only org skill without the ali- prefix

### Tools by agent type

| Type | tools field | Notes |
|------|-------------|-------|
| -expert | `Read, Write(.tmp/**), Grep, Glob` | No Bash, no Edit |
| -admin | `Bash, Read, Grep, Glob` | Some add Edit |
| -developer | `Read, Grep, Glob, Edit, Write, Bash` | Full write access |

### expert-metadata (expert agents only)

```yaml
# Expert metadata parsed by: scripts/build_agent_manifest.py
expert-metadata:
  domains: [security, compliance]
  keywords: [auth, password, token]
  file-patterns: ["src/auth/**"]
  anti-keywords: [css, frontend]
```

The parser comment is required. After creating or modifying an expert agent, rebuild the manifest:

```bash
python ~/.claude/bin/build_agent_manifest.py
```

### Frontmatter example (expert agent)

```yaml
---
name: ali-security-expert
description: Security and compliance review specialist
model: sonnet
skills: ali-agent-operations, ali-secure-coding
tools: Read, Write(.tmp/**), Grep, Glob

# Expert metadata parsed by: scripts/build_agent_manifest.py
expert-metadata:
  domains: [security, compliance]
  keywords: [auth, password, encryption, token]
  file-patterns: ["src/auth/**"]
  anti-keywords: [css, frontend, ui]
---
```

---

## Context Management (PRIMARY CONSTRAINT)

**Context window fills fast. Performance degrades as it fills.** This is THE primary constraint when working with Claude Code.

### Why Context Management Matters

- Context window has hard limits (200k tokens for Sonnet 4.5)
- As context fills, response quality degrades
- Token costs increase with context size
- Large CLAUDE.md files waste context every session
- Verbose skills consume valuable tokens

### Context Management Strategies

| Strategy | When to Use | How |
|----------|-------------|-----|
| /clear | Between unrelated tasks | Reset conversation to start fresh |
| /compact | Context approaching limits | Summarize older messages with optional focus hint |
| Subagents | Investigation tasks | Spawn separate context for exploration (Explore agent) |
| Progressive disclosure | Large skills | 3-level system: YAML frontmatter (always loaded), SKILL.md body (auto-loaded when relevant), references/*.md (loaded on demand) |
| Lean CLAUDE.md | Always | Keep <200 lines, prune quarterly |

### /clear - Your Most Important Tool

**Use /clear between unrelated tasks.** Don't let context accumulate from yesterday's bug fix into today's feature work.

**Symptoms you need /clear:**
- Claude starts referencing old, irrelevant code
- Responses become verbose or unfocused
- Performance feels slow
- You're working on something unrelated to previous messages

**Example workflow:**
```
Session 1: Fix authentication bug → /clear
Session 2: Add new API endpoint → /clear
Session 3: Update documentation → /clear
```

### /compact - Reduce Without Starting Over

When you want to keep the current task but reduce context:

```bash
/compact                           # Automatic summarization
/compact "focus on API changes"    # Directed compaction
```

Use when:
- Long back-and-forth debugging session
- Large file reads early in conversation
- Context warning appears
- Want to preserve current task context

### Subagents for Separate Context

Spawn subagents for investigation work that would pollute your main context:

```bash
# Main session stays clean
"Spawn Explore agent to find all database connection patterns"

# Subagent explores in separate context
# Returns findings without cluttering main conversation
```

### Frame Context as THE Constraint

When planning work:
1. Will this task consume lots of context? (multi-file refactor)
2. Should I use /plan mode first? (read-only exploration)
3. Do I need to /clear before starting?
4. Should I delegate to subagent?

---

## Verification

**Verification is the single highest-leverage thing you can do.** Always provide concrete ways to verify changes work.

### Why Verification Matters

Without verification methods:
- Claude can't self-check work
- Bugs slip through to commit
- You waste time manually testing
- Confidence in changes is low

### Verification Methods

| Method | Use Case | Example |
|--------|----------|---------|
| Tests | Backend logic | pytest tests/test_auth.py -v |
| Screenshots | UI changes | Take before/after screenshots |
| Expected outputs | CLI tools | Run command, verify output format |
| Behavioral specs | Complex flows | "User should see error message when..." |

### Provide Verification Upfront

**BAD:**
> "Add a logout button to the header"

**GOOD:**
> "Add a logout button to the header. Verify by:
> 1. Take screenshot showing button in header
> 2. Click button, verify redirects to /login
> 3. Verify session cookie is cleared (check browser dev tools)"

### UI Verification with Screenshots

For any UI change, verification MUST include screenshots:

```bash
# Before starting
"Current state: [paste screenshot]"

# After changes
"Verify: Take screenshot showing [specific change]"
"Verify: Screenshot should show button in top-right with blue background"
```

### Test Verification

For backend changes, provide test commands:

```bash
# Unit tests
pytest tests/unit/test_auth.py::test_login -v

# Integration tests
pytest tests/integration/test_workflow.py -v

# Specific assertion
"Test should verify error message contains 'Invalid credentials'"
```

### Expected Output Verification

For CLI tools and scripts:

```bash
# Run command
python scripts/migrate.py --dry-run

# Expected output
"Output should show:
- 5 tables to migrate
- 0 errors
- 'Dry run complete' message"
```

### Self-Verification Checklist

Before marking work complete, Claude should verify:
- [ ] Tests pass (if applicable)
- [ ] Screenshot matches expected (if UI)
- [ ] Output format correct (if CLI)
- [ ] Error cases handled (behavioral spec)
- [ ] No regressions (related features still work)

---

## Common Failure Patterns

| Anti-Pattern | Why It's Bad | Fix |
|--------------|--------------|-----|
| Kitchen sink session | Unrelated tasks accumulate, polluting context | /clear between unrelated tasks |
| Correcting over and over | Polluted context prevents learning | /clear and restart with better prompt |
| Nested claude CLI subprocess | CLAUDECODE env var conflict + TMPDIR socket contention + auth token leak causes exit code 1 | Use `claude-agent-sdk` Python package instead — handles all three conflict layers natively via `env={"CLAUDECODE": ""}` |
| Over-specified CLAUDE.md | Too long (>200 lines), Claude ignores | Prune to essentials, link to docs |
| Trust without verify | No verification, bugs slip through | Always provide test commands, screenshots |
| Infinite exploration | Unscoped investigation wastes time | Scope specifically: "Check these 3 files" |
| Huge CLAUDE.md | Wastes context every session | Keep concise, use imports |
| No skill description modes | Only triggers 25% of time | Include PLANNING/IMPLEMENTATION/GUIDANCE/REVIEW |
| Skills over 6000 words | Slow to load, context waste | Split into multiple skills |
| Nested skill directories | Claude won't find them | Use flat structure |
| Agents for everything | Isolated context, overhead | Use skills for auto-triggered knowledge |
| Allow all bash commands | Security risk | Whitelist specific commands |
| Hardcoded absolute paths | Breaks on other machines | Use relative paths, environment variables |
| No permission defaults | Repetitive prompts | Configure settings.local.json |
| @import with tilde (~) | Tilde not expanded — file silently not loaded | Use absolute path: @/Users/username/path/file.md |
| tools: as YAML list | Claude Code does not parse multi-line list format | Use inline comma-separated: `tools: Read, Grep, Glob` |

For systematic diagnosis and step-by-step fixes, see `references/troubleshooting.md`.

---

## Detailed Reference

For detailed guidance on specific topics, consult the reference files below. These are loaded on demand to keep context lean.

For CLAUDE.md structure, hierarchy, and pruning guidance, consult `references/claude-md-guide.md` covering:
- Hierarchy and precedence (Enterprise > Project > User > Local)
- What to include vs exclude
- Pruning recommendations and before/after examples
- Import syntax and path-specific rules

For writing skills or defining agents, consult `references/skills-and-agents.md` covering:
- SKILL.md YAML structure and four-mode trigger pattern
- Negative triggers and allowed-tools documentation
- Size guidelines and progressive disclosure
- Five workflow patterns (sequential, multi-MCP, iterative, context-aware, domain-specific)
- Agent definition, scopes, and when to use agents vs skills

For workflow phases and prompting techniques, consult `references/workflow-and-prompting.md` covering:
- Four phases: Explore > Plan > Implement > Commit
- Prompting best practices and anti-patterns
- Rich content patterns (@file, images, URLs, piping)

For CLI flags, shortcuts, settings, and IDE integration, consult `references/cli-reference.md` covering:
- Session management and automation/scaling
- Settings configuration and permission patterns
- Keyboard shortcuts, slash commands, CLI flags
- VS Code and JetBrains integration
- File locations cheatsheet

For hooks, MCP servers, or migration framework, consult `references/hooks-mcp-migrations.md` covering:
- Hook events, configuration, and exit codes
- MCP server commands and scopes
- Migration framework (when to create, how they work)

For skill testing methodology including trigger tests, functional tests, and performance comparison, consult `references/skill-testing.md` covering:
- Triggering test case structure (positive and negative)
- Functional test methodology and checklists
- Performance comparison baselines and metrics
- Testing tools and iterative improvement

For troubleshooting skill issues including upload errors, trigger problems, and MCP failures, consult `references/troubleshooting.md` covering:
- Skill upload errors and frontmatter validation
- Trigger debugging (under-triggering and over-triggering)
- Instruction compliance issues
- MCP connection diagnosis
- Large context performance problems

For skill distribution, API usage, and positioning guidance, consult `references/skill-distribution.md` covering:
- Distribution model (individual and organization-level)
- Skills API endpoint and Messages API integration
- Positioning skills for different audiences
- Distribution checklist and best practices

For invoking Claude Code as a subprocess from within an active Claude Code session (nested session pattern, claude-agent-sdk, bypassPermissions), consult `references/nested-sessions.md`. Covers:
- Three conflict layers that cause exit code 1 (CLAUDECODE, TMPDIR socket, auth token)
- claude-agent-sdk solution with ClaudeAgentOptions parameters
- LangGraph asyncio.run() bridge for sync nodes
- Unit test mocking pattern for claude_agent_sdk.query

---

## References

- [Claude Code Documentation](https://docs.anthropic.com/claude-code)
- [CLI Reference](https://docs.anthropic.com/claude-code/cli-reference)
- [Settings](https://docs.anthropic.com/claude-code/settings)
- [Skills](https://docs.anthropic.com/claude-code/skills)
- [Subagents](https://docs.anthropic.com/claude-code/sub-agents)
- [MCP Servers](https://docs.anthropic.com/claude-code/mcp)
- [Hooks](https://docs.anthropic.com/claude-code/hooks)

---

**Document Version:** 3.4
**Last Updated:** 2026-03-16
**Maintained By:** ALI AI Team
