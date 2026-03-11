---
name: ali-context-assembler
description: |
  Assembles project and org documentation into a structured context-package.md
  for use as plan-builder expert-panel input. Use when:

  PLANNING: Preparing context before running plan-builder; assembling project
  context for a new feature; gathering relevant docs before expert review.

  IMPLEMENTATION: Writing context-package.md to a feature workspace; reading
  ARCHITECTURE.md, architecture_stack.txt, design principles, and CLAUDE.md
  to produce a structured summary.

  GUIDANCE: Asking how to assemble project context; asking what context
  experts need; asking how context-package.md relates to plan-builder.

  REVIEW: Checking whether a context package is complete; validating that
  org and project design principles are correctly merged.
keywords:
  - context assembler
  - context package
  - assemble context
  - project context
  - feature context
  - context-package.md
---

# Context Assembler Skill

This skill guides the main session (as CoS) through assembling project and org documentation into a structured context-package.md. The context package becomes input to the plan-builder expert-panel workflow, giving experts the project context they need to make informed recommendations.

## When to Use

**Load this skill before running plan-builder when you want experts to have full project context.** It is also useful standalone when you need a structured summary of a project's current state.

## When NOT to Use This Skill

- For tasks that do not involve plan-builder or expert review — use CLAUDE.md and ARCHITECTURE.md directly
- When the feature workspace already contains a recent context-package.md — check before re-assembling
- When context budget is critically constrained — context assembly adds ~2000 words to context window

---

## Key Principles

- Org design principles (ali-design-principles.md) take precedence over project-specific ones when they conflict; project principles extend or refine org principles, they do not override them
- Context budget is ~2000 words — enough to inform experts without overwhelming their context windows
- Missing input documents are noted but do not block assembly — partial context is better than no context
- This skill is not user-invoked directly — it is called as part of the plan-builder workflow or by CoS before spawning expert-panel
- Always check docs/feature/.active-feature before assuming a feature name
- Delegate file writing to an appropriate developer agent (CoS does not write files directly)

---

## Workflow

### Step 0: Determine Feature Workspace Path

Before reading any documents, resolve the output location.

```
Read: docs/feature/.active-feature
```

- If file exists: use the feature name it contains (e.g., `auth-system`) → output to `docs/feature/auth-system/context-package.md`
- If file does not exist: require a feature-name parameter from the caller → output to `docs/feature/<feature-name>/context-package.md`
- If the feature directory does not exist yet, it must be created before writing

### Step 1: Read Project Documentation

Read these documents in order. Skip any that do not exist and note the absence.

| Document | Path | Required? |
|----------|------|-----------|
| Project architecture | ARCHITECTURE.md | Required |
| Tech stack | architecture_stack.txt | Required |
| Org design principles | ~/.claude/docs/ali-design-principles.md | Required |
| Project design principles | docs/design-principles.md | Optional |
| Project configuration + handoff | CLAUDE.md | Required |
| Prior feature workspace files | docs/feature/<feature-name>/ | Optional |

**Reading strategy (CoS):**
- CoS may read ARCHITECTURE.md and CLAUDE.md directly (routing-level reads)
- Delegate reading architecture_stack.txt, design principles, and feature workspace files to a specialist agent if context is constrained
- Note which documents were found vs missing before proceeding

### Step 2: Assemble Context Package

Delegate writing of context-package.md to ali-claude-developer or an appropriate writer agent.

**Output file:** `docs/feature/<feature-name>/context-package.md`

**Output format:**

```markdown
# Context Package: <feature-name>

Generated: <YYYY-MM-DD>
Project: <project-name>
Feature: <feature-name>

## Project Overview
Summary from ARCHITECTURE.md — what the project does, key components,
directory structure. Keep to 3-5 sentences.

## Technology Stack
Structured summary from architecture_stack.txt — languages, frameworks,
databases, infrastructure. Use bullet lists for scannability.

## Design Principles
Merged org + project principles. Org principles first; project principles
extend or refine. Flag any project principles that conflict with org principles.

**Org principles (non-negotiable):**
- [key constraints from ali-design-principles.md]

**Project-specific extensions:**
- [project principles from docs/design-principles.md, if file exists]
- (None — docs/design-principles.md not found) [if file missing]

## Active Constraints
Non-negotiable rules from CLAUDE.md, design principles, and project config
that constrain implementation choices for this feature. This section answers
the question: "What must experts keep in mind when reviewing this plan?"

## Session Context
Relevant context from CLAUDE.md session handoff — what was recently completed,
what's pending, key architecture facts. Omit anything not relevant to this feature.

## Prior Feature Context
(If existing feature workspace files found — context-package.md, plan.md, implementation-log.md)
Summary of previous planning output, decisions made, and implementation progress.
Include status field from plan.md if present.

(If no prior feature workspace files exist, omit this section.)

## Relevant Prior Decisions
(If docs/feature/*/decisions/ or docs/plans/ contain material relevant to this feature)
Key decisions that affect this feature's approach.

(If nothing relevant found, omit this section.)
```

**Size target:** Keep total output under 2000 words. If source material is large, summarize aggressively — this is a briefing document, not a transcript.

### Step 3: Return Summary

After writing, return a brief summary to CoS:

```
Context package written to: docs/feature/<feature-name>/context-package.md

Assembled from:
- ARCHITECTURE.md (found)
- architecture_stack.txt (found)
- ~/.claude/docs/ali-design-principles.md (found)
- docs/design-principles.md (not found — omitted)
- CLAUDE.md (found)
- docs/feature/<feature-name>/ (prior context: none / found: <files>)

Total: ~<word count> words
```

---

## Integration with Plan-Builder

The context-package.md is referenced in the plan-builder Step 1 expert-panel prompt as supplementary input alongside the source document:

```
Source: {source document path or inline context}
Context: docs/feature/<feature-name>/context-package.md
Project: {project path}
Output-Path: docs/feature/<feature-name>/
```

**When to assemble context:**
- Run Context Assembler before step 1 of plan-builder when experts need project context
- Skip if context-package.md already exists and was generated recently (same session or same day)
- Re-assemble if the session handoff has changed or if ARCHITECTURE.md was updated

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Including full document text in context package | Overwhelms expert context windows | Summarize aggressively; 3-5 sentences per section |
| Project principles overriding org principles | Violates ali-design-principles.md hierarchy | Org principles first; project principles extend only |
| Skipping assembly when context-package.md already exists | Unnecessary re-work | Check modification date; re-use if current |
| Writing to .tmp/ instead of feature workspace | Context not persisted or shared | Always write to docs/feature/<feature-name>/ |
| Assembling without knowing the active feature | Context written to wrong location | Always read .active-feature first |
| CoS writing files directly | Violates CoS role boundary | Delegate Write operations to a developer agent |

---

## File Structure Reference

```
docs/feature/
├── .active-feature                    # sentinel: active feature directory name
└── <feature-name>/
    ├── README.md                      # entry point
    ├── context-package.md             # THIS SKILL'S OUTPUT — input to plan-builder
    ├── plan-brief.md                  # expert-panel output
    ├── plan.md                        # synthesis output (status: active|completed|archived)
    ├── executive-summary.md           # CoS-ready synthesis summary
    ├── expert-findings/               # individual expert reports
    ├── decisions/                     # approved decisions
    └── implementation-log.md         # running implementation log
```

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| Org design principles | ~/ali-ai/docs/dist/ali-design-principles.md (source) |
| Deployed design principles | ~/.claude/docs/ali-design-principles.md (runtime) |
| Feature Workspace convention | ~/ali-ai/ARCHITECTURE.md lines 136-164 |
| Feature Workspace template | ~/ali-ai/PROJECT_TEMPLATE.md "Feature Workspace" section |
| Plan-builder skill | ~/ali-ai/skills/plan-builder/SKILL.md |
| Active feature sentinel | docs/feature/.active-feature (per project) |

---

**Version:** 1.0
**Created:** 2026-02-23
**Category:** Workflow Automation (Category 2)
