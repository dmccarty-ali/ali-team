---
name: ali-local-agent-builder
description: |
  Guide for creating project-specific agents in .claude-local/agents/. Use when:

  PLANNING: Deciding whether an agent should be project-local or org-wide,
  "create a project agent", "local agent for this project", "project-specific agent",
  planning what a project-specific agent should do, scoping tools and skills for a
  local agent, evaluating whether a project agent should shadow an org agent

  IMPLEMENTATION: "add an agent to this project", writing a new agent file under
  .claude-local/agents/, setting required frontmatter fields, authoring the identity
  and workflow sections, verifying the agent is platform-visible after creation

  GUIDANCE: Asking how to add an agent to a specific project, asking where project
  agents live vs. org agents, asking how project agents get loaded at runtime, asking
  whether a release is needed for project agents, asking about .claude-local/ vs.
  .claude/ and why the distinction matters

  REVIEW: Reviewing a .claude-local/ agent for correct frontmatter, naming, required
  sections, Step 0 Handshake presence, and platform visibility
---

# Local Agent Builder (Project-Specific Agents)

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Deciding whether a new agent belongs to one project or all projects
- Planning what project-specific capabilities to encode in an agent
- Scoping which tools an agent should have access to for this project
- Evaluating whether the agent should shadow or extend an existing org agent
- Designing a project-domain expert that understands this project's architecture

**Implementation:**
- Creating a new agent file under `.claude-local/agents/`
- Writing frontmatter, identity section, and workflow steps
- Setting up `.claude-local/` directory structure for the first time
- Adding project-specific skills to the agent's skills list
- Verifying the agent is visible to Claude Code after creation

**Guidance/Best Practices:**
- Asking how to add an agent that only applies to one project
- Asking where project-local agents live vs. org agents
- Asking how project agents are loaded at runtime and whether a release is needed
- Asking about `.claude-local/agents/` vs. `.claude/agents/` and which is correct
- Asking whether `ali-` prefix is required for project agents

**Review/Validation:**
- Reviewing a `.claude-local/` agent for correct frontmatter and naming
- Checking that Step 0: Handshake is the first workflow output
- Confirming the agent file is in `.claude-local/agents/`, NOT `.claude/agents/`
- Validating the agent is git-tracked (not in `.gitignore`)
- Confirming the agent is platform-visible after a `claude-ali` relaunch

## When NOT to Use This Skill

This skill does NOT activate for:
- Creating org-wide agents available to all projects (use ali-org-agent-builder instead)
- Creating org-wide skills (use ali-org-skill-builder instead)
- Creating project-local skills (use ali-local-skill-builder instead)
- Writing CLAUDE.md content (use ali-claude-code skill)
- Configuring hooks or settings (use ali-claude-code skill)

---

## Key Principles

- **Project agents live in `.claude-local/agents/`** — NOT `.claude/agents/`. This is the single most important rule. `.claude/` is auto-generated and gitignored; anything placed there is lost at next sync. `.claude-local/` is git-tracked.
- **CRITICAL: `.claude-local/agents/` is NOT `.claude/agents/`** — the platform scans `.claude/agents/` at runtime. The launcher overlays `.claude-local/agents/` on top of `.claude/agents/` at sync time. If you place an agent directly in `.claude/agents/`, it will be overwritten on next `claude-ali` launch.
- **No `ali-` prefix required** — project agent names are kebab-case without the org prefix (e.g., `tax-domain-expert`, `ingestion-reviewer`). The `ali-` prefix is reserved for org agents in `~/ali-ai/agents/`.
- **Filename must match `name:` field** — `my-project-expert.md` requires `name: my-project-expert`. Mismatch causes spawn errors.
- **No release needed** — project agents are available on the next `claude-ali` launch after commit. No tarball needed.
- **Git-track the agent** — `.claude-local/` is committed to the project repo. It is NOT in `.gitignore`.
- **No manifest rebuild** — only org-wide expert agents need `build_agent_manifest.py`. Project-local agents are overlaid directly from `.claude-local/agents/` and do not have AGENT_MANIFEST.json entries.
- **Step 0: Handshake required** — same as org agents. Every project agent must acknowledge task receipt before doing any work.
- **Always include `ali-agent-operations`** — always include it in the skills list. Add any project-specific skills the agent should use.

---

## CRITICAL: .claude-local/ vs .claude/ — The Key Distinction

This is the most common mistake when creating project agents:

```
{project}/
├── .claude/              # AUTO-GENERATED — gitignored — DO NOT EDIT
│   └── agents/           # Synced from org plugin cache at claude-ali launch
│                         # Your changes here will be OVERWRITTEN
│
└── .claude-local/        # GIT-TRACKED — edit here
    └── agents/           # Platform-visible after claude-ali overlay
                          # Your project agents live here
```

At `claude-ali` launch, the launcher automatically overlays `.claude-local/` content into `.claude/` via `ensure_project_local_overlay()`. This runs after plugin pack installation. Claude Code then reads the merged `.claude/` result — org agents from the plugin cache at `~/.claude/`, project agents from `PROJECT/.claude/agents/`.

**What this means:** Your project agent must be in `.claude-local/agents/`. Placing it in `.claude/agents/` makes it temporarily visible but it will be deleted on the next launch.

---

## When to Use Project-Local vs. Org-Wide

| Scenario | Use |
|----------|-----|
| Agent with knowledge specific to one project (domain expert, architecture reviewer) | `.claude-local/agents/` |
| Agent useful across all Aliunde projects | `~/ali-ai/agents/` (org-wide) |
| Expert agent for expert-panel discovery | `~/ali-ai/agents/` (org-wide, needs manifest) |
| Temporary or experimental agent in development | `.claude-local/agents/` |
| Agent that references project-specific file paths or domain rules | `.claude-local/agents/` |
| Agent you want versioned and distributed via tarball | `~/ali-ai/agents/` (org-wide) |

**Decision rule:** If only this project's team needs it, make it local. If any other Aliunde project would benefit, make it org-wide.

---

## Directory Structure

Project-local agents live here (git-tracked):

```
{project-root}/
├── .claude-local/               # Git-tracked project resources
│   ├── agents/                  # Project-only agents
│   │   └── {agent-name}.md      # One file per agent
│   └── skills/
│       └── {skill-name}/
│           └── SKILL.md
│
├── .claude/                     # Auto-generated at launch (gitignored)
│   └── agents/                  # Org agents land here after sync
│                                # .claude-local/agents/ is overlaid on top
│
└── .gitignore                   # Contains: .claude/  (NOT .claude-local/)
```

---

## Patterns We Use

### Minimal Agent Frontmatter (Required)

Every project agent needs valid frontmatter. The `name:` must match the filename exactly:

```yaml
---
name: {project-name}-expert
description: |
  {Project name} domain expert. Reviews {domain} patterns and validates
  implementation against project architecture.
model: sonnet
skills:
  - ali-agent-operations
tools:
  - Read
  - Grep
  - Glob
---
```

**`ali-` prefix in `skills:`:** Skills in the skills list still use their full name including the `ali-` prefix. Only the agent's own `name:` field omits the prefix.

### Project Agent with Write Access

For project agents that implement changes, add write tools scoped to appropriate paths:

```yaml
---
name: {project-name}-developer
description: |
  {Project name} developer. Implements features and fixes within
  {project-specific domain}.
model: sonnet
skills:
  - ali-agent-operations
  - ali-backend-developer
tools:
  - Read
  - Write
  - Edit
  - Grep
  - Glob
---
```

### Agent Body Template

Copy this for a new project agent:

```markdown
---
name: {agent-name}
description: |
  {One-line description of what this agent does and when to use it.}
model: sonnet
skills:
  - ali-agent-operations
tools:
  - Read
  - Grep
  - Glob
---

# {Agent Display Name}

{One-sentence description of what this agent does.}

## Step 0: Handshake (MANDATORY - FIRST OUTPUT)

Acknowledge task receipt immediately:

```
**HANDSHAKE:** {Agent Display Name} here. Received task to [brief summary]. Beginning work now.
```

**Why:** Orchestrators need confirmation that agent spawn succeeded.
Silent API Error 400 failures cause agents to never start.
Handshake is spawn verification.

**This is your FIRST output before any work begins.**

---

## Your Role

{What this agent is responsible for. What scope it operates within.
What project-specific knowledge it brings.}

---

## Workflow

### Step 1: {Verb} {Object}

{What this step does.}

### Step 2: {Verb} {Object}

{What this step does.}

---

## Output Format

{How the agent formats its results.}

---

## Error Handling

**Scenario: {error condition}**
1. {Detection}
2. {Recovery}
3. {Reporting}
```

---

## Naming Convention

Project agents do NOT use the `ali-` prefix. Use descriptive kebab-case:

```
# CORRECT - project agent names
.claude-local/agents/tax-domain-expert.md       # name: tax-domain-expert
.claude-local/agents/ingestion-reviewer.md      # name: ingestion-reviewer
.claude-local/agents/client-portal-developer.md # name: client-portal-developer

# WRONG - ali- prefix is for org agents only
.claude-local/agents/ali-tax-domain-expert.md

# WRONG - placed in wrong directory
.claude/agents/tax-domain-expert.md             # will be overwritten at next launch
```

The `name:` field in frontmatter must match the filename (without extension):

```
# File: .claude-local/agents/tax-domain-expert.md
# Frontmatter: name: tax-domain-expert   <-- must match
```

---

## .gitignore Check

Before adding a project agent, verify `.claude-local/` is NOT gitignored:

```bash
# This should return nothing (no match)
grep "claude-local" .gitignore

# If it returns something, remove the line from .gitignore
# .claude-local/ must be git-tracked for teammates to get project agents
```

The `.gitignore` should contain `.claude/` (auto-generated, gitignored) but NOT `.claude-local/` (git-tracked).

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Placing agent in `.claude/agents/` | Gitignored, overwritten at next `claude-ali` launch | Place in `.claude-local/agents/` |
| Using `ali-` prefix for project agents | Looks like an org agent, creates confusion | Omit prefix, use plain kebab-case |
| Filename mismatches `name:` field | Task tool resolves by name; mismatch causes spawn failure | Keep them identical |
| No Step 0: Handshake | Silent spawn failures are undetectable | Make handshake the first workflow output |
| `.claude-local/` in `.gitignore` | Project agent exists locally but teammates never get it | Remove `.claude-local/` from `.gitignore`; only `.claude/` is gitignored |
| Running `build_agent_manifest.py` for a project agent | Not needed; only org expert agents use the manifest | Skip — project agents are overlaid directly, no manifest entry |
| Forgetting to `git add .claude-local/` | Agent exists locally but teammates never get it | Add and commit `.claude-local/agents/` |
| Writing to `.claude-local/agents/` with wrong path | Agent file not found at runtime | Use absolute paths when writing files |

---

## Quick Reference

### Create a New Project Agent

```bash
# From your project root
mkdir -p .claude-local/agents
# Then use Write tool to create .claude-local/agents/{agent-name}.md

# Stage and commit
git add .claude-local/agents/{agent-name}.md
git commit -m "feat(agents): add {agent-name} project agent"

# Relaunch claude-ali to make agent visible
# The launcher overlays .claude-local/agents/ on top of .claude/agents/
```

### Verify Agent Is Loadable

```bash
# Confirm it is in .claude-local/ (NOT .claude/)
ls .claude-local/agents/{agent-name}.md   # Should exist

# Confirm it is NOT in .gitignore
grep "claude-local" .gitignore            # Should return nothing

# Confirm name matches filename
grep "^name:" .claude-local/agents/{agent-name}.md
# Output should be:  name: {agent-name}

# Confirm Step 0 Handshake is present
grep "Step 0" .claude-local/agents/{agent-name}.md

# The overlay runs automatically at claude-ali launch — confirm agent is visible
ls .claude/agents/{agent-name}.md         # Should exist after overlay
```

### Availability Timeline

| Action | Availability |
|--------|-------------|
| Write agent file in `.claude-local/agents/` + commit | Available automatically on next `claude-ali` launch — `ensure_project_local_overlay()` runs at session start |
| No release or tarball needed | Confirmed — project agents overlay via `.claude-local/` |
| Teammates get it | On their next `git pull` + `claude-ali` launch |

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| Project template (full `.claude-local/` reference) | `~/ali-ai/PROJECT_TEMPLATE.md` |
| Sync/overlay script | `~/ali-ai/scripts/sync-org-resources.sh (deprecated — Phase 1 only, no longer invoked by launcher)` |
| Example project agent in template | `~/ali-ai/PROJECT_TEMPLATE.md` ("Adding Project-Specific Agents" section) |
| Org agent examples to model from | `~/ali-ai/agents/ali-bash-expert.md` |
| Org agent builder (for org-wide agents) | `~/ali-ai/skills/ali-org-agent-builder/SKILL.md` |

---

**Document Version:** 1.0
**Last Updated:** 2026-03-24
