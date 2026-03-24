---
name: ali-cos-staff-discovery
description: |
  Agent routing and staff discovery reference for the Chief of Staff role. Use when:

  PLANNING: Session start, planning who to involve in a task, selecting agents for a
  multi-step plan, deciding whether a task needs a specialist or orchestrator

  IMPLEMENTATION: Discovering available agents via manifest, invoking the Skill tool
  for manifest lookup, routing a specific request to the correct specialist or admin agent

  GUIDANCE: "Which agent handles this?", "Who should I use for X?", "What agents are
  available?", listing available staff and their domains

  REVIEW: Confirming correct agent selection, reviewing routing decisions before
  delegation, validating that the right specialist was chosen for a task type
---

# CoS Staff Discovery

Reference material for the Chief of Staff role: how to discover available agents, which specialist handles which domain, how to use admin agents and orchestrators, and how to route tasks to the right staff member.

---

## Your Staff

You manage specialists and admin agents. Know when to deploy each.

### Staff Discovery (Manifest-Based)

**Available agents are discovered via pre-built manifest generated at session start.**

**How agent discovery works:**

At every session start, the SessionStart hook `ali-agent-routing.sh` runs automatically. It reads `~/.claude/plugins/installed_plugins.json` to determine which plugin packs are installed, derives the installed agent set from the plugin cache directories, then loads the org catalog from `~/ali-ai/agents/AGENT_MANIFEST.json`. It classifies each agent as installed or uninstalled and writes two output files:

- `.claude/agent-routing.md` — installed agents classified by role (Expert, Developer, Admin, Orchestrator, Other)
- `.claude/agent-suggestions.md` — uninstalled agents from the org catalog with their pack name annotated

Both files are generated on every session start. CoS reads them directly using the Read tool at session start — no Bash command or delegation required.

**If agent-routing.md is missing:**
- Report to the engineer: "Agent routing table not found at .claude/agent-routing.md"
- This indicates a session start hook failure — check that ali-agent-routing.sh ran and that ~/.claude/plugins/installed_plugins.json exists

**Workflow:**
1. **At session start:** Read .claude/agent-routing.md and .claude/agent-suggestions.md to identify available staff and surfaceable suggestions
2. **When routing tasks:** Match task requirements to agent capabilities from the routing table
3. **For uninstalled agents:** Read agent-suggestions.md to identify domain packs that would be a better fit and surface them to the engineer

### Reading the Active Routing Table

**CoS reads .claude/agent-routing.md directly using the Read tool. No delegation required.**

```
Read: /path/to/project/.claude/agent-routing.md
```

This file is generated on every session start by ali-agent-routing.sh from installed_plugins.json and the org catalog. It contains the current roster of installed agents organized by role (specialist, admin, orchestrator), making it the fastest way to answer "who is available right now?"

**.claude/agent-routing.md is on the CoS explicit Read allowlist** alongside .claude/AGENT_MANIFEST.json. Reading it is a routing decision (allowed), not implementation work (requires delegation). See Chief-of-Staff-Core.md Read Tool Boundaries.

**When to read agent-routing.md directly:**
- Session start: confirm roster before routing first task
- "Who handles X?" questions that need a quick name lookup
- Verifying an agent name before writing an ISPR

### Reading Agent Suggestions

**CoS reads .claude/agent-suggestions.md directly using the Read tool. No delegation required.**

```
Read: /path/to/project/.claude/agent-suggestions.md
```

This file is generated at session start alongside agent-routing.md. It lists agents that exist in the org catalog but are not installed in the current project, with their domain pack name annotated.

**.claude/agent-suggestions.md is on the CoS explicit Read allowlist.** Reading it at session start is a routing decision, not implementation work.

**When to read agent-suggestions.md:**
- Session start: note available uninstalled packs before the first routing decision
- When routing a task and the installed agents feel like a poor fit
- When the engineer asks "is there a better agent for this?"
- When starting a new type of work not previously done in this project (e.g., Azure infra, Fabric pipelines)

**What to do when an agent appears in agent-suggestions.md:**
1. Note the pack name annotated next to the suggestion (e.g., "ali-azure-expert — Available in: ali-azure pack")
2. Surface it to the engineer: "ali-azure-expert is available but not installed — it's in the ali-azure pack"
3. Do not route to a weaker installed agent as a workaround without mentioning the better option
4. Engineer decides whether to install the pack via plugin-config.json

**If agent-suggestions.md is absent:** The hook may not have found any uninstalled agents, or the org catalog was unavailable. This is not an error — proceed with agent-routing.md only.

**Key principle:** Surfacing a suggestion closes the domain pack opt-in discovery loop. An uninstalled expert is not absent — it is opt-in. Always name it.

### Expert Panel Selection (Plan-Specific Discovery)

**Use `list_experts.py --show-suggestions` when you need agents scored against a specific plan, not a generic session roster.**

```
python ~/ali-ai/scripts/list_experts.py --show-suggestions <path-to-plan.md>
```

**When to use:**
- Assembling an expert panel for plan-builder design or implement mode
- Any time a task is scoped to a specific plan file and you need the best-fit specialists
- When generic session roster (agent-routing.md) gives you a broad list but the plan is narrowly scoped

**What it returns:** A JSON object with two arrays:
- `recommended` — installed agents scored by relevance to the plan, sorted by score
- `suggestions` — uninstalled agents from the org catalog that would be relevant, with pack name

**How it differs from agent-routing.md:** agent-routing.md gives the full installed roster organized by role. `list_experts.py --show-suggestions` scores agents against the content of a specific plan and surfaces both installed recommendations and relevant uninstalled candidates in one call.

**Action on suggestions:** Same as agent-suggestions.md — name the pack, surface to the engineer, do not route to a weaker installed agent without mentioning the better option.

### Specialist Agents

Spawn for focused work in specific domains. Each agent is backed by skills that provide domain knowledge.

**Common patterns:**
- Single-domain tasks (security review, database optimization, frontend component)
- Focused implementation (add feature, fix bug, write tests)
- Targeted analysis (performance review, compliance check)

### Admin Agents

Spawn for administrative operations requiring elevated permissions.

**Admin agents follow the ali-*-admin naming convention.** To find available admin agents:
- Read .claude/agent-routing.md (admin section) — fastest lookup

**Delegation modes:**

**Explicit execution mode** (when you provide specific instructions):

```
Execute immediately - delegated execution mode:

1. Stage these files: [exact file list]
2. Commit with message: "[message]"
3. Push to origin

Do not ask about other files. Execute and report completion.
```

**Interactive mode** (when scope is exploratory):

```
Review git status and recommend next steps for cleaning up this branch.
```

**Key principle:** If you provide explicit file lists and operations, admin agent executes without asking clarifying questions. If scope is open-ended, agent will ask for guidance.

### Orchestrators

Spawn when multiple specialists need coordination.

**To find available orchestrators:** Read .claude/agent-routing.md (orchestrator section). Orchestrators follow project-specific naming and are listed there with their domains.

Do not maintain a hardcoded list of orchestrator names here — agent-routing.md is the current source of truth and is regenerated on every session start.

---

## Domain-Specific Routing

Before generic delegation, check for domain-specific orchestrators that manage their own specialist workflows:

| Domain | Keywords / Patterns | Route To |
|--------|---------------------|----------|
| Presentations | presentation, deck, slide, pitch, proposal deck, storyboard, QBR, training deck, build slides, **/presentations/**, **/slides/**, **/manifest.json | slide-ops orchestrator (check agent-routing.md for current name) |
| macOS config file writes | iTerm2 Dynamic Profiles, Launch Agents, ~/Library/, macOS config, plist, launchd | bash-developer agent |
| macOS automation review | AppleScript, Shortcuts, osascript, Quick Action, review | macOS-expert agent |

**Why:** Domain orchestrators (like slide-ops) have purpose-built workflows, specialist routing, and quality gates. Routing through them ensures the full pipeline runs. Routing directly to a generic frontend-developer bypasses these gates and produces non-compliant output.

**macOS routing distinction:** Use bash-developer when the task involves writing files to disk (config files, plist files, ~/Library/ paths, Launch Agents). Use macOS-expert when the task involves reviewing, advising on, or understanding macOS automation patterns (AppleScript, Shortcuts.app, osascript, Quick Actions).

**To resolve role labels to current agent names:** Read .claude/agent-routing.md.

---

## Delegation Decision Tree

```
New task arrives
    │
    ├─ Is this a QUESTION about status/progress?
    │   └─ Answer from your situational awareness (no agent needed)
    │
    ├─ Is this a STRATEGIC DISCUSSION?
    │   └─ Engage directly with the engineer (your job)
    │
    ├─ Does it match a DOMAIN-SPECIFIC ORCHESTRATOR?
    │   └─ Route to that orchestrator (see Domain-Specific Routing table above)
    │
    └─ Does this require DELEGATION?
        │
        ├─ Present ISPR CHECKPOINT block FIRST
        │   └─ Identify task, Summarize context, Propose agent(s), Request approval via AskUserQuestion
        │
        └─ After approval:
            │
            ├─ Is this an ADMIN operation (git, AWS account)?
            │   └─ Spawn the appropriate admin agent (check agent-routing.md for -admin agents)
            │
            ├─ SINGLE-DOMAIN task?
            │   └─ Spawn the appropriate specialist agent (match domain to agent from agent-routing.md)
            │
            └─ RESEARCH/EXPLORATION?
                └─ Spawn Task with subagent_type=Explore

```

**Every delegation goes through ISPR. No exceptions.**

---

## Write Delegation Routing

When delegating write operations, use the role label below to identify the correct developer. Resolve the current agent name from .claude/agent-routing.md before writing the ISPR.
| File Type | Developer Role | Examples |
|-----------|----------------|----------|
| Documentation | document-developer | ARCHITECTURE.md, README.md, docs/**, roles/*.md |
| Agents | claude-developer | agents/*.md |
| Skills | claude-developer | skills/**/SKILL.md |
| Claude Config | claude-developer | .claude/settings.json, hooks, CLAUDE.md |
| Backend Code | backend-developer | Python, FastAPI, services, APIs |
| Frontend Code | frontend-developer | React, TypeScript, components |
| Tests | test-developer | pytest, jest, test files |
| Database | database-developer | migrations, SQL, schemas |
| macOS Config Files | bash-developer | Launch Agents, iTerm2 profiles, .plist, ~/Library/ writes |

**Resolution steps:** (1) Identify file type from the table. (2) Note the Developer Role label. (3) Read .claude/agent-routing.md to resolve the role label to the current installed agent name. (4) Use the resolved agent name in your ISPR.

**Anti-pattern:** Routing agents/*.md or skills/**/SKILL.md to document-developer is a common misroute. These are Claude Code configuration artifacts, not documentation.

**Pattern:** Match file extension and location to developer role, not task description.

---

## Tool Permission Standards

When routing tasks, understand what each agent class can and cannot do. This determines whether a task requires a developer (who can write files) or an expert (who can only review).

| Agent Class | Allowed Tools | Scope |
|-------------|--------------|-------|
| Expert agents | Read, Write(.tmp/**), Grep, Glob | Review only - findings written to .tmp/, no production file writes |
| Developer agents | Read, Grep, Glob, Edit, Write, Bash | Full implementation - can write any file, execute commands |
| Admin agents | Bash, Read, Grep, Glob | Operational command execution - git, AWS CLI, system ops |

**Routing implication:** If a task requires writing a file outside .tmp/ (a config file, a script, a source file), it must go to a developer or admin agent, not an expert agent. Expert agents produce findings and recommendations; developer agents implement them.

**Common misroute to avoid:** macOS config file writes (iTerm2 profiles, Launch Agents, plist files) look like macOS automation work and attract macOS-expert, but writing to ~/Library/ requires developer-class tools. Route to bash-developer instead.

**CoS Agent Tool Rule:** CoS (main session) uses Task exclusively to spawn all agent classes listed above. The Agent platform tool bypasses this routing and is prohibited. See Chief-of-Staff-Core.md Prohibited Tools section.

---

## Background Delegation Rule

**Before setting run_in_background: true on any Task delegation, CoS must check the background_safe field in .claude/AGENT_MANIFEST.json.**

### Classification Lookup

```
Read: /path/to/project/.claude/AGENT_MANIFEST.json
```

Find the entry for the target agent and check the `background_safe` field:

| background_safe value | Meaning | Action |
|-----------------------|---------|--------|
| `true` | Agent has no Bash in its tools field — no guard hook exposure | Background execution permitted |
| `false` | Agent is Bash-capable (guard hook exposed) or is ali-plan-builder | Foreground only — do NOT set run_in_background: true |
| field absent | Manifest predates this feature — unclassified | Foreground only (fail-safe) |

### Decision Rule

```
Before setting run_in_background: true:
    │
    ├─ Read .claude/AGENT_MANIFEST.json
    ├─ Find the target agent entry
    ├─ Check background_safe field
    │
    ├─ background_safe: true  → Background permitted. Proceed.
    │
    └─ background_safe: false OR field absent → FOREGROUND ONLY.
        Do not set run_in_background: true.
        Remove the flag and spawn as foreground.
```

### Why This Matters

Background agents cannot service write permission prompts. If a Bash-capable agent is spawned in background and a guard hook fires (e.g., ali-git-guard, ali-aws-guard), the hook's exit 2 is unactionable — the agent fails silently with no recovery path.

**Classification source:** The `background_safe` field is derived from each agent's frontmatter `tools:` field by build_agent_manifest.py. Agents without `Bash` in their tools field are classified `background_safe: true`. Agents with `Bash` are classified `background_safe: false`. ali-plan-builder is hardcoded `background_safe: false` regardless of tools (Task scope allows spawning Bash sub-agents).

### Two-Layer Enforcement

- **CoS (this rule):** Smart enforcement at delegation time — manifest-aware, agent-aware. Primary policy layer.
- **ali-background-agent-guard.sh (hook):** Dumb backstop — enforces structural ban on run_in_background: true unconditionally. Does NOT consult the manifest. Catches cases where CoS policy is bypassed.

The hook is not a substitute for this check. CoS is the intelligent enforcement layer. The hook catches failures; CoS prevents them.

---

**Last Updated:** 2026-03-22
