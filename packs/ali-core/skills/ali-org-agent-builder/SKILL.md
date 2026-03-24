---
name: ali-org-agent-builder
description: |
  Guide for creating org-wide agents in ~/ali-ai/agents/ with the ali- prefix. Use when:

  PLANNING: Deciding an agent should be available across all Aliunde projects,
  "I need an agent that can", "we need a new specialist agent", planning agent
  capabilities and tool access, evaluating whether a task needs a new agent vs.
  extending an existing one, naming a new org agent, scoping domains and keywords
  for expert-panel discovery

  IMPLEMENTATION: "create a new agent", "build an agent for", "add an org agent",
  "new specialist agent", writing a new agent file under ~/ali-ai/agents/, creating
  frontmatter with required fields, authoring the identity section, defining tool
  access, writing the workflow and error handling sections, registering the agent
  in AGENT_MANIFEST.json by running build_agent_manifest.py

  GUIDANCE: Asking how org agents are distributed to projects, asking about the
  ali- prefix convention, asking what fields are required in agent frontmatter,
  asking how agents differ from skills, asking how agents get discovered by
  expert-panel, asking what the manifest rebuild step does

  REVIEW: Reviewing an org agent for naming correctness, required frontmatter
  fields, Step 0 Handshake presence, expert-metadata completeness, manifest
  registration, and release readiness
---

# Org Agent Builder (Org-Wide Agents)

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Deciding an agent should be available to all Aliunde projects, not just one
- Planning what capabilities a new org-wide agent should have
- Choosing which tools to allow and which to restrict
- Evaluating whether to build a new agent or extend an existing one
- Designing expert-metadata keywords and domains for expert-panel routing
- Deciding between an expert agent (review-only) and a developer agent (write access)

**Implementation:**
- Creating a new agent file under `~/ali-ai/agents/`
- Writing frontmatter, identity section, workflow steps, and error handling
- Setting the tools list to enforce least-privilege access
- Authoring expert-metadata for dynamic expert-panel discovery
- Running `build_agent_manifest.py` to register the agent
- Committing and releasing the agent for distribution

**Guidance/Best Practices:**
- Asking how org agents are distributed to user machines (via tarball release)
- Asking about the `ali-` prefix requirement for org agent names
- Asking what the `expert-metadata` block does and when to include it
- Asking what release steps are needed before teammates get the new agent
- Asking how agents differ from skills and when to use each
- Asking why manifest rebuild is required after creating an expert agent

**Review/Validation:**
- Reviewing an org agent for `ali-` prefix and kebab-case naming
- Checking that required frontmatter fields are present (name, description, model, tools)
- Confirming Step 0: Handshake is the first workflow step
- Validating expert-metadata blocks for completeness and domain accuracy
- Confirming manifest rebuild was run after creating or updating an expert agent
- Checking that the agent does not write outside its domain

## When NOT to Use This Skill

This skill does NOT activate for:
- Creating skills for org-wide knowledge (use ali-org-skill-builder instead)
- Creating project-specific agents visible only in one project (use ali-local-agent-builder instead)
- Writing CLAUDE.md files (use ali-claude-code skill)
- Configuring hooks or settings (use ali-claude-code skill)
- Building MCP servers (use ali-mcp skill)

---

## Key Principles

- **`ali-` prefix is required** — every org agent name starts with `ali-` (e.g., `ali-bash-expert`, `ali-backend-developer`). The `name:` field and filename must match exactly.
- **Filename must match `name:` field** — `ali-bash-expert.md` requires `name: ali-bash-expert`. Mismatch causes routing errors.
- **Manifest rebuild is mandatory for expert agents** — after creating or updating an agent with `expert-metadata`, run `python3 scripts/build_agent_manifest.py`. Without it, expert-panel cannot discover the agent.
- **Step 0: Handshake is non-negotiable** — every agent's first workflow output must be the handshake. Silent spawn failures are undetectable without it.
- **Least-privilege tools** — list only the tools the agent actually needs. Expert agents typically use `Read, Grep, Glob`. Developer agents add `Write` and `Edit` scoped to their domain.
- **Release required to distribute** — org agents reach users via the ali-ai tarball. Commit + cut a new version + upload; users get it on next `claude-ali` launch.
- **Expert vs. developer agent** — expert agents review (Read/Grep/Glob only, write to `.tmp/**`). Developer agents implement (Read/Write/Edit scoped to domain). Do not conflate the two.
- **Skills list in frontmatter** — agents load skills via the `skills:` field. Always include `ali-agent-operations` as the first entry; add domain skills as needed.

---

## When to Use Org-Wide vs. Project-Local

| Scenario | Use |
|----------|-----|
| Agent that all Aliunde projects need (bash expert, backend developer) | `~/ali-ai/agents/` (org-wide) |
| Agent with knowledge specific to one project's domain | `.claude-local/agents/` (project-local) |
| Agent you want versioned, released, and distributed via tarball | `~/ali-ai/agents/` (org-wide) |
| Experimental or temporary agent in development | `.claude-local/agents/` (project-local) |
| Expert agent to be discovered by expert-panel | `~/ali-ai/agents/` (org-wide, with expert-metadata) |

**Decision rule:** If you'd want any Aliunde project to delegate to this agent, make it org-wide.

---

## Directory Structure

Org agents live here (tracked in ali-ai repo):

```
~/ali-ai/
├── agents/
│   ├── ali-{name}.md          # One file per agent
│   ├── AGENT_MANIFEST.json    # Auto-generated by build_agent_manifest.py
│   └── ...
│
└── scripts/
    └── build_agent_manifest.py   # Run after creating/updating expert agents
```

After release, org agents are installed into the Claude Code plugin cache via `claude plugin install ali-core@ali-team` — they are NOT copied to `PROJECT/.claude/agents/`. The platform loads them from `~/.claude/` (plugin cache location).

Users never edit `.claude/agents/` directly. Changes to org agents are made in `~/ali-ai/agents/`, released, and distributed via the plugin pack.

---

## Patterns We Use

### Minimal Agent Frontmatter (Required)

Every org agent requires these frontmatter fields. The `name:` must match the filename exactly:

```yaml
---
name: ali-{agent-name}
description: |
  One-line summary. Use for specific task types.
model: sonnet
skills: ali-agent-operations, ali-{domain-skill}
tools:
  - Read
  - Grep
  - Glob
---
```

**model values:** `sonnet` (default for most agents), `haiku` (lightweight tasks), `opus` (complex architecture decisions)

**tools format:** List format (one tool per line). Scoped write access uses glob notation: `Write(.tmp/**)`.

### Expert Agent with expert-metadata

Expert agents include an `expert-metadata` block for dynamic discovery by expert-panel. Place it in the frontmatter after `tools:`:

```yaml
---
name: ali-{domain}-expert
description: |
  {Domain} expert for reviewing {domain} implementations, patterns, and best practices.
  Use for reviews, security analysis, and best practice validation.
model: sonnet
skills: ali-agent-operations, ali-{domain}, ali-code-change-gate
tools:
  - Read
  - Write(.tmp/**)
  - Grep
  - Glob

# Dynamic expert selection metadata
expert-metadata:
  domains:
    - {primary-domain}
    - {secondary-domain}
  file-patterns:
    - "**/*.{ext}"
    - "**/domain-dir/**"
  keywords:
    - {keyword1}
    - {keyword2}
  anti-keywords:
    - {unrelated-domain}
---
```

**When to include expert-metadata:** Any agent intended to be discovered by `ali-expert-panel` needs this block. Expert agents (reviewers) always include it. Developer agents (implementers) typically do not.

### Developer Agent (Write Access)

Developer agents have write access scoped to their domain. They never write to files owned by another agent's domain:

```yaml
---
name: ali-{domain}-developer
description: |
  {Domain} developer. Implements {domain} features, fixes, and refactors.
model: sonnet
skills: ali-agent-operations, ali-{domain}, ali-code-change-gate
tools:
  - Read
  - Write
  - Edit
  - Grep
  - Glob
---
```

### Agent Body Structure

Every agent body follows this structure (adapted from ali-bash-expert.md pattern):

```markdown
# {Agent Display Name}

One-sentence description of what this agent does.

## Step 0: Handshake (MANDATORY - FIRST OUTPUT)

Acknowledge task receipt immediately:

```
**HANDSHAKE:** {Agent Name} here. Received task to [brief summary]. Beginning work now.
```

**Why:** Orchestrators need confirmation that agent spawn succeeded.
Silent API Error 400 failures cause agents to never start.
Handshake is spawn verification.

**This is your FIRST output before any work begins.**

---

## Your Role

{Brief description of what this agent does and its scope.}

---

## Operating Procedure

{How the agent approaches its work. Evidence requirements if an expert agent.}

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
1. {Detection step}
2. {Recovery step}
3. {Reporting step}
```

---

## Naming Convention

Org agents use the `ali-` prefix followed by descriptive kebab-case. The filename and `name:` field must be identical:

```
# CORRECT - org agent naming
~/ali-ai/agents/ali-bash-expert.md       # name: ali-bash-expert
~/ali-ai/agents/ali-backend-developer.md # name: ali-backend-developer
~/ali-ai/agents/ali-aws-admin.md         # name: ali-aws-admin

# WRONG - missing ali- prefix
~/ali-ai/agents/bash-expert.md
~/ali-ai/agents/BackendDeveloper.md
```

The `-expert` suffix is used for review-only agents (Read/Grep/Glob). The `-developer` suffix is for implementation agents (Write access). The `-admin` suffix is for infrastructure/ops agents.

---

## Release Process

Org agents require a release before teammates receive them. Project-local agents do not.

### Steps

1. Create the agent file in `~/ali-ai/agents/ali-{name}.md`
2. If the agent includes `expert-metadata`, rebuild the manifest:
   ```bash
   cd ~/ali-ai
   python3 scripts/build_agent_manifest.py
   ```
3. Commit to the ali-ai repo:
   ```bash
   git add agents/ali-{name}.md agents/AGENT_MANIFEST.json
   git commit -m "feat(agents): add ali-{name} agent"
   ```
4. Build the tarball locally:
   ```bash
   bash scripts/build-local.sh
   ```
5. Cut a new version and upload the release:
   ```bash
   gh release upload v{X.Y.Z} dist/ali-ai-v{X.Y.Z}.tar.gz
   ```
6. Users receive the agent on their next `claude-ali` launch when the launcher detects the new ali-core pack version via marketplace and installs the updated pack. Agents load from the plugin cache — not from `.claude/agents/`.

**Important:** Skills do NOT need `build_agent_manifest.py`. Only agents with `expert-metadata` need it.

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Missing `ali-` prefix | Routes as project agent; expert-panel cannot discover it | Name it `ali-{name}` |
| Filename differs from `name:` field | CoS and Task tool resolve by name; mismatch causes spawn failure | Keep them identical: `ali-foo.md` = `name: ali-foo` |
| No Step 0: Handshake | Silent spawn failures are undetectable | Make handshake the first workflow output |
| Expert agent with Write tool (not scoped) | Expert agents can overwrite source files | Scope writes: `Write(.tmp/**)` only |
| Expert-metadata added but manifest not rebuilt | expert-panel cannot discover the new agent | Run `python3 scripts/build_agent_manifest.py` after every expert-metadata change |
| Developer agent with all tools unrestricted | Agent can modify any file, including files owned by other agents | Scope `Write` and `Edit` to agent's domain files |
| No `ali-agent-operations` in skills list | Agent lacks handshake protocol and evidence requirements | Always include `ali-agent-operations` as first skill |
| Forgetting to release | Teammates never get the agent | Commit + `bash scripts/build-local.sh` + `gh release upload` |
| Editing `.claude/agents/` directly | Gitignored; overwritten at next sync | Edit in `~/ali-ai/agents/`, then release |

---

## Quick Reference

### Create a New Org Agent

```bash
# From ~/ali-ai
# Use Write tool to create agents/ali-{name}.md

# If agent has expert-metadata, rebuild manifest
python3 scripts/build_agent_manifest.py

# Validate name matches filename
grep "^name:" agents/ali-{name}.md
# Should output:  name: ali-{name}

# Commit (include AGENT_MANIFEST.json if manifest was rebuilt)
git add agents/ali-{name}.md agents/AGENT_MANIFEST.json
git commit -m "feat(agents): add ali-{name} agent"

# Build and release
bash scripts/build-local.sh
gh release upload v{X.Y.Z} dist/ali-ai-v{X.Y.Z}.tar.gz
```

### Validate Before Release

```bash
# Confirm name matches filename
grep "^name:" agents/ali-{name}.md

# Confirm Step 0 Handshake is present
grep -n "Step 0" agents/ali-{name}.md

# Confirm expert-metadata present (for expert agents)
grep -n "expert-metadata" agents/ali-{name}.md

# Confirm manifest was rebuilt (timestamp updated)
grep "generated_at" agents/AGENT_MANIFEST.json
```

### Availability Timeline

| Action | Availability |
|--------|-------------|
| Write agent file + commit | Not yet (unreleased) |
| `bash scripts/build-local.sh` + `gh release upload` | Available on next `claude-ali` launch for users who pull the new version |
| User runs `claude-ali` | Launcher detects new ali-core pack version via marketplace, installs updated pack. Agents load from plugin cache — not from `.claude/agents/`. |

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| Agent manifest generator | `~/ali-ai/scripts/build_agent_manifest.py` |
| Local tarball build script | `~/ali-ai/scripts/build-local.sh` |
| Agent manifest (output) | `~/ali-ai/agents/AGENT_MANIFEST.json` |
| Style reference (expert agent) | `~/ali-ai/agents/ali-bash-expert.md` |
| Style reference (developer agent) | `~/ali-ai/agents/ali-bash-developer.md` |
| Style reference (admin agent) | `~/ali-ai/agents/ali-aws-admin.md` |
| Project template (agent section) | `~/ali-ai/PROJECT_TEMPLATE.md` |

---

**Document Version:** 1.0
**Last Updated:** 2026-03-24
