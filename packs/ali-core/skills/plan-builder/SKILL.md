---
name: plan-builder
description: |
  Two-mode design-first workflow for implementation planning. Orchestrates design
  validation before implementation planning through expert collaboration and
  iteration-until-clean review cycles.

  Design mode: Given a feature brief, produces a validated design doc.
  Implement mode: Given an approved design doc, produces an implementation plan.

  Use when:
  - Creating implementation plans that need expert review
  - Validating a feature design before committing to implementation
  - Converting feature briefs into approved design documents
  - Producing phase-separated implementation plans from approved designs
  - Iterating plans until they pass structured clean criteria
  - Converting findings into remediation plans
  - Formalizing plans that require multi-domain sign-off
trigger: /plan-builder
keywords:
  - build a plan
  - create a plan
  - plan builder
  - implementation plan
  - expert review plan
  - remediation plan
  - get expert feedback
  - expert panel review
  - design mode
  - implement mode
  - draft mode
  - design doc
  - design-first
  - plan review
  - PASS
  - FAIL
  - clean criteria
  - iteration
---

# Plan Builder Skill

This skill guides the main session (as CoS) through the plan-builder workflow. Plan-builder is a two-mode design-first skill: design mode validates an approach before implement mode plans the execution.

---

## Mode Detection

Mode is inferred from input context — no CLI flags required:

| Input Context | Mode Triggered |
|---------------|----------------|
| Feature brief (no design doc present) | Design mode |
| Approved design doc (status: approved) | Implement mode |
| Explicit "design mode" / "implement mode" request | As requested |
| Ambiguous — both present, status unclear | CoS asks for clarification |

**Clarification prompt (ambiguous case):** "Is there an approved design doc, or should I start with design?"

---

## Draft Mode

Draft mode produces advisory output from a single reviewer pass. No iteration loop runs.

**Triggered by:** "draft mode" or "--draft" in the request (conversational — not a CLI flag).

**Effect:**
- Reviewer runs exactly once
- Verdict output is advisory only (no PASS/FAIL gate, no iteration loop)
- Plan-builder produces output regardless of verdict content
- Advisory findings are surfaced to the user

**Applies to:** Either design mode or implement mode.

---

## File Locations

| File | Purpose | Mode |
|------|---------|------|
| `docs/feature/{name}/design.md` | Design document output | Design |
| `docs/feature/{name}/plan.md` | Implementation plan output | Implement |
| `docs/feature/{name}/expert-findings/design/` | Per-expert review files (design mode) | Design |
| `docs/feature/{name}/expert-findings/implement/` | Per-expert review files (implement mode) | Implement |
| `docs/feature/{name}/codebase-context.md` | Codebase exploration summary | Implement |
| `docs/feature/{name}/executive-summary.md` | CoS-ready summary | Design |
| `docs/feature/{name}/implementation-log.md` | Phase-by-phase completion log | Both |
| `.tmp/{review-id}/reviewer-verdict-{N}.md` | Reviewer verdict per cycle | Both |

---

## Design Mode Walkthrough

Design mode takes a feature brief and produces a validated design document.

### STEP 1/4 — Draft Design Doc

Spawn `ali-document-developer` to write the initial design document at `docs/feature/{name}/design.md` using the feature brief.

### STEP 2/4 — Expert Review

Spawn domain experts (manifest score >=30) + `ali-technical-feasibility-expert` in parallel to review the design doc. Each expert writes findings to `docs/feature/{name}/expert-findings/design/{expert-name}-findings.md`. Experts return: filename + 2-sentence summary only (not full findings in context).

### STEP 3/4 — Synthesis

Spawn `ali-merge-synthesis-expert` to merge expert findings from `docs/feature/{name}/expert-findings/design/` into `docs/feature/{name}/executive-summary.md`.

### STEP 4/4 — Iterate Until Clean (or Draft advisory)

**Default (iterate-until-clean):**

1. Spawn `ali-technical-feasibility-expert` as reviewer with the Design Mode Clean Criteria (below) provided verbatim in the delegation prompt.
2. Reviewer writes verdict to `.tmp/{review-id}/reviewer-verdict-{N}.md`.
3. Reviewer returns: `REVIEWER_VERDICT: PASS | FAIL` + filename only.
4. CoS reads first line of verdict file only.
5. On PASS: proceed to User Approval.
6. On FAIL: spawn `ali-document-developer` to revise `design.md` — pass verdict file path only (not full findings). Increment cycle. Repeat from step 1.
7. Domain experts do NOT re-run on iterations 2+.
8. Safety valve triggers after 5 FAIL cycles — see Safety Valve section.

**Draft mode (single-pass advisory):**
- `ali-technical-feasibility-expert` runs once.
- Verdict is advisory. CoS skips the PASS/FAIL gate.
- Proceed to User Approval regardless of verdict.

### User Approval

Present `executive-summary.md` to user. Ask via AskUserQuestion: Approve design / Request changes / Cancel.

On user approval: CoS delegates to `ali-document-developer` to set `status: approved` in `design.md`.

---

## Design Mode Clean Criteria

These criteria are passed verbatim to the reviewer in the delegation prompt. All must be true for PASS.

1. **Decision made:** exactly one approach chosen; alternatives section has >=3 alternatives with rejection rationale for each rejected
2. **Acceptance criteria measurable:** every criterion is verifiable (a test or verification step can be written for it)
3. **Feasibility validated:** ali-technical-feasibility-expert returns APPROVE or APPROVE with notes
4. **No open blocking issues:** no BLOCKING entries in reviewer verdict file
5. **No unresolved open questions:** all Open Questions rows have Status: RESOLVED; any PENDING rows requiring human input trigger immediate user escalation (not an iteration cycle)
6. **Scope boundaries present:** "What This Design Does Not Change" section is populated
7. **Source references factually accurate:** Plan-agent must write claims about source
   artifacts that are verifiable from the source or explicitly flagged VERIFY:. Plan-agent
   habit: when making a factual claim about a source artifact, note the source file
   parenthetically (per DATABASE_SCHEMA.sql) — this is a traceability habit, not a required
   citation format.
   Reviewer check: any claim in the design doc about a source artifact (syntax, structure,
   count, schema shape) is either (a) not contradicted by the referenced file as read by
   the reviewer, or (b) explicitly flagged with VERIFY: in the design doc; claims against
   inaccessible source files are recorded as OPEN_QUESTIONS, not BLOCKING

---

## Phase Sizing Criteria

A well-formed implementation phase must meet all three criteria:

1. **Single coherent activity type:** each phase belongs to exactly one activity type — code authoring, test execution, review, documentation, or configuration. Combinations are not permitted.
2. **Completable in one agent session:** the phase can be completed by one agent in one session without context compaction.
3. **Terminated by a concrete artifact:** each phase ends with an artifact CoS can inspect — changed files, test results, a reviewed document — not "implementation complete."

**Activity types (exactly five):**
- **Code authoring:** creating or modifying source files
- **Test execution:** running any test suite
- **Review:** reading output and producing a verdict
- **Documentation:** updating docs, comments, or changelogs
- **Configuration:** updating environment files, settings, or infrastructure config

**Split rule:** When a proposed phase spans multiple activity types, plan-builder must split it. More phases is the correct outcome. A 5-phase plan that bundles write+test should become 7-8 phases.

**Before/after example:**

| Bad Phase (bundled, multiple types) | Split Into |
|-------------------------------------|-----------|
| "Implement auth middleware and run e2e suite" | Phase N: Implement auth middleware (code); Phase N+1: Run e2e suite (test) |
| "Update 18 spec files and run integration tests" | Phase N: Update spec files (code); Phase N+1: Run integration tests (test) |
| "Fix helpers.ts and update ARCHITECTURE.md" | Phase N: Fix helpers.ts (code); Phase N+1: Update ARCHITECTURE.md (docs) |

**Enforcement:** Phase sizing criteria are enforced at two points:
1. By plan-builder (ali-plan-agent) at draft time — any phase that bundles activity types is split before the draft is returned
2. By ali-plan-reviewer at review time via Implement Mode Clean Criterion 1

---

## Implement Mode Walkthrough

Implement mode takes an approved design document and produces a phase-separated implementation plan.

**Pre-condition check:** Before starting, verify `docs/feature/{name}/design.md` has `status: approved` in its frontmatter or header. If not found or status is not "approved", block with: "Design doc not yet approved. Update `docs/feature/{name}/design.md` status to 'approved' before running implement mode."

### STEP 1/3 — Verify and Codebase Exploration

Confirm `status: approved` in `docs/feature/{name}/design.md` — block if absent.

Spawn codebase exploration agent (Anthropic Plan agent via Task tool with constrained exploration prompt, or `ali-context-assembler` if Plan agent unavailable). Use existing `codebase-context.md` if present from design mode run. Produces or updates `docs/feature/{name}/codebase-context.md`.

### STEP 2/3 — Draft Implementation Plan

Spawn `ali-plan-agent` to write implementation plan at `docs/feature/{name}/plan.md` using the approved design doc and `codebase-context.md`.

If any referenced source file exceeds 500 lines, apply the large-file protocol before
the write step begins. See: skills/plan-builder/references/large-file-protocol.md

### STEP 3/3 — Iterate Until Clean (or Draft advisory)

**Default (iterate-until-clean):**

1. Spawn `ali-plan-reviewer` with the Implement Mode Clean Criteria (below) provided verbatim in the delegation prompt.
2. Reviewer writes verdict to `.tmp/{review-id}/reviewer-verdict-{N}.md`.
3. Reviewer returns: `REVIEWER_VERDICT: PASS | FAIL` + filename only.
4. CoS reads first line of verdict file only.
5. On PASS: proceed to User Approval.
6. On FAIL: spawn `ali-plan-agent` to revise `plan.md` — pass verdict file path only (not full findings). Increment cycle. Repeat from step 1.
7. Safety valve triggers after 5 FAIL cycles — see Safety Valve section.

**Draft mode (single-pass advisory):**
- `ali-plan-reviewer` runs once.
- Verdict is advisory. CoS skips the PASS/FAIL gate.
- Proceed to User Approval regardless of verdict.

### User Approval

Present `plan.md` summary to user. Ask via AskUserQuestion: Approve plan / Request changes / Cancel.

---

## Implement Mode Clean Criteria

These criteria are passed verbatim to the reviewer in the delegation prompt. All must be true for PASS.

1. **Single deliverable per phase:** (a) single coherent activity type per phase — code authoring, test execution, review, documentation, or configuration; combinations are not permitted; (b) each phase title is expressible in 2 sentences; (c) no phase title contains "and" unless "and" connects two words naming a single artifact (e.g., "setup and migration scripts" is one artifact; "implement auth and run tests" is two activity types and must be split)
2. **Verification criteria present:** each phase has a "Phase Complete When" checklist with >=2 measurable items
3. **Agent routing explicit:** each phase specifies model and rationale
4. **Rollback plan exists:** each phase has a Rollback statement (N/A acceptable for Phase 0)
5. **No dependency gaps:** each phase's pre-conditions are satisfiable from prior phases' outputs
6. **Phase 0 exists:** first phase is validation-only with no code changes; design doc's acceptance criteria appear in at least one phase's verification checklist
7. **Implementation-critical values cited:** Plan-agent must cite source files for all
   implementation-critical values; values that cannot be verified during planning must
   carry a VERIFY: flag with the source file name.
   Reviewer check: all implementation-critical values in the plan (column names, function
   signatures, API contracts, migration revision IDs, config keys, syntax blocks) include
   a source file citation; prose descriptions without citation are present for
   implementation-critical values only when accompanied by a VERIFY: flag;
   unverified values carry a VERIFY: flag
   (VERIFY-flagged values are WARNINGS, not BLOCKING)

---

## Safety Valve

After 5 FAIL cycles without PASS:

1. Surface to user: current draft path, latest verdict path (`.tmp/{review-id}/reviewer-verdict-5.md`), and the failing criteria.
2. Present options via AskUserQuestion:
   - **Continue:** grant N more cycles (user specifies count) — resume from current draft
   - **Redesign:** return to design mode with the blocker as the new brief
   - **Cancel:** abandon this run; artifacts remain in `docs/feature/{name}/` for reference

Plan-builder does NOT auto-approve on safety valve trigger. User decides.

---

## Open Question Escalation

If the reviewer's verdict file contains an OPEN_QUESTIONS entry:

1. Do NOT consume an iteration cycle for this.
2. Surface the open question to the user immediately via AskUserQuestion.
3. Wait for user answer before continuing iteration.
4. Resume from the same cycle count after user response.

---

## Agent Model Tiers

| Stage | Mode | Model | Rationale |
|-------|------|-------|-----------|
| CoS main session | Both | Sonnet | Human-validated in real-time via ISPR |
| Plan-builder orchestrator (ali-plan-builder) | Both | Sonnet | Coordination/routing, not domain reasoning |
| ali-document-developer (design doc drafter) | Design | Sonnet | Multi-section document drafting; human reviews output |
| Domain experts | Design | Opus | Autonomous domain analysis, no in-flight human check |
| ali-technical-feasibility-expert (design reviewer) | Design | Opus | PASS/FAIL quality gate, no human review mid-cycle |
| ali-merge-synthesis-expert | Design | Opus | Autonomous artifact quality — human approval is guidance, not verification |
| Codebase exploration agent | Implement | Haiku | File reading and indexing, no synthesis |
| ali-plan-agent | Implement | Sonnet | Multi-phase plan production; human reviews output |
| ali-plan-reviewer | Implement | Opus | PASS/FAIL quality gate, no human review mid-cycle |

**Model assignment rationale:**

Agents split into three tiers by oversight level:
- **Opus:** Agents that produce a quality gate artifact autonomously (PASS/FAIL, APPROVE/BLOCK) with no human check until the verdict affects workflow. Domain experts, reviewers, and the merge synthesizer.
- **Sonnet:** Agents that produce artifacts reviewed by humans before those artifacts gate downstream steps. Document drafters, plan agents.
- **Haiku:** Agents performing search/indexing with no synthesis requirement.

**Note on merge synthesizer:** ali-merge-synthesis-expert is assigned Opus. The synthesizer must produce artifact-quality output autonomously. Human approval at the User Approval step is guidance, not verification — the design minimizes reliance on human verification in favor of autonomous quality output.

**Enforcement note:** This matrix is advisory documentation for CoS. It describes the intended model for each agent in the plan-builder workflow. Implementation does NOT change agent frontmatter (model: lines in agent .md files). CoS should use these model designations when writing delegation prompts if the Task tool supports model override; otherwise the matrix documents design intent for future agent frontmatter updates.

---

## Iteration Loop: Key Principle

All iteration communication is file-based. CoS never passes full reviewer findings in context — only filenames. This keeps context bounded across the loop regardless of cycle count.

```
Writer produces draft → writes to file
    │
    ▼
Reviewer reads draft file
    │ → Writes verdict to: .tmp/{review-id}/reviewer-verdict-{N}.md
    │ → Returns to CoS: REVIEWER_VERDICT: PASS | FAIL + filename only
    │
CoS reads first line of verdict file
    │
    ├── PASS → Surface non-blocking warnings to user. Output artifact.
    │
    └── FAIL → Pass verdict file PATH to writer (not full findings)
                Writer reads verdict file, addresses blocking issues
                Writer updates draft file (not in-context)
                Loop repeats (max 5 cycles — safety valve at 5)
```

**Reviewer mandatory return format:**

```
REVIEWER_VERDICT: PASS | FAIL
File: .tmp/{review-id}/reviewer-verdict-{N}.md
```

---

## Integration with CoS Role

This skill is invoked by the main session acting as CoS. The workflow respects CoS constraints:

- **CoS determines mode** (reads input context, checks design doc status)
- **CoS spawns agents** (has Task tool)
- **Agents write to feature workspace or .tmp/** (have Write(docs/feature/**) and Write(.tmp/**))
- **Agents return filenames only** (keeps agent outputs out of CoS context)
- **CoS reads first line of verdict files** (bounded token cost per cycle)
- **CoS presents to user** (AskUserQuestion for approval gates)

**Parallel cap:** Respect the 5-task limit per message. Design mode expert review may spawn multiple domain experts in parallel — batch in groups of 5 if more than 5 experts are selected.

---

## Single-Approval Workflow

User approves once at workflow start (initial ISPR). CoS then runs all steps autonomously:

- Do NOT ask permission for each step
- Do NOT ask permission for file writes (paths are pre-approved)
- Do NOT ask permission for spawning intermediate agents
- Only pause at final approval step (User Approval for both modes)
- Pause for open question escalation when reviewer flags OPEN_QUESTIONS
- Pause for safety valve escalation after 5 FAIL cycles

---

**Version:** 3.2
**Created:** 2026-03-01
**Replaces:** 2.1 (single-mode expert-panel pipeline)
**Change:** Full two-mode design-first rewrite. Added design mode (feature brief → validated design doc), implement mode (approved design → implementation plan), iterate-until-clean loop with ali-plan-reviewer, safety valve at 5 cycles, draft mode advisory path, open question escalation. v3.1: Removed progress log — TaskList is the progress tracker. v3.2: Added Phase Sizing Criteria section, updated Implement Mode Clean Criterion 1 (activity-type split rule), replaced two-section Agent Model Tiers with unified model matrix (Mode column).
