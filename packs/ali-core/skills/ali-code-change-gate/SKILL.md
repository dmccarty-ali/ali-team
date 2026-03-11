---
name: ali-code-change-gate
description: |
  Mandatory checkpoint for all code changes. Use BEFORE writing any code to verify
  compliance with documented patterns and standards. This is a gate-keeping skill
  that enforces design rules and prevents pattern violations.

  PLANNING: Check before proposing any code implementation approach

  IMPLEMENTATION: MANDATORY checkpoint before writing any code - verify all rules

  GUIDANCE: Reference when unsure about design rules or change protocol

  REVIEW: Verify that code changes followed the mandatory protocol
---

# Code Change Gate

**CRITICAL: This skill MUST be applied before writing any code.**

This is a mandatory checkpoint that ensures all code changes follow documented patterns and the pre-code change protocol.

---

## When This Skill Activates

This skill MUST be used:

- **BEFORE any code change** - No exceptions, even for "simple" changes
- **Before proposing a solution** - Verify you've read existing code
- **Before implementing a fix** - Ensure you understand existing patterns
- **After identifying issues** - Before writing code to fix them

---

## Mandatory Pre-Code Checklist

Before writing ANY code, verify ALL items:

### 0. Delegation Check (First Priority)

**Check if a specialist skill should handle this change:**

- [ ] Checked if changes involve test files → delegate to /test-developer
- [ ] Checked if changes involve frontend components → delegate to /frontend-developer
- [ ] Checked if changes involve backend services → delegate to /backend-developer (when created)
- [ ] If specialist skill exists for this domain, STOP and invoke that skill instead
- [ ] Only proceed with general changes if no specialist skill applies

**File Pattern Triggers:**

| Pattern | Skill | Examples |
|---------|-------|----------|
| tests/**, *.feature, *.steps.ts | /test-developer | tests/unit/, frontend/tests/, features/chapter-1/upload.feature |
| conftest.py, playwright.config.ts | /test-developer | Test configuration files |
| frontend/apps/**/*.tsx | /frontend-developer | React components, pages |
| frontend/packages/ui/** | /frontend-developer | Shared UI library components |
| services/**, dags/** | /backend-developer | API services, Airflow DAGs (when skill exists) |

**Why Delegation Matters:**
- Specialist skills have domain-specific expertise and checklists
- test-developer enforces data-test-id requirements
- frontend-developer enforces accessibility and UX patterns
- Prevents violating domain-specific rules that general code review might miss

### 1. Reading and Research
- [ ] Read the specific files you plan to modify
- [ ] Checked relevant ARCHITECTURE.md for project-specific patterns
- [ ] Reviewed @skills/design-patterns for applicable standards
- [ ] Reviewed @skills/clean-code for coding standards
- [ ] Verified existing patterns in the codebase (not guessed)

### 2. Pattern Compliance Verification
- [ ] Checked for duplicate code - will extract to shared function if found
- [ ] Verified no hardcoded strings - will use enums/constants instead
- [ ] Confirmed no duplicate schema definitions - single source of truth
- [ ] Ensured no magic numbers - will use named constants
- [ ] For tests: confirmed data-test-id attributes will be included

### 3. Protocol Compliance
- [ ] IDENTIFIED issues and existing patterns
- [ ] SUMMARIZED findings for Don
- [ ] PROPOSED solution following documented patterns
- [ ] REQUESTED approval before writing code (unless in Delegated Execution Mode - see Step 6)
- [ ] Did NOT say "I fixed it" or present completed code (unless in Delegated Execution Mode - see Step 6)

---

## Non-Negotiable Design Rules

These rules MUST be followed in ALL code changes:

### DRY (Don't Repeat Yourself)
**Rule:** NO duplicate code across the codebase
**Action:** Extract duplicate code to shared functions, utilities, or base classes

**Bad:**
```python
# File A
result = data.strip().lower().replace(" ", "_")

# File B
clean_data = data.strip().lower().replace(" ", "_")
```

**Good:**
```python
# shared/utils.py
def normalize_identifier(data: str) -> str:
    """Normalize string to identifier format."""
    return data.strip().lower().replace(" ", "_")

# File A and File B
result = normalize_identifier(data)
```

### No Hardcoded Strings
**Rule:** Use enums, constants, or configuration - never hardcoded strings
**Action:** Define enums for string values used in logic

**Bad:**
```python
if status == "pending":
    process_pending()
elif status == "approved":
    process_approved()
```

**Good:**
```python
from enum import Enum

class DocumentStatus(Enum):
    PENDING = "pending"
    APPROVED = "approved"
    REJECTED = "rejected"

if status == DocumentStatus.PENDING.value:
    process_pending()
elif status == DocumentStatus.APPROVED.value:
    process_approved()
```

### No Duplicate Schema Definitions
**Rule:** Single source of truth for all data schemas
**Action:** Define schema once, import everywhere

**Bad:**
```python
# File A
client_schema = {
    "id": str,
    "name": str,
    "email": str
}

# File B
client_fields = {
    "id": str,
    "name": str,
    "email": str
}
```

**Good:**
```python
# models/client.py
from dataclasses import dataclass

@dataclass
class Client:
    id: str
    name: str
    email: str

# File A and File B
from models.client import Client
```

### No Magic Numbers
**Rule:** All numeric literals must be named constants (except 0, 1, -1 in obvious contexts)
**Action:** Define constants with descriptive names

**Bad:**
```python
if retry_count > 3:
    raise MaxRetriesExceeded()
time.sleep(5)
```

**Good:**
```python
MAX_RETRY_ATTEMPTS = 3
RETRY_DELAY_SECONDS = 5

if retry_count > MAX_RETRY_ATTEMPTS:
    raise MaxRetriesExceeded()
time.sleep(RETRY_DELAY_SECONDS)
```

### Always Use data-test-id
**Rule:** All test element selectors must use data-test-id attributes
**Action:** Add data-test-id to elements and use in test selectors

**Bad:**
```typescript
// Test file
await page.click('.submit-button');
await page.getByRole('button', { name: 'Submit' }).click();
```

**Good:**
```typescript
// Component
<button data-test-id="submit-button">Submit</button>

// Test file
await page.click('[data-test-id="submit-button"]');
```

---

## Protocol Workflow

### Step 1: STOP
- Do not write code immediately
- Even if the change seems trivial, follow the protocol
- Resist the urge to "just fix it quickly"

### Step 2: READ
Read and verify:
- The specific files you plan to modify
- Related files that use similar patterns
- ARCHITECTURE.md for project-specific conventions
- @skills/design-patterns for standards
- @skills/clean-code for coding practices

### Step 3: IDENTIFY
Document what you found:
- What issues exist in the current code
- What patterns are already in use
- What files will be affected
- What design rules apply

### Step 4: SUMMARIZE
Present your findings to Don:
- "I found X issues in files Y, Z"
- "The existing pattern for this is [pattern name]"
- "Current code violates [rule], I will fix by [approach]"
- List all affected files

### Step 5: PROPOSE
Describe your solution:
- "Here's my proposed solution following [pattern/standard]"
- Explain how it adheres to documented patterns
- Note any design pattern violations you'll fix
- Reference specific rules from this skill

### Step 6: REQUEST APPROVAL
**First, check for Delegated Execution Mode:**

If your task prompt contains the heading "## DELEGATED EXECUTION MODE" (case-insensitive substring match), then:
- **Skip the approval request** - Don's ISPR approval of the orchestrator is your authorization to execute
- **Proceed directly to implementation** after Step 5
- **Continue to escalate** if you encounter scope expansion, blocking issues, ambiguity, or design pattern violations (see Exception section below)

Otherwise (standard execution mode):
- **Wait for explicit approval** before writing code
- Ask: "Should I proceed with this approach?"
- **NEVER say "I fixed it"** or present completed code without prior approval

### Exception: Delegated Execution Mode

**When an agent's prompt contains the heading "## DELEGATED EXECUTION MODE", the REQUEST APPROVAL step is modified.**

**Recognition pattern:** Case-insensitive substring match for "## DELEGATED EXECUTION MODE" heading in the agent's task prompt.

**Modified workflow:**
1. Agent MUST still follow Steps 2-5 (READ, IDENTIFY, SUMMARIZE, PROPOSE)
2. After PROPOSE, agent proceeds directly to implementation
3. Agent does NOT wait for additional approval from orchestrator
4. Don's approval of the orchestrator's ISPR delegation is the agent's authority to execute

**This exception explicitly OVERRIDES the "NEVER present code without approval" language at lines 260-263 above.**

**Agent MUST escalate if encountering:**
1. **Scope expansion** - Task is larger than originally described
2. **Blocking issue** - Technical blocker requiring decision
3. **Ambiguity** - Unclear requirement that could lead to wrong outcome
4. **Design pattern violation** - Proposed solution would violate DRY, hardcoded strings, magic numbers, or duplicate schemas

**Escalation format:**
```
ESCALATION REQUIRED

**Issue:** [scope expansion / blocking issue / ambiguity / design violation]

**Context:** [what was encountered]

**Options:** [possible paths forward]

**Recommendation:** [agent's recommendation with rationale]

**Request:** Approval to proceed with recommended approach or alternative guidance
```

**Cross-reference:** See ~/.claude/docs/claude-aliunde.md "Agent Delegation Requirements" section for complete Delegated Execution Mode definition and which agents receive it.

---

## Post-Implementation Verification Checklist

After writing code, verify ALL items before reporting complete:

### 1. Test Execution
- [ ] Ran existing tests in affected area
- [ ] All tests pass (or failures are pre-existing and documented)
- [ ] Added new test if this was a bug fix (regression test)

### 2. Problem Verification
- [ ] Re-read original problem statement
- [ ] Change directly addresses the stated problem
- [ ] Can demonstrate it works (not just that it compiles)

### 3. Regression Check
- [ ] No unintended changes to other functionality
- [ ] Imports/dependencies still resolve
- [ ] Linting passes (if applicable)

### 4. Completion Evidence
- [ ] Verification section included in output
- [ ] Test results shown (not just "tests pass")
- [ ] Method of verification documented

**Work without verification evidence is incomplete.**

---

## Red Flags - Stop Immediately If You Notice These

| Red Flag | Why It's Wrong | What To Do |
|----------|----------------|------------|
| "I fixed it" (when NOT in Delegated Execution Mode) | Code written without approval | Back out, follow protocol |
| "I found 8 issues and fixed them" (when NOT in Delegated Execution Mode) | Violated protocol entirely | Explain what you WOULD do, get approval |
| Hardcoded "pending", "approved" strings | Should use enum | Extract to enum, propose change |
| Duplicate validation logic | Violates DRY | Extract to shared function |
| Magic number 3, 5, 30 | Should be named constant | Define constants with clear names |
| test clicking `.button-class` | Should use data-test-id | Update component and test |
| Same schema in 3 files | Violates single source of truth | Create shared model |
| "The pattern is probably X" | Guessing without reading | Read the code first |

---

## Examples

### Example 1: Bad - Violates Protocol

```
User: "Fix the validation bug"
Assistant: "I found the issue and fixed it. The validation was checking for
'active' but should check for 'enabled'. Updated 3 files."
```

**Problems:**
- Wrote code without approval
- Hardcoded strings not replaced with enum
- Didn't summarize findings first
- Didn't propose approach

### Example 2: Good - Follows Protocol

```
User: "Fix the validation bug"
Assistant: "Let me investigate the validation code first."

[Reads validation files: validators/user.py, validators/common.py]
Assistant: "I found the validation issue in validators/user.py:

SUMMARY:
- Line 45: Hardcoded string "active" used in status check
- Line 52: Duplicate validation logic also in validators/common.py:78
- No enum defined for status values
- Violates Non-Negotiable Rules: No hardcoded strings, DRY principle

PROPOSED SOLUTION:
1. Create StatusEnum in models/enums.py with values: ACTIVE, ENABLED, DISABLED
2. Extract duplicate validation logic to shared function in validators/common.py
3. Update both validators to use the enum and shared function

This follows the design-patterns skill (Factory pattern for enum) and
clean-code skill (no magic strings).

Should I proceed with this approach?"
```

**Why This is Good:**
- Followed STOP-READ-IDENTIFY-SUMMARIZE-PROPOSE-REQUEST protocol
- Identified hardcoded string violation
- Identified DRY violation
- Proposed enum-based solution
- Referenced specific skills and patterns
- Requested approval before writing code

---

## Enforcement

### For Claude Code Assistants

When you see a code change request:

1. **Immediately invoke this skill** before doing anything else
2. **Follow the checklist** - all items must be checked
3. **Stop if you catch yourself**:
   - Writing code without reading existing patterns
   - Saying "I fixed X" (unless in Delegated Execution Mode)
   - Presenting completed code without approval (unless in Delegated Execution Mode)
   - Using hardcoded strings
   - Duplicating code
4. **Request approval before writing code** (unless in Delegated Execution Mode - see Exception section)

### For Expert Agents

When reviewing code changes:

1. **Check protocol compliance** - was the protocol followed?
2. **Flag violations as CRITICAL** - not recommendations
3. **Reference this skill** - cite specific rules violated
4. **Block the change** - violations must be fixed

---

## Summary

This skill is the enforcement mechanism for Don's documented patterns and protocols. It is:

- **Mandatory** - Must be used before ANY code change
- **Comprehensive** - Covers all non-negotiable design rules
- **Blocking** - Violations prevent code from being accepted
- **Referenced by experts** - Expert agents cite this skill when reviewing

Success = Following the protocol, adhering to design rules, getting approval before coding.
