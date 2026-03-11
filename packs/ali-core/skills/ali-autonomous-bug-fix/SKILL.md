---
name: ali-autonomous-bug-fix
description: |
  End-to-end autonomous bug fix process for tax-ai-chat. Use when:

  PLANNING: Deciding whether a reported bug qualifies for autonomous resolution
  vs. a scoped design-mode treatment first.

  IMPLEMENTATION: Running the full fix loop — investigate, design, implement,
  test, verify, then surface for deploy approval.

  GUIDANCE: Asking how the autonomous loop works, what artifacts it requires,
  or how to invoke it correctly.

  REVIEW: Auditing a completed autonomous fix for process compliance (journal
  completeness, E2E clean, deploy gate enforced).
---

# Autonomous Bug Fix

## When to Use This Process

Use this process when ALL of the following are true:

- Bug is reported with reproducible steps (screenshot, video, or clear description)
- Root cause is expected to be localized (1-3 files, not an architecture change)
- Fix can be developed and validated locally without cloud access
- Don has explicitly authorized autonomous end-to-end execution for the session

Do NOT use this process when:

- Root cause is unknown and may require architectural changes (use plan-builder design mode first)
- Fix requires database migrations on cloud (ISPR required before proceeding)
- Multiple unrelated systems are implicated (split into separate issues)
- Don has not granted autonomous session-level approval

---

## Hard Constraints (Non-Negotiable)

These constraints apply to every autonomous fix run regardless of session context:

1. **No cloud deploys without ISPR.** Local development only. Never assume deploy is authorized.
2. **Zero regressions required.** Full E2E suite must pass (minus pre-existing annotated failures) before declaring complete.
3. **plan-builder must rate IMPLEMENTATION READY.** If rating is below that threshold, stop and ISPR Don.
4. **Scope must not expand during implementation.** If a related issue is discovered, note it in the journal as a procedural gap. Do not fix it in the same branch unless it is directly in the prop/data thread of the current fix.
5. **Journal must be initialized before any agent spawns.** If the journal does not exist, create it first.

---

## Pre-Conditions Checklist

Before starting:

- [ ] Journal initialized at `.tmp/bug-fix-<YYYY-MM-DD>/journal.md`
- [ ] Bug description recorded in journal (user action, expected, actual)
- [ ] Assumptions recorded in journal
- [ ] Risks recorded in journal
- [ ] Branch strategy confirmed (separate branch, not working on main)
- [ ] Don's hard constraints confirmed (e.g., "no cloud", "no deploy")
- [ ] Session-level autonomous approval confirmed

---

## Process Steps

### Step 1 — Initialize Journal

Create `.tmp/bug-fix-<YYYY-MM-DD>/journal.md` using the journal format below.

Record:
- Bug ID (format: `BUG-YYYY-MM-DD-SHORT-SLUG`)
- Reporter
- Who initiated (CoS autonomous or Don direct)
- Pre-approval status and hard constraints
- Bug description (user action, expected behavior, actual behavior)
- Assumptions
- Risks

**Gate:** Journal must exist before any agent spawns.

---

### Step 2 — Parallel Expert Investigation

Spawn backend-expert and frontend-expert **in parallel** (foreground, per background-agent-guard constraint).

Delegation prompt for each expert:

```
Investigate bug BUG-YYYY-MM-DD-SHORT-SLUG.

Bug description: [paste from journal]

Your task:
1. Identify the root cause within your domain (backend or frontend)
2. Identify any related secondary gaps discovered during investigation
3. Write findings to: .tmp/bug-fix-<date>/[backend|frontend]-expert-findings.md

Findings must include:
- Root cause with file:line citations
- Proposed fix (high level — not implementation)
- Test gap (what test would have caught this)
- Relation to any open items from handoff

Output-Path: .tmp/bug-fix-<date>/[backend|frontend]-expert-findings.md
```

**Artifacts required at this step:**
- `.tmp/bug-fix-<date>/backend-expert-findings.md`
- `.tmp/bug-fix-<date>/frontend-expert-findings.md`

**Journal entry required:** Summarize expert consensus, root cause, and proposed fix direction.

**Gate:** Both expert findings must be present before submitting to plan-builder.

---

### Step 3 — Plan-Builder Review

Spawn plan-builder with expert findings as input.

Delegation prompt:

```
Review findings for bug BUG-YYYY-MM-DD-SHORT-SLUG and produce a design doc.

Expert findings:
- Backend: .tmp/bug-fix-<date>/backend-expert-findings.md
- Frontend: .tmp/bug-fix-<date>/frontend-expert-findings.md

Produce a design doc at: .tmp/bug-fix-<date>/design-doc.md

The design doc must include:
- RATING (IMPLEMENTATION READY or NOT READY — with reason)
- Root cause summary
- Proposed fix with specific file changes
- Key design decisions with rationale
- Risks and mitigations
- Any secondary gaps noted (do not include in fix scope)
```

**Artifacts required:**
- `.tmp/bug-fix-<date>/design-doc.md`
- Rating must be explicit: `IMPLEMENTATION READY` or `NOT READY`

**Journal entry required:** Record the rating and key design decisions.

**Hard gate:** If rating is NOT READY, stop. Do not proceed to implementation. ISPR Don with the design doc and the reason for the NOT READY rating.

---

### Step 4 — Create Feature Branch

```bash
git checkout -b fix/<short-slug>
```

Branch naming: `fix/<short-slug>` (e.g., `fix/analysis-chat-context`)

Branch from current main HEAD. Do not branch from another feature branch.

**Journal entry required:** Branch name and base commit hash.

---

### Step 5 — Implementation

Spawn ali-frontend-developer or ali-backend-developer (or both) per plan-builder's design doc.

Delegation prompt must include:
- Reference to design doc path
- File list with specific changes required
- TypeScript check requirement: `npx tsc --noEmit` must pass
- Instruction to keep all changed props optional (no breaking changes to call sites)

**Artifacts required:**
- All modified files committed to the fix branch
- Zero TypeScript errors reported
- No unintended scope expansion

**Journal entry required:** List of modified files, changes made per file, any discoveries during implementation.

**Scope gate:** If developer discovers a related gap during implementation, it must be evaluated. If the gap is directly in the same prop/data thread, it may be included. If it is a separate concern, it goes in the journal as a procedural gap — not in this PR.

---

### Step 6 — Unit and E2E Tests

Spawn ali-test-developer.

Delegation prompt must include:
- What the new test must verify (the specific user flow that was broken)
- Unit test conventions to follow (existing test files to match as a pattern)
- E2E spec requirements:
  - Cloud skip condition: `test.skip(!!process.env.CLOUD_URL, 'cloud-gap: ...')`
  - localStorage.clear() at start to avoid session key collision
  - Must use existing fixture patterns (not ad-hoc setup)

**Artifacts required:**
- Unit test file with all tests passing
- E2E spec file covering the primary regression path
- Test count reported (e.g., 11/11 unit pass, 3/3 E2E pass)

**Journal entry required:** Test file paths, test count, pass/fail result.

---

### Step 7 — Full E2E Suite

Run `make test-e2e` (not targeted spec — the full suite).

**Pass criteria:**
- Zero new failures vs. pre-existing annotated baseline
- New spec tests pass
- git diff confirms changed files have no intersection with any failing tests

**Failure triage protocol:**
1. Check if failure is in the pre-existing annotated list (FLOW-032, VR darwin drift, etc.)
2. Check if failing test file has any intersection with changed files via git diff
3. If no intersection: classify as pre-existing, record in journal
4. If intersection exists: this is a regression — stop, investigate, fix before proceeding

**Artifacts required:**
- E2E run output (pass/fail counts)
- Failure analysis table (test name, classification, action)
- Confirmation that branch regressions are zero

**Journal entry required:** Full E2E results, failure analysis, zero-regression confirmation.

**Hard gate:** If any regression is traced to changed files, stop. Do not proceed to commit. Fix the regression or ISPR Don.

---

### Step 8 — ISPR for Deploy Approval

After E2E passes, surface to Don for review and deploy decision.

ISPR block must include:
- Branch name and commit hashes
- What was fixed (user-facing description)
- Test evidence (unit count, E2E count, regression status)
- Procedural gaps discovered (for follow-up)
- Explicit deploy proposal (or explicit note that deploy is not being proposed)

**Never assume deploy is authorized.** Even with session-level autonomous approval for the fix loop, production deployment always requires explicit ISPR approval.

---

## Artifacts Summary

| Step | Artifact | Location |
|------|----------|----------|
| 1 | Bug journal | `.tmp/bug-fix-<date>/journal.md` |
| 2 | Backend findings | `.tmp/bug-fix-<date>/backend-expert-findings.md` |
| 2 | Frontend findings | `.tmp/bug-fix-<date>/frontend-expert-findings.md` |
| 3 | Design doc | `.tmp/bug-fix-<date>/design-doc.md` |
| 4 | Feature branch | `fix/<short-slug>` |
| 5 | Code changes | Modified source files |
| 6 | Unit tests | `frontend/src/features/.../ComponentName.test.tsx` |
| 6 | E2E spec | `frontend/e2e/<feature-name>.spec.ts` |
| 7 | E2E results | Journal entry (not a separate file) |

---

## Journal Format

```markdown
# Bug Fix Journal — YYYY-MM-DD

**Bug ID:** BUG-YYYY-MM-DD-SHORT-SLUG
**Reporter:** [name]
**Initiated by:** [CoS autonomous / Don direct]
**Pre-approval:** [status]
**Hard constraint:** [e.g., NO cloud. NO deploy. Local development only.]

---

## Bug Description

**User action:**
1. [step 1]
2. [step 2]

**Expected behavior:**
[what should happen]

**Actual behavior:**
[what actually happens]

---

## Assumptions

1. [assumption 1]
2. [assumption 2]

---

## Risks

- [risk 1]
- [risk 2]

---

## Process Log

### Entry 1 — YYYY-MM-DD HH:MM
[status update]
Next: [what happens next]

### Entry 2 — YYYY-MM-DD (Expert Investigation Complete)
[summary of findings]

### Entry 3 — YYYY-MM-DD (Plan-Builder: RATING)
[design doc summary, key decisions]

### Entry 4 — YYYY-MM-DD (Implementation Complete)
[files modified, changes]

### Entry 5 — YYYY-MM-DD (Tests Written)
[test files, counts]

### Entry 6 — YYYY-MM-DD (E2E Results)
[pass/fail counts, failure analysis, zero-regression confirmation]

---

## Decisions Log

| # | Decision | Rationale | Made By |
|---|----------|-----------|---------|
| 1 | [decision] | [why] | [who] |

---

## Procedural Gaps

1. [gap 1 — what it is, recommendation, not blocking this PR]
2. [gap 2]

---

## Final Status

**COMPLETED AND READY FOR DON TO REVIEW**

**Branch:** [branch name]
**Commits:**
- [hash] — [message]

**What was fixed:** [user-facing description]

**Tests:** [N/N unit pass. N/N E2E regression pass. Zero regressions from our changes.]

**Not pushed.** Branch is local and ready for Don to review.
```

---

## Hard Gates Summary

| Gate | Condition | Action if Failed |
|------|-----------|-----------------|
| Journal initialized | Before any agent spawn | Create journal first |
| Expert findings present | Before plan-builder | Wait for both findings |
| Plan-builder IMPLEMENTATION READY | Before implementation | Stop; ISPR Don |
| Zero TypeScript errors | After implementation | Fix before tests |
| Zero regressions | After full E2E | Fix or ISPR Don |
| ISPR approval | Before any deploy | Never skip; always ISPR |

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Skipping plan-builder | May implement wrong fix or miss scope issues | Always rate before implementing |
| Targeted E2E only (not full suite) | Misses regressions in adjacent features | Always run make test-e2e |
| Fixing related gaps without journal entry | Creates invisible scope expansion | Note as procedural gap, separate PR |
| Assuming deploy is authorized | Production changes require explicit approval | Always ISPR for deploy |
| Branching from feature branch | Creates dependency chain, harder to merge | Always branch from main |
| Background agent spawn | ali-background-agent-guard hook blocks it | Spawn foreground; parallel still works |

---

## First Run Reference

The first run of this process (session 29, 2026-03-05) fixed BUG-2026-03-05-CHAT-CONTEXT — the analysis chat bubble creating sessions with null tax_return_id.

Journal: `.tmp/bug-fix-2026-03-05/journal.md`

Key outcomes from first run:
- Backend + frontend parallel investigation confirmed bug was 100% frontend
- plan-builder rated IMPLEMENTATION READY in one pass
- 4 files modified, all props optional, zero breaking changes
- 11/11 unit pass, 3/3 E2E regression pass, zero new failures in full suite
- Procedural gaps noted (documentHandling:399 fragility, session-client-isolation scope) — carried forward separately

---

**Version:** 1.0
**Created:** 2026-03-06
**Author:** Document Developer (from session 29 journal, first autonomous run)
