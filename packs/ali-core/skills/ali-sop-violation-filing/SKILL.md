---
name: ali-sop-violation-filing
description: |
  Process for quickly filing Standard Operating Procedure (SOP) violations. Use when:

  PLANNING: Designing workflows involving agent compliance or SOP enforcement

  IMPLEMENTATION: Agent or person identifies an SOP violation during work and needs to document it

  GUIDANCE: Asking when/how to file violations, what constitutes a violation, filing thresholds

  REVIEW: Reviewing filed violations for patterns, completeness, or severity assessment
---

# SOP Violation Filing

## When I Speak Up

This skill activates when:

**Planning/Design:**
- Designing compliance workflows or violation tracking systems
- Architecting SOP enforcement mechanisms
- Planning process improvement initiatives based on violations

**Implementation:**
- Agent or person detects an SOP violation during work
- Need to document a violation quickly without derailing current work
- Filing a violation report to the backlog

**Guidance/Best Practices:**
- Asking when to file a violation vs. when to just correct and move on
- Asking what information to capture
- Clarifying severity levels or root cause analysis depth
- Understanding the backlog pattern for violations

**Review/Validation:**
- Reviewing filed violations for completeness
- Analyzing violation patterns across multiple reports
- Validating severity assessments
- Checking if corrective actions are appropriate

---

## Key Principles

**Backlog pattern - Capture quickly and move on:**
- Filing should take ~5 minutes maximum
- This is NOT an investigation phase - that happens separately
- Document what you know, then return to your work
- Like adding a bug to a backlog - enough to act on later, not solve now

**Theories welcome, research later:**
- Note hypotheses about root causes
- Do NOT investigate during filing
- Deep analysis happens in a separate session

**Speed over completeness:**
- Better to file quickly with basic info than delay the session
- Can always update the issue later with more details
- The goal is to not lose the observation, not to solve it now

**Capture artifacts for later investigation:**
- Log entries, error messages (paste snippets)
- Related file paths
- Agent IDs or task IDs
- Anything that would help someone do deep-dive analysis later

---

## Filing Process

### Step 1: Quick Assessment (30 seconds)

**Is this worth filing?**

| File It | Don't File (Just Fix) |
|---------|----------------------|
| Pattern that keeps recurring | One-time typo or obvious mistake |
| Affects multiple agents/people | Personal slip-up, already internalized |
| Process gap or unclear SOP | Clear violation, trivial to correct |
| Causes inefficiency or errors | Cosmetic issue with no impact |
| Could teach others | Obvious in hindsight |

**When in doubt, file it.** Better to have the data than lose the observation.

### Step 2: Create GitHub Issue (2 minutes)

**Use the gh CLI to file violations to the ali-ai repository:**

> **IMPORTANT:** All SOP violations are filed to the ali-ai repo, not individual project repos.
> Always use the `-R dmccarty-ali/ali-ai` flag to ensure correct targeting.

```bash
# Interactive form-based filing (recommended)
gh issue create -R dmccarty-ali/ali-ai --template sop-violation.yml

# Or specify fields directly
gh issue create -R dmccarty-ali/ali-ai \
  --title "SOP: Brief description of violation" \
  --label "sop-violation" \
  --label "severity:medium" \
  --body "$(cat <<'EOF'
### Severity

Medium

### Project

project-name

### Agent/Person

Chief of Staff

### Summary

One-sentence description of what happened

### What Happened

1. Event 1
2. Event 2
3. Event 3

### What the Standard Requires

**Incorrect (what was done):**
What actually happened

**Correct pattern:**
What should have been done

### Impact

- **Actual:** What impact occurred
- **Potential:** What could happen if pattern continues
EOF
)"
```

**For agents:** Use the Bash tool to run `gh issue create` with the appropriate fields.

### Step 3: Fill in Required Fields (2 minutes)

- **Title:** "SOP: " prefix + brief description
- **Severity:** Low/Medium/High/Critical (auto-labels applied)
- **Project:** Which project it occurred in
- **Agent/Person:** Who made the violation
- **Summary:** One sentence describing what happened
- **What Happened:** Brief timeline or description
- **What the Standard Requires:** Incorrect vs. correct pattern
- **Impact:** Actual and potential impact

**Optional fields (skip if pressed for time):**
- Root Cause Theories (hypotheses only, not research)
- Artifacts (error messages, IDs, file paths)

### Step 4: Submit and Move On (1 minute)

- Issue created in ali-ai repo
- Severity label auto-applied via GitHub Action
- Return to your work
- Investigation and pattern analysis happen later

---

## Severity Guide

Use this quick guide to assess severity:

| Severity | Criteria | Examples |
|----------|----------|----------|
| **Critical** | Data loss, security breach, compliance violation, system unavailable | Committed secrets, deleted production data, violated HIPAA |
| **High** | Repeated pattern causing significant rework, blocks progress, affects multiple people | Recurring API failures, broken delegation pattern, architectural violation |
| **Medium** | Causes inefficiency or delays, could lead to errors, affects one person/agent | Skipped approval gate, incorrect pattern usage, unclear SOP interpretation |
| **Low** | Minor issue, recovered quickly, no lasting impact, one-time occurrence | Attempted resume that failed harmlessly, typo in config that was caught |

**When between two levels, default to lower.** Severity can be adjusted during pattern analysis.

### Severity Decision Tree

1. Is data lost, security breached, or compliance violated? → **Critical**
2. Is this a repeated pattern causing significant issues? → **High**
3. Does it cause inefficiency or could lead to errors? → **Medium**
4. Minor issue, recovered quickly, one-time? → **Low**

---

## Searching and Filtering Violations

GitHub Issues provides built-in search and filtering:

```bash
# All commands target ali-ai repo explicitly with -R flag

# View all open violations
gh issue list -R dmccarty-ali/ali-ai --label "sop-violation"

# Filter by severity
gh issue list -R dmccarty-ali/ali-ai --label "sop-violation" --label "severity:high"
gh issue list -R dmccarty-ali/ali-ai --label "sop-violation" --label "severity:critical"

# Search by project (searches title and body)
gh issue list -R dmccarty-ali/ali-ai --label "sop-violation" --search "Project: tax-ai-chat"

# Search by agent/person
gh issue list -R dmccarty-ali/ali-ai --label "sop-violation" --search "Agent/Person: Chief of Staff"

# View closed violations (for pattern analysis)
gh issue list -R dmccarty-ali/ali-ai --label "sop-violation" --state closed

# View all violations including closed
gh issue list -R dmccarty-ali/ali-ai --label "sop-violation" --state all

# Web interface with filters
gh issue list -R dmccarty-ali/ali-ai --label "sop-violation" --web
```

---

## When NOT to File

**Don't file violations for:**

| Scenario | Why Not | What to Do Instead |
|----------|---------|-------------------|
| Trivial typos or formatting | Not a process issue | Just fix it |
| One-time obvious mistakes | No pattern to track | Learn and move on |
| Issues already well-documented | Adds no new info | Reference existing docs |
| Personal learning moments | Not organizational issue | Update personal notes |
| Experimental approaches that didn't work | Not a violation | Note as "tried X, didn't work" |

**If it feels like bureaucracy, it probably is.** This system exists to capture patterns worth tracking, not to punish mistakes.

---

## Pattern Analysis (Separate Session)

**Filing is NOT investigation.** Pattern analysis happens separately:

1. **Pattern identification**: Review violation issues for recurring themes
2. **Root cause investigation**: Deep dive into why patterns exist (technical, process, clarity gaps)
3. **SOP updates**: Propose documentation clarifications or process changes
4. **Training**: Identify areas where agents or people need additional guidance

**Tools for pattern analysis:**
- GitHub issue search and filters
- Label-based aggregation (by severity, by pattern tags)
- Timeline analysis using issue dates
- Correlation with other events (deployments, team changes)

---

## Labels

The following labels are used for SOP violations:

| Label | Color | Purpose |
|-------|-------|---------|
| `sop-violation` | Red (#d73a4a) | All SOP violations have this label |
| `severity:low` | Light blue (#c5def5) | Low severity - minor, one-time issues |
| `severity:medium` | Yellow (#fbca04) | Medium severity - causes inefficiency |
| `severity:high` | Orange (#ff7518) | High severity - repeated patterns, blocks progress |
| `severity:critical` | Dark red (#b60205) | Critical - data loss, security, compliance |

**Severity labels are auto-applied** based on the dropdown selection in the issue form.

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Filing takes >5 minutes | Derails current work, becomes a tangent | Fill in basics only, add details later |
| Investigating root cause during filing | Turns quick capture into deep dive | Note theories only, investigate separately |
| Filing trivial one-time issues | Creates noise, hides real patterns | Use judgment - file patterns, not typos |
| Waiting until end of day to file | Lose context and details | File immediately when observed (5 min max) |
| No artifacts captured | Makes later investigation difficult | Paste error messages, IDs, file paths while fresh |

---

## Quick Reference

### Filing Checklist

- [ ] Title starts with "SOP: "
- [ ] Severity selected (auto-labels applied)
- [ ] Project identified
- [ ] Agent/Person identified
- [ ] Summary is one clear sentence
- [ ] "What happened" describes events
- [ ] "What the standard requires" shows incorrect vs. correct pattern
- [ ] Impact (actual and potential) noted
- [ ] Artifacts captured if available (error messages, IDs, paths)
- [ ] No investigation or root cause research performed (theories only)
- [ ] Time spent: ≤5 minutes

### Common Commands

```bash
# All commands target ali-ai repo explicitly with -R flag

# File new violation (interactive form)
gh issue create -R dmccarty-ali/ali-ai --template sop-violation.yml

# List all open violations
gh issue list -R dmccarty-ali/ali-ai --label "sop-violation"

# List high/critical violations
gh issue list -R dmccarty-ali/ali-ai --label "sop-violation" --label "severity:high"
gh issue list -R dmccarty-ali/ali-ai --label "sop-violation" --label "severity:critical"

# Search violations by keyword
gh issue list -R dmccarty-ali/ali-ai --label "sop-violation" --search "keyword"

# View violation in browser
gh issue view -R dmccarty-ali/ali-ai 123 --web

# Close resolved violation
gh issue close -R dmccarty-ali/ali-ai 123 --comment "Resolved: SOP updated to clarify X"
```

---

## Your Codebase

### File Locations

| Purpose | Location |
|---------|----------|
| Issue template | ~/ali-ai/.github/ISSUE_TEMPLATE/sop-violation.yml |
| Auto-label workflow | ~/ali-ai/.github/workflows/sop-violation-labels.yml |
| This skill | ~/ali-ai/skills/ali-sop-violation-filing/SKILL.md |
| Legacy violations (archived) | ~/.claude/docs/archive/sop-violations-legacy/ |

### Tracking

All violations are tracked as GitHub Issues in the ali-ai repository:
- URL: https://github.com/dmccarty-ali/ali-ai/issues
- Filter: `label:sop-violation`

---

## References

- ~/.claude/docs/claude-aliunde.md: Organization-wide SOPs and compliance requirements
- ~/ali-ai/roles/Chief-of-Staff-Role.md: Agent delegation patterns

---

**Document Version:** 2.0
**Last Updated:** 2026-01-26
**Maintained By:** Don McCarty
