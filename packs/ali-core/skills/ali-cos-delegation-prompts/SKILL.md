---
name: ali-cos-delegation-prompts
description: |
  Chief of Staff delegation prompt reference. Use when:

  PLANNING: Designing delegation strategy, decomposing work into subtasks,
  deciding what context sub-agents will need, planning agent spawn sequence

  IMPLEMENTATION: Writing delegation prompts, constructing Task tool calls,
  spawning agents, elaborating brief user requests into complete context packages

  GUIDANCE: Asking how to write a good delegation prompt, what goes in a system
  prompt override, how to pass context to sub-agents, what the handshake is for

  REVIEW: Reviewing a Task tool call for completeness, checking a delegation
  prompt for missing context, auditing agent spawn patterns for correctness
---

# CoS Delegation Prompts

Reference material for writing complete, correct delegation prompts when spawning sub-agents.

---

## Universal Spawn Pattern

**CoS defaults to foreground subtasks. Background spawn is available for long-running tasks but requires explicit output routing in the delegation prompt. The write-to-file, return-filename pattern described here applies to all substantive delegations regardless of execution mode.**

When spawning an agent in foreground, use the Task tool without the run_in_background parameter. The agent runs to completion, writes results to a .tmp/ file, and returns the filename only. CoS context receives the filename — not the full content.

**The foreground pattern:**

1. CoS spawns agent (foreground, no run_in_background)
2. Agent does its work
3. Agent writes complete findings to .tmp/{descriptive-path}.md
4. Agent returns to CoS: "Findings written to .tmp/{path}" plus a brief summary (5-10 lines)
5. CoS receives the brief summary — not the full document

**What CoS does with the filename:**

- Records it in the todo list: "security-expert findings at .tmp/security-review-20260117/security-expert.md"
- Reports the path to Don so he can read findings if desired
- Passes the path (not the content) to subsequent agents that need to reference prior findings

**Never do this in foreground agents:**

Do not ask foreground agents to return their full findings inline. Do not paste large agent outputs into the CoS todo list. Keep CoS context lean.

---

## Artifact Output Pattern

**Applies to:** Agents doing substantive work that produces artifacts — code changes, reviews, documents, analysis, plans, or any output with more than a few lines of content.

**Does not apply to:** Conversational or routing agents returning a brief inline answer (e.g., "the file is at X", "use agent Y for this task"). These agents may return inline.

**The pattern:**

- Agent writes complete output to: `.tmp/{descriptive-slug}/{agent-name}-{yyyymmdd_hhmmss}.md`
- Agent returns to CoS: filename + exec summary (5-10 lines max)
- CoS records the path; does not paste full content into conversation

**Naming convention:**

```
.tmp/{descriptive-slug}/{agent-name}-{yyyymmdd_hhmmss}.md

Examples:
  .tmp/security-review-auth-20260124/security-expert-20260124_143022.md
  .tmp/plan-builder-improvements-20260302/phase-2-output-20260302_091500.md
  .tmp/rbac-plan-review-20260125/claude-developer-20260125_160345.md
```

**For background agents:** The same naming convention applies, but additional return format constraints apply because background agent responses are truncated at 30K characters. See the Background Agents section below for the mandatory CRITICAL - RETURN FORMAT block required in background delegation prompts.

---

## Background Agents

**Background agents are for tasks expected to take more than 30 seconds.** CoS stays responsive while the agent works in parallel.

### What Background Means (and Doesn't Mean)

`run_in_background` controls **execution concurrency**, not tool access. Background agents have full tool access including Write, Read, Grep, Bash, and all other tools. They can and do write files to disk.

**Correct understanding:**
- Foreground: CoS blocks and waits for agent to finish
- Background: CoS continues working while agent runs in parallel

**Incorrect understanding:**
- Background does NOT restrict Write or other tools
- Background does NOT force output into CoS context

### Mandatory Return Format for Background Agents

Background agent responses are truncated at 30K characters when returned to the parent. For a 500-line findings document, this truncation makes inline return unreliable. File-based output is mandatory, not optional.

**Include this block in every background agent delegation prompt:**

```
CRITICAL - RETURN FORMAT:
Write your full output to: [specified file path, e.g., .tmp/{uuid}/{name}.md]
Your response back must contain ONLY:
- A 1-2 sentence completion summary
- The file path where output was written
DO NOT return full content in your response.
```

**Example of what a background agent should return:**

```
Security review complete. Found 2 critical issues and 4 warnings.
Output: .tmp/security-review-auth-20260124/security-expert.md
```

**Not this (will be truncated, content may be lost):**

```
[500 lines of findings returned inline...]
```

### Checking Background Agent Output

After a background agent completes, CoS reads the output file directly:

```
Read: .tmp/security-review-auth-20260124/security-expert.md
```

Report the path to Don and record it in the todo list.

### Background Agent Limitations

- Cannot prompt for permission mid-execution — they rely on pre-approved permissions
- Output returned to parent is truncated at 30K chars (which is why file-based output is mandatory)
- No handshake confirmation is visible to CoS in real time (CoS does not block waiting for it)

### When to Use Background vs Foreground

| Scenario | Use |
|----------|-----|
| Task expected to take < 30 seconds | Foreground |
| Task expected to take > 30 seconds | Background |
| Multiple independent agents running in parallel | Background (up to 5 simultaneous — see Chief-of-Staff-Core.md for hard constraint) |
| Agent needs CoS to review output before next step | Foreground |
| Long-running research or large codebase analysis | Background |

---

## Foreground Requirement for Bash Agents

**Any agent that will execute Bash commands MUST be spawned in foreground.**

### Why Background Bash Fails

Background agents cannot reliably execute Bash commands for two reasons:

**Permission prompts auto-deny.** When an agent runs in background, it cannot surface permission prompts to the user. Any Bash command requiring approval is silently denied. The agent either fails immediately or loops retrying without ever receiving permission.

**Guard hooks block with interactive instructions.** Hooks like ali-git-guard.sh and ali-aws-guard.sh exit 2 with slash-command instructions (e.g., "run /github-admin to perform git operations"). A background agent cannot execute slash commands — it has no interactive session to dispatch them through. This causes one of two failure modes:
- Instant denial (~13 seconds): Agent receives the block, cannot follow the instruction, fails
- Retry loop (up to 45-minute timeout): Agent retries the command repeatedly, never making progress

### Which Agents This Affects

Any agent with Bash in its tool list that will actually execute bash commands:

- Admin agents (github-admin, aws-admin)
- Developer agents running tests, builds, or scripts
- Any agent running linters, formatters, or CLI tools
- Any agent performing git operations

Review-only agents that use Read/Grep/Glob but not Bash are unaffected — they can run background safely.

### The Pattern

**Always foreground for Bash-capable agents:**

```
# CORRECT — foreground (default, no run_in_background)
Task: github-admin
Prompt: Stage and commit the changes...

# WRONG — background with Bash execution
Task: github-admin
run_in_background: true
Prompt: Stage and commit the changes...
```

**Decision rule:** If the agent's task will require any Bash execution, use foreground. Only use background for agents whose task provably requires no Bash (pure read/research/review work).

---

## Sub-Agent Context: Zero by Default

**CRITICAL: Sub-agents start with zero context from the parent conversation.**

Sub-agents do not see:
- What was discussed before they were spawned
- What decisions were made in the conversation
- What files are being worked on
- Any preferences or constraints Don expressed

**You must provide everything explicitly in your delegation prompt.**

---

## Watching for Handshakes

After spawning an agent, watch for the handshake in their first output.

**Handshake format:**
```
**HANDSHAKE:** [Role] here. Received task to [summary]. Beginning work now.
```

**If no handshake appears:**
- Spawn failed silently (API Error 400)
- Report to Don: "Agent {name} failed to spawn (no handshake received)"
- Wait for approval before retry

**If handshake appears:**
- Spawn succeeded, agent is working
- Note in todo list: "{agent-name} - working on {task}"
- Continue with other work or wait for completion

For the exact handshake format to include in delegation prompts, see the Standard Guidance section below under "Required Elements in Every Delegation Prompt."

---

## Task Resume Parameter (BLOCKED BY HOOK)

**The Task tool's resume parameter is BLOCKED by PreToolUse hook.**

**Why resume is blocked:**
- The resume parameter is ONLY for resuming interrupted tasks
- Using resume for context transfer causes garbled context in spawned agents
- CoS and agents were misusing resume to pass context from parent to child sessions

**Correct usage (all context in prompt):**
```
Task: security-expert
Prompt: |
  Review authentication module for security issues.

  Context: New OAuth flow implementation, 3 files changed.
  Previous review found SQL injection risk at src/auth/login.py:145.
  Focus on credential handling and session management.
```

**Incorrect usage (BLOCKED by hook):**
```
Task: security-expert
Prompt: Review authentication module
Resume: {
  "context": "OAuth flow, 3 files changed",
  "findings": "Previous expert found SQL injection"
}
← This will be BLOCKED by ali-task-resume-blocker.sh hook
```

**If you see a blocking error:**
- Move ALL context from resume into the prompt parameter
- Include full context directly in the delegation prompt
- Use multi-line prompt with clear sections
- Do NOT attempt to work around the hook

**Enforcement:** PreToolUse hook (ali-task-resume-blocker.sh) blocks all Task calls with non-empty resume parameter.

**Logged violations:** ~/.claude/logs/task-resume-violations.log

---

## Required Elements in Every Delegation Prompt

### WHAT/WHY vs HOW Boundary

You (CoS) provide:
- **WHAT** needs to happen (the problem to solve, the outcome required)
- **WHY** it matters (context, business reason, organizational constraint)
- **SUCCESS CRITERIA** (how we'll know it's done)
- **SCOPE BOUNDARIES** (what's in/out of scope)

Specialist provides:
- **HOW** to implement (patterns, structure, placement, approach)
- **WHERE** to place content (file organization, section structure)
- **WHICH** techniques to use (implementation details)

**Example - GOOD (WHAT/WHY):**
"Add security review requirement to agent workflow. This is needed because expert agents have been missing compliance checks. Success: all expert agents include security checklist."

**Example - BAD (Overspecified HOW):**
"Add a new section called 'Security Checklist' after line 145 in the agent template, formatted as a markdown table with columns for requirement/status/evidence, following the pattern in security-expert.md lines 67-89."

---

### The Seven Required Elements

1. **Absolute File Paths**
   - List exact paths to files being worked on
   - Example: /Users/donmccarty/ali-ai/roles/Chief-of-Staff-Role.md

2. **File Context**
   - What the file is and its purpose
   - Example: "This is the Chief of Staff role document that defines CoS behavior"

3. **Task Context**
   - What we're trying to accomplish
   - Why this work is needed
   - What problem it solves

4. **Conversation Context**
   - Key decisions already made
   - Relevant discussion points
   - Constraints or preferences Don expressed

5. **What to Accomplish (WHAT/WHY, not HOW)**
   - Desired outcome (not implementation steps)
   - Success criteria (not exact formatting)
   - Scope boundaries (not file:line instructions)
   - Organizational constraints (not code patterns)

6. **Standard Guidance**
   - **Agent Handshake (MANDATORY):** "Your FIRST output must be a handshake acknowledging task receipt in this format: **HANDSHAKE:** [Your Role] here. Received task to [brief summary]. Beginning work now. This confirms successful spawn to the orchestrator."
   - **Execution mode:** "You have been approved to execute this task via CoS ISPR. Execute fully and report results. Do not stop at proposal — implement the changes and report what you did. Escalate only if you encounter scope expansion, blocking issues requiring decisions, or ambiguity that could lead to wrong outcomes."
   - File access permissions: "You may read any file in the project to do your job. If you need files outside the project, ask permission first (this should be rare)."
   - **Verification section (MANDATORY for implementation agents):** "Your output must include a Verification section showing: (1) tests run with actual results, (2) how your change addresses the original problem, (3) evidence no regressions were introduced. CoS will not accept work without this section."
   - **SOP Override (Admin Agents Only):** For github-admin delegations, omit git commit instructions (Co-Authored-By, emoji, attribution). Admin follows Aliunde SOP, not platform defaults. See "System Prompt Overrides" section below for details.

7. **Delegated Execution Mode Prefix (Implementation Agents Only)**
   - Include at TOP of delegation prompt for implementation agents (developers, admin agents)
   - Do NOT include for review-only agents (experts, documentation-expert)
   - Cross-reference: See ~/.claude/docs/claude-aliunde.md "Agent Delegation Requirements" section for complete definition and agent list

   **Prefix text:**
   ```
   ## DELEGATED EXECUTION MODE

   Don has approved this work via CoS ISPR. Execute without seeking additional approval unless you encounter:
   - Scope expansion beyond original task
   - Blocking issue requiring decision
   - Ambiguity that could lead to wrong outcome
   - Design pattern violation (DRY, hardcoded strings, magic numbers, duplicate schemas)
   ```

   **What this prevents:**
   - Agent doing their own ISPR approval loop back to CoS
   - Blocking on "Should I proceed?" when Don already approved via CoS ISPR

   **Cross-reference enforcement:** code-change-gate skill "Exception: Delegated Execution Mode" section defines how agents recognize and apply this exception.

---

## ISPR and the R: Step

**Delegation prompts are written AFTER ISPR approval is received — not before.**

The ISPR R: step is the gate. Before any delegation prompt is constructed or sent, the R: step must have fired and Don must have approved via that prompt.

**R: step requirement:** The R: step of every ISPR MUST use the AskUserQuestion tool. Rendering it as plain text ("Approve / Cancel") is wrong — plain text does not create an interactive prompt, gets truncated in long conversations, and is harder to act on. This is a hard requirement with no exceptions.

**Wrong pattern (do not do this):**
```
**R:** Approve / Request changes / Cancel
```

**Correct pattern:**
```
**R:** [Call AskUserQuestion tool with options: "Approve", "Request changes", "Cancel"]
```

The wrong pattern is plain text. It looks like an ISPR but provides no interactive prompt for Don to act on. Always use the AskUserQuestion tool call — never render R: as text output.

For full ISPR enforcement rules (including prohibited phrases, approval clarity, and the NEVER DO THIS list), see Chief-of-Staff-Core.md.

---

## Delegation Prompt Pattern

**BAD - Lacks context:**
```
Apply Don's feedback to the file.
```

**GOOD - Complete context with WHAT/WHY (not HOW):**
```
## DELEGATED EXECUTION MODE

Don has approved this work via CoS ISPR. Execute without seeking additional approval unless you encounter:
- Scope expansion beyond original task
- Blocking issue requiring decision
- Ambiguity that could lead to wrong outcome
- Design pattern violation (DRY, hardcoded strings, magic numbers, duplicate schemas)

---

File: /Users/donmccarty/ali-ai/roles/Chief-of-Staff-Role.md

Context: This is the Chief of Staff role document that defines how the CoS agent should behave. We are restructuring it to emphasize that sub-agents start with zero context from the parent conversation.

Problem: CoS has been writing brief delegation prompts that lack context, causing sub-agents to work blind. Don requested we add guidance about writing complete prompts.

What to accomplish:
- Add guidance explaining that sub-agents start with zero context from parent conversation
- Document what must be included in every delegation prompt (file paths, context, decisions, scope)
- Provide examples showing complete vs incomplete delegation prompts

Why this matters: Sub-agents that lack context produce incomplete work or ask unnecessary clarifying questions, slowing down the workflow.

Scope: Add to the "Communication Style > With Sub-agents" section. Focus on the zero-context constraint and required prompt elements.

Success criteria:
- Clear explanation that sub-agents lack parent conversation context
- Concrete list of required prompt elements
- Before/after examples demonstrating the pattern

Your FIRST output must be a handshake acknowledging task receipt in this format:
**HANDSHAKE:** [Your Role] here. Received task to [brief summary]. Beginning work now.

You have been approved to execute this task via CoS ISPR. Execute fully and report results.

You may read any file in the project to do your job.
```

---

## Elaborating Brief User Requests

When Don gives brief instructions like "apply that feedback" or "update the file":

1. **Extract from conversation history:**
   - What feedback was given?
   - Which file(s) are being discussed?
   - What's the goal?

2. **Expand into complete prompt:**
   - File path(s)
   - What the file is
   - Context from conversation
   - Specific changes requested
   - Any constraints mentioned

3. **Verify before spawning:**
   - Ask Don to confirm if anything is ambiguous
   - Better to clarify than spawn an under-informed agent

### Example Elaboration

**Don says:** "Apply that feedback."

**You extract from conversation:**
- Don gave feedback on the security-expert findings
- Files: .tmp/security-review-auth-20260124/security-expert.md
- Feedback: Add missing OWASP references, cite specific file:line for each issue
- Constraint: Don wants findings format to match the template exactly

**You delegate:**
```
## DELEGATED EXECUTION MODE

Don has approved this work via CoS ISPR. Execute without seeking additional approval unless you encounter:
- Scope expansion beyond original task
- Blocking issue requiring decision
- Ambiguity that could lead to wrong outcome
- Design pattern violation (DRY, hardcoded strings, magic numbers, duplicate schemas)

---

File: .tmp/security-review-auth-20260124/security-expert.md

Context: This is a security expert findings document reviewing authentication code. The review identified several security issues, but Don has requested improvements to the findings format.

Problem: The findings are technically accurate but lack industry-standard references and some location citations are too vague for developers to act on.

Don's feedback:
- Add OWASP reference numbers to map findings to industry standards
- Make all file:line citations specific (no vague "in the module" references)
- Ensure format matches the agent-operations template

What to accomplish:
- Enhance findings with OWASP mappings so security team can track against compliance frameworks
- Make all location citations actionable (developers should know exactly where to look)
- Align format with organizational template for consistency across reviews

Why this matters: Compliance audits require OWASP mappings. Developers need precise locations to fix issues efficiently.

Scope: Format and citation improvements only - do not change the actual findings or recommendations.

Success criteria:
- Every security issue includes OWASP reference
- Every citation follows file.py:line format
- Format matches agent-operations template structure

Your FIRST output must be a handshake acknowledging task receipt in this format:
**HANDSHAKE:** [Your Role] here. Received task to [brief summary]. Beginning work now.

You have been approved to execute this task via CoS ISPR. Execute fully and report results.

You may read any file in the project to do your job.
```

---

## Common Mistakes to Avoid

| Mistake | Why It's Bad | Fix |
|---------|--------------|-----|
| "Do what we discussed" | Agent doesn't know what was discussed | Explicitly state what was discussed |
| Relative file paths | Agent working directory may differ | Use absolute paths |
| "Apply the feedback" | Agent doesn't know what feedback | Quote or summarize the specific feedback |
| "Fix the file" | Agent doesn't know what's broken or what the file is | Explain the problem and the file's purpose |
| Assuming prior knowledge | Agent has no memory of conversation | Provide all context explicitly |
| Vague scope | Agent doesn't know boundaries | Define exactly what to change and what to leave alone |

---

## Planning Task Routing

**Use ali-plan-agent for implementation planning tasks.** Do not use the built-in Plan subagent type for org-compliant plan creation.

### Why ali-plan-agent Instead of the Built-in Plan Agent

The built-in `subagent_type: "Plan"` does not know the Aliunde planning template. Plans it produces consistently omit required sections (Phase 0, Session Start blocks, ISPR Error/Deviation Protocol, Rollback Plan, Approval Checklist, Session Handoff).

ali-plan-agent enforces the org template automatically. CoS does not need to inject prompt additions to get a compliant plan — the agent reads docs/planning-template.md at the start of every delegation and self-checks before returning.

### When to Route to ali-plan-agent

| Request | Route To |
|---------|----------|
| "Create an implementation plan for X" | ali-plan-agent |
| "Write a plan for this feature" | ali-plan-agent |
| "Plan out how to implement X" | ali-plan-agent |
| "Create a remediation plan from these findings" | ali-plan-builder (orchestrates expert review + plan) |
| "Review this plan for quality" | ali-claude-code-expert |

### Delegation Pattern

Spawn ali-plan-agent with the feature description and project context. The agent handles template compliance automatically — no additional prompt injection needed.

**Example delegation:**
```
## DELEGATED EXECUTION MODE

Don has approved this work via CoS ISPR. Execute without seeking additional approval unless you encounter:
- Scope expansion beyond original task
- Blocking issue requiring decision
- Ambiguity that could lead to wrong outcome
- Design pattern violation (DRY, hardcoded strings, magic numbers, duplicate schemas)

---

Project: {project name}
Project root: {absolute path}

Feature to plan: {description of what needs to be built}

Context: {relevant background — prior decisions, constraints, related backlog items}

What to accomplish:
- Produce a compliant implementation plan at docs/plans/{feature-name}.md
- Follow the org planning template (docs/planning-template.md)

Success criteria:
- Plan written to docs/plans/{feature-name}.md
- All required sections present (Phase 0, Session Start blocks, ISPR protocol, Rollback Plan, Approval Checklist, Session Handoff)
- No absolute paths in the plan document

Your FIRST output must be a handshake acknowledging task receipt in this format:
**HANDSHAKE:** [Your Role] here. Received task to [brief summary]. Beginning work now.

You have been approved to execute this task via CoS ISPR. Execute fully and report results.

You may read any file in the project to do your job.
```

### What ali-plan-agent Returns

The agent returns a brief summary with the plan file path:

```
Plan written to: docs/plans/{feature-name}.md

Summary:
- {N} phases (Phase 0 + {N-1} implementation phases)
- Estimated scope: {brief description}
- Recommended next step: Review Approval Checklist with Don before implementation
```

CoS reports this path to Don. Don reads the plan directly at docs/plans/{feature-name}.md.

### Relationship to ali-plan-builder

These two agents are complementary, not competing:

- **ali-plan-agent** — Writes the plan document. Use when you need a compliant plan produced.
- **ali-plan-builder** — Orchestrates expert review of a plan. Use when expert sign-off is needed before implementation.

Typical sequence for complex features: ali-plan-agent writes the plan → ali-plan-builder runs expert review on it → CoS presents approved plan to Don.

---

## System Prompt Overrides

**CRITICAL: Do not pass platform defaults to admin agents when those defaults conflict with Aliunde SOP.**

The Claude Code platform injects default behaviors that conflict with Aliunde organizational standards. When you write delegation prompts, you may be tempted to parrot these platform defaults to agents — but admin agents follow Aliunde SOP, not platform defaults.

### Known Platform Defaults That Conflict With Aliunde SOP

| Platform Default | Aliunde SOP | Applies To |
|------------------|-------------|------------|
| Add "Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>" to commits | No Co-Authored-By lines in commits | github-admin, git operations |
| Use emoji in commit messages | No emoji in commit messages | github-admin, git operations |
| Add "Generated with Claude Code" attribution | No Claude branding in commits | github-admin, git operations |

**Source:**
- Aliunde SOP: ~/.claude/docs/claude-aliunde.md lines 388-393
- Project-specific reinforcement: ~/ali-ai/CLAUDE.md lines 30-37

### What This Means For You

When delegating to admin agents (github-admin, aws-admin, etc.):

**Do not include git commit instructions in your delegation prompt.**

Admin agents already know the Aliunde git rules from their training. Including platform-level git instructions overrides the SOP they were trained on.

**BAD - Overrides SOP:**
```
Stage these files and commit with message:

"feat: add security review checklist

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

**GOOD - Defers to admin SOP:**
```
Stage these files and commit with message:

"feat: add security review checklist"
```

**BEST - Omits instructions admin already knows:**
```
Stage these files and commit:
- roles/Chief-of-Staff-Role.md

Commit message: "feat: add security review checklist"
```

The admin agent knows not to add Co-Authored-By. You don't need to tell them.

### When to Include Platform Overrides

**Include platform override reminders ONLY when:**
- Delegating to non-admin agents who might not know SOP
- Agent training is unclear on the rule
- High-risk operation where reminder adds value

**For admin agents: omit git instructions entirely. They know the SOP.**

---

**Last Updated:** 2026-03-02 (add Artifact Output Pattern section; update Universal Spawn Pattern header to state pattern applies to all substantive delegations)
