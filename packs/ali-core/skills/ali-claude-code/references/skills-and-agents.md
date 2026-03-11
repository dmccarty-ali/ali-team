# Skills System

## SKILL.md Structure

```yaml
---
name: skill-name                    # lowercase, hyphens, max 64 chars
description: |                      # max 1024 chars, include trigger keywords
  What this skill covers. Use when:

  PLANNING: [planning activities]
  IMPLEMENTATION: [implementation activities]
  GUIDANCE: [guidance activities]
  REVIEW: [review activities]
allowed-tools: Read, Grep, Glob     # optional: restrict available tools
model: sonnet                       # optional: specific model
---

# Skill Content

## When I Speak Up
[Trigger conditions]

## Key Principles
[Bullet points of decisions]

## Patterns We Use
[Code examples from YOUR projects]

## Anti-Patterns to Catch
[What to flag and why]

## Quick Reference
[Commands, snippets, lookups]

## Your Codebase
[Specific file paths]
```

## Four-Mode Trigger Pattern

Every skill description MUST cover all four modes:

| Mode | When | Example Keywords |
|------|------|------------------|
| PLANNING | Designing solutions | "designing", "architecting", "considering" |
| IMPLEMENTATION | Writing code | "writing", "implementing", "building" |
| GUIDANCE | Asking for advice | "best practices", "how do others", "recommended" |
| REVIEW | Evaluating work | "review", "check", "validate", "audit" |

Without all four modes, skills only trigger ~25% of the time they should.

## Negative Triggers

Prevent over-triggering by explicitly stating what the skill should NOT handle.

**Add to YAML description:**
```yaml
description: |
  Advanced data analysis for CSV files. Use for statistical modeling,
  regression, clustering. Do NOT use for simple data exploration
  (use data-viz skill instead).
```

**Add to SKILL.md body -- "When NOT to Use This Skill" section:**

Place immediately after "When I Speak Up":

```markdown
## When NOT to Use This Skill

This skill does NOT activate for:
- {Thing this skill doesn't handle} (use {other-skill} instead)
- {Another exclusion}
- {Another exclusion}
```

**Guidelines:**
- Be specific about alternatives: name the skill or approach to use instead
- Include 3-6 negative triggers per skill
- Cover the most common over-trigger scenarios
- Reference competing skills by name when applicable

## Tool Restrictions (allowed-tools)

Skills can restrict which tools are available using the `allowed-tools` frontmatter field.

### When to Use

- Skills that should only read, not modify (documentation reviewers)
- Skills that need specific tool access (test runners needing only certain bash commands)
- Skills for security-sensitive operations (audit, compliance)

### Syntax

```yaml
allowed-tools: Read, Grep, Glob                    # Simple tool list
allowed-tools: "Bash(npm:*) Bash(git:*) Read"      # Pattern matching for bash
allowed-tools: Read, Write, mcp__github__*          # MCP tool wildcards
```

### Examples

**Read-only documentation skill:**
```yaml
---
name: doc-reviewer
description: Reviews documentation for accuracy and completeness
allowed-tools: Read, Grep, Glob
---
```

**Skill with specific bash access:**
```yaml
---
name: node-tester
description: Runs Node.js test suites and analyzes results
allowed-tools: "Bash(npm test:*) Bash(npm run test:*) Read"
---
```

**Skill requiring MCP access:**
```yaml
---
name: github-reviewer
description: Reviews GitHub pull requests for code quality
allowed-tools: Read, Grep, mcp__github__*
---
```

### Notes

- Tool restrictions in skills are additive with project-level permissions
- The most restrictive rule wins when multiple restrictions apply
- Omitting allowed-tools grants access to all tools (default)
- Use allowlists (allowed-tools) in skills, denylists (disallowedTools) in agents

## Size Guidelines

### SKILL.md Body (Second Level -- auto-loaded when relevant)

Target: under 5,000 words. Keep focused on core instructions.

### references/*.md (Third Level -- loaded on demand)

No hard limit, but keep individual reference files focused on one topic.

### Total Skill Size (All Files)

| Size | Words (total) | Use Case |
|------|---------------|----------|
| Lean | 1,500-3,000 | Project patterns, quick reference |
| Standard | 3,000-7,000 | Technology skill with examples |
| Comprehensive | 7,000-15,000 | Complex domain (security, AWS) |

**If SKILL.md body exceeds 5,000 words, move content to references/.**

## Progressive Disclosure

```
my-skill/
├── SKILL.md          # Essential info (<500 lines)
├── reference.md      # Detailed docs (loaded on demand)
├── examples.md       # Usage examples
└── scripts/
    └── helper.py     # Utility scripts (executed, not read)
```

## What to Include vs Exclude

**Include:**
- YOUR patterns and decisions (Claude doesn't know these)
- YOUR codebase specifics (file paths, conventions)
- Anti-patterns specific to your context
- Quick reference / cheat sheets

**Exclude:**
- Basic language syntax
- General explanations of well-known patterns
- Verbose tutorials
- Full API documentation (link instead)

## Skill Workflow Patterns

Common approaches for structuring skills, based on Anthropic's guide and early adopter patterns.

### Pattern 1: Sequential Workflow Orchestration

**Use when:** Users need multi-step processes in a specific order.

**Structure:** Numbered steps with explicit dependencies, validation at each stage, rollback instructions for failures.

**Example:** Customer onboarding (create account > setup payment > create subscription > send welcome email).

**Key techniques:**
- Explicit step ordering
- Dependencies between steps
- Validation at each stage
- Rollback instructions for failures

### Pattern 2: Multi-MCP Coordination

**Use when:** Workflows span multiple services.

**Structure:** Phased approach with clear data passing between MCPs. Each phase uses one MCP.

**Example:** Design-to-development handoff (Figma export > Drive storage > Linear tasks > Slack notification).

**Key techniques:**
- Clear phase separation
- Data passing between MCPs
- Validation before moving to next phase
- Centralized error handling

### Pattern 3: Iterative Refinement

**Use when:** Output quality improves with iteration.

**Structure:** Initial draft > quality check > refinement loop > finalization.

**Example:** Report generation with validation script and re-generation loop.

**Key techniques:**
- Explicit quality criteria
- Iterative improvement
- Validation scripts
- Know when to stop iterating

### Pattern 4: Context-Aware Tool Selection

**Use when:** Same outcome can be achieved with different tools depending on context.

**Structure:** Decision tree based on input characteristics, then execute with appropriate tool.

**Example:** Smart file storage (file type/size determines cloud storage vs Notion vs GitHub vs local).

**Key techniques:**
- Clear decision criteria
- Fallback options
- Transparency about choices

### Pattern 5: Domain-Specific Intelligence

**Use when:** Skill adds specialized knowledge beyond tool access.

**Structure:** Domain checks before action, conditional processing, comprehensive audit trail.

**Example:** Payment processing with compliance (sanctions check > jurisdiction verification > risk assessment > process or flag).

**Key techniques:**
- Domain expertise embedded in logic
- Compliance before action
- Comprehensive documentation
- Clear governance

---

# Agent System

## Agent Definition

```yaml
---
name: code-reviewer
description: |
  Code review specialist. Use immediately after code changes
  to validate quality, patterns, and security.
model: sonnet                        # sonnet|opus|haiku|inherit
tools: Read, Grep, Glob              # allowlist (omit for all)
disallowedTools: Write, Edit         # denylist
permissionMode: default              # default|acceptEdits|dontAsk|plan
skills:                              # load specific skills
  - secure-coding
  - design-patterns
---

You are a senior code reviewer...
[System prompt with review checklist and output format]
```

## Built-In Subagents

| Agent | Purpose | Model |
|-------|---------|-------|
| Explore | Fast codebase exploration | Haiku |
| Plan | Plan mode research | Inherited |
| general-purpose | Complex multi-step tasks | All tools |
| Bash | Terminal commands | - |

## Agent Scopes

| Scope | Location | Use Case |
|-------|----------|----------|
| Project | .claude/agents/ | Team shared (git) |
| User | ~/.claude/agents/ | Personal, all projects |
| Plugin | plugin-name/agents/ | Bundled with plugin |
| CLI | --agents JSON | Current session only |

## When to Use Agents vs Skills

| Use Case | Skill | Agent |
|----------|-------|-------|
| Auto-triggered knowledge | Yes | No |
| Formal structured review | No | Yes |
| Isolated context needed | No | Yes |
| Tool restrictions | No | Yes |
| High-volume output | No | Yes |
