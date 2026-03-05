---
name: ali-agent-operations
description: |
  Operational guidelines for expert subagents. Use when:

  PLANNING: N/A - This skill is for agent execution, not planning

  IMPLEMENTATION: N/A - This skill is for agent execution, not implementation

  GUIDANCE: N/A - This skill is for agent execution, not guidance

  REVIEW: Conducting any formal review as an expert subagent - file operations,
  output format, working directory handling, error reporting
---

# Agent Operations

Operational guidelines that all expert subagents must follow for reliable file access and accurate reviews.

---

## Agent Handshake (MANDATORY - FIRST OUTPUT)

**CRITICAL: Acknowledge task receipt BEFORE beginning any work.**

**Format:**
```
**HANDSHAKE:** [Role] here. Received task to [brief summary]. Beginning work now.
```

**Why This Exists:**
- Orchestrators cannot see if sub-agents successfully spawned due to silent API Error 400 failures
- Handshake confirms spawn succeeded before work begins
- If orchestrator sees no handshake after spawning, it knows spawn failed

**Examples:**
- "**HANDSHAKE:** Security Expert here. Received task to review authentication implementation for OWASP compliance. Beginning work now."
- "**HANDSHAKE:** Frontend Developer here. Received task to add logout button to header component. Beginning work now."
- "**HANDSHAKE:** Database Developer here. Received task to optimize slow queries in client service. Beginning work now."

**This is your FIRST output.** Do not read files, do not analyze, do not plan - handshake FIRST, then begin work.

---

## Design Pattern Validation (MANDATORY)

**Before making ANY recommendation, check for design pattern violations.**

Expert agents are the highest-level gate against anti-patterns. You are NOT just answering questions - you are CHALLENGING proposals against documented standards.

### Mandatory Pre-Recommendation Check

Before recommending any change, verify it doesn't violate:

1. **DRY (Don't Repeat Yourself)** - Does this duplicate existing code/config?
2. **No hardcoded strings** - Does this add values that should come from centralized config?
3. **Single source of truth** - Does this create a second place where this value is defined?
4. **No magic numbers** - Are there unexplained numeric values?

If the proposed change violates any of these, **FLAG IT AS A BLOCKING ISSUE** - do not provide implementation guidance for an anti-pattern.

### Example

**BAD Expert Response:**
> "To fix the database name, update the hardcoded value in these 10 files from 'tax_practice' to 'tax_ai_chat'"

**GOOD Expert Response:**
> "BLOCKING ISSUE: This proposal perpetuates hardcoded database names across 10 files, violating DRY and the 'no hardcoded strings' rule. Instead, define the database name once in config.yaml and import it everywhere."

### Your Role

You are not a rubber stamp. You are the last line of defense. If the proposed approach is architecturally wrong, say so - even if it technically answers the question asked.

---

## Evidence-Based Recommendations (MANDATORY)

**Before making ANY claim about code, you MUST provide file:line evidence.**

Expert agents are trusted advisors. Trust requires verification. **NEVER make claims about code without citing specific evidence.**

### Citation Requirements

**Every claim must include:**
- **File path:** Absolute or project-relative path
- **Line number:** Specific line(s) where evidence exists
- **Format:** `path/to/file.py:123` or `file.py:123-145` (range)

**Examples:**

**GOOD (with evidence):**
> "BLOCKING ISSUE at src/auth/oauth.py:145 - SQL injection vulnerability in login query. The user input is not parameterized."

**BAD (no evidence):**
> "BLOCKING ISSUE in auth module - SQL injection vulnerability in login query."

**GOOD (with evidence):**
> "Recommendation: Add rate limiting to API endpoints. Current implementation at api/routes.py:67-89 has no throttling."

**BAD (no evidence):**
> "Recommendation: Add rate limiting to API endpoints. No throttling detected."

### When You Can't Verify

If you cannot verify a claim, say so explicitly:

**Acceptable alternatives:**
- "Unable to verify: I cannot access the database schema files"
- "Unable to verify: The authentication configuration may exist outside this project"
- "Unable to verify: Need clarification on [specific ambiguity]"

**NOT acceptable:**
- Making assumptions without verification
- Guessing line numbers or file paths
- Claiming patterns exist without reading the files
- Recommending solutions based on assumed code structure

### Pre-Recommendation Search Protocol

**Before making ANY recommendation, verify the current state:**

1. **Use Grep to search** for existing patterns:
   ```
   Grep: "pattern to search" with path: "src/"
   ```

2. **Read the actual files** where patterns found:
   ```
   Read: src/auth/oauth.py
   ```

3. **Cite specific evidence** in your recommendation:
   ```
   "At oauth.py:145, the query uses string concatenation..."
   ```

4. **If pattern not found**, say so:
   ```
   "Grep search for 'rate_limit' returned no results. Rate limiting not implemented."
   ```

### Evidence in Findings Files

When writing findings to .tmp/ directory, structure evidence clearly:

```markdown
## Critical Issues

### C1: SQL Injection Vulnerability
**Location:** src/auth/oauth.py:145
**Evidence:**
```python
query = "SELECT * FROM users WHERE username='" + username + "'"
```
**Recommendation:** Use parameterized queries
```

**NOT this:**
```markdown
## Critical Issues

### C1: SQL Injection Vulnerability
**Location:** auth module
**Recommendation:** Use parameterized queries
```

---

## File Operations

### Use the Right Tools

**ALWAYS use these tools for file operations:**
- `Read` - Read file contents
- `Grep` - Search for patterns in files
- `Glob` - Find files by pattern

**NEVER use bash commands for file operations:**
- No `cat`, `head`, `tail`, `less`
- No `find`, `ls` for file discovery
- No `grep` via bash (use the Grep tool instead)

Bash commands may fail due to working directory issues in subagent context.

### Use Absolute Paths

**ALWAYS use absolute paths** starting from root:

```
# GOOD - full path from root (construct from working directory context)
/home/user/ali-ai/projects/ingestion-engine/terraform/modules/kms/main.tf

# BAD - relative paths may fail
terraform/modules/kms/main.tf
./terraform/modules/kms/main.tf
```

The working directory for subagents may differ from the parent session. Use the project root path provided in your task context.

### Verify Before Claiming

**NEVER make claims about file content you haven't read:**
- If a Read fails, report that explicitly
- Do not guess or hallucinate file contents
- Do not assume files exist without verifying

```markdown
# BAD
"The file contains lifecycle blocks at lines 73-76"
(when you haven't actually read the file)

# GOOD
"I was unable to read the file at [path]. Please verify the path exists."

# GOOD
"Reading terraform/modules/kms/main.tf..."
[Actually read the file]
"The file contains lifecycle blocks at lines 266-269 and 333-336"
```

---

## Working Directory

Subagents may have a different working directory than the parent conversation. To handle this:

1. **Ask for or construct absolute paths** when given relative paths
2. **Use Glob with absolute base path** to discover files
3. **Report path issues** clearly if files cannot be found

Example discovery pattern:
```
# Use Glob to find files - construct path from project root
Glob pattern: "*.tf" with path: "{project_root}/terraform/modules/kms"
```

---

## Output Requirements

### Files Reviewed Section

Every review MUST include a "Files Reviewed" section listing:
- The absolute path of each file actually read
- Only files you successfully read (not files you attempted but failed)

```markdown
### Files Reviewed
- {project_root}/terraform/modules/kms/main.tf
- {project_root}/terraform/modules/kms/variables.tf
- {project_root}/terraform/modules/kms/outputs.tf
```

### Line Number References

When citing specific code:
- Include file path and line number: `main.tf:266`
- Verify line numbers by actually reading the file
- Use the Read tool's line numbers (shown as `N→` prefix)

---

## Writing Findings

**Expert agents write their own findings to .tmp/ directory.**

### Determining Output Path

Check your prompt for an Output-Path:

1. **If Output-Path provided** → Write to that path
   ```
   Output-Path: .tmp/security-review-auth-20260124/security-expert.md
   ```

2. **If no Output-Path** → Auto-generate:
   ```
   .tmp/{YYYYMMDD-HHMMSS}-{your-expert-name}.md
   ```
   Example: `.tmp/20260124-203045-security-expert.md`

### File Structure

```
.tmp/
├── security-review-auth-20260124/     # Coordinated review (orchestrator set path)
│   ├── security-expert.md
│   ├── backend-expert.md
│   └── database-expert.md
├── 20260124-203045-security-expert.md  # Direct invocation (auto-generated)
└── 20260124-204512-backend-expert.md   # Direct invocation (auto-generated)
```

### What You Write

Write your complete findings using this structure:

```markdown
# {Your Expert Name} Review Findings

**Output:** {path where this file was written}
**Date:** {YYYY-MM-DD HH:MM}

## Scope
Files reviewed:
- /absolute/path/to/file1
- /absolute/path/to/file2

## Critical Issues (Must Fix)

| ID | Location | Description |
|----|----------|-------------|
| C1 | {file:line} | {description} |

### C1: {Issue Title}
**Location:** {file:line}
**Description:** {detailed description}
**Recommendation:** {how to fix}

## Warnings (Should Address)
...

## Recommendations (Consider)
...

## Approval Status
{Approved / Approved with revisions / Blocked}

## Files Reviewed
- {list all files actually read}
```

### After Writing

Return a brief summary in context:
```
Findings written to: {output-path}

Summary:
- {N} critical issues
- {N} warnings
- {N} recommendations
- Status: {Approved / Approved with revisions / Blocked}
```

This keeps context small while full findings are persisted to file.

---

## Error Handling

### File Not Found

If a file cannot be read:
1. Report the attempted path
2. Suggest possible corrections (typos, wrong directory)
3. Do NOT guess what the file might contain

```markdown
### Unable to Review
The following files could not be read:
- `/path/to/file.tf` - File not found

Please verify the paths and re-run the review.
```

### Partial Review

If some files are readable and others are not:
1. Complete the review for accessible files
2. Clearly note which files were not reviewed
3. Indicate that findings may be incomplete

---

## CRITICAL: No Guessing or Assumptions

**Guessing is unacceptable.** Expert agents are expected to VERIFY, not assume.

### The Verification Mandate

Before making ANY claim or finding:

1. **READ the actual file** - Use Read tool with absolute path
2. **CITE specific evidence** - File path and line number (format: file.py:123)
3. **If you cannot verify, say so** - "Unable to verify" is acceptable; guessing is NOT

### What Happens When You Guess

When expert agents make assumptions instead of verifying:
- Findings get challenged and invalidated
- Owner loses trust in expert reviews
- Time is wasted on verification passes
- The review provides no value

**Example of violation:**
> "The authentication module has a security issue at line 145"
> (Agent never read the file, guessed the line number)

**Correct approach:**
> "Unable to verify authentication module security without access to src/auth/ directory"
> OR (if accessible):
> "BLOCKING ISSUE at src/auth/oauth.py:145 - SQL injection vulnerability [evidence cited]"

### Evidence Requirements Summary

| Claim Type | Evidence Required | Format |
|------------|------------------|--------|
| Code issue | File path + line number + code snippet | `file.py:123` + snippet |
| Pattern exists | Grep results + file verification | `Grep found in file.py:45` |
| Pattern missing | Grep search results (empty) | `Grep 'pattern' returned no results` |
| Recommendation | Current state evidence + proposed change | `At file.py:123, currently X. Recommend Y` |

### Delegation Strategy

If you cannot access a file or need additional context:

1. **Report the limitation** - "I cannot access [path]"
2. **Delegate if appropriate** - Spawn a Haiku agent to search/retrieve
3. **Ask for clarification** - If truly blocked, request the information

See `ali-ai/docs/claude-aliunde.md` (Model Delegation Strategy section) for delegation hierarchy:
- Opus: Architecture decisions, ambiguous requirements
- Sonnet: Feature implementation, multi-file changes
- Haiku: File searches, simple tasks

### Before Submitting Findings

Ask yourself:
- Did I actually READ every file I'm citing? ✓
- Do I have SPECIFIC line numbers from actual file content? ✓
- Am I making ANY assumptions I haven't verified? ✗
- Have I listed ALL files I reviewed? ✓
- Are ALL my citations in format `file.py:123`? ✓

If any answer is wrong, **go back and do the work.**

---

## Before Declaring Complete (Implementation Tasks)

For agents performing implementation (not review), complete this self-check:

### Ownership Checklist

1. **Did I verify it works?**
   - [ ] Ran tests or demonstrated functionality
   - [ ] Can show evidence, not just assertion

2. **Does it solve the stated problem?**
   - [ ] Re-read the original request
   - [ ] My change directly addresses it
   - [ ] I can explain HOW it addresses it

3. **Is my output complete?**
   - [ ] Verification section included
   - [ ] Test results shown (actual output, not just "passed")
   - [ ] Problem-to-solution mapping clear

4. **Would I ship this to production?**
   - [ ] Confident it works
   - [ ] Not just "should work" or "might work"

**If any answer is NO, you are not done.**

### The "Might Work" Test

| Red Flag Phrases | What to Do Instead |
|------------------|-------------------|
| "This should fix it" | Run tests, show results |
| "Try this and see" | Verify yourself first |
| "I think this works" | Demonstrate it works |
| "Testing Notes: run X" | Actually run X, show output |

**Don't throw code over the wall. Deliver verified solutions.**

---

## File Scope Check

Before writing any file, verify you are the correct agent for that file type. Writing outside your domain is a routing error — even if you have the Write tool.

### File Type Routing

| File Type | Correct Agent | Examples |
|-----------|---------------|----------|
| Shell scripts, hooks | ali-bash-developer | *.sh, hooks/*, scripts/* |
| Backend code | ali-backend-developer | *.py (app code), services/*, api/* |
| Frontend code | ali-frontend-developer | *.tsx, *.ts, components/*, pages/* |
| Tests | ali-test-developer | test_*.py, *.test.ts, conftest.py |
| Database | ali-database-developer | migrations/*, *.sql, schemas/* |
| Documentation | ali-document-developer | ARCHITECTURE.md, README.md, docs/*, roles/*.md |
| Claude Code config | ali-claude-developer | CLAUDE.md, skills/**/SKILL.md, agents/*.md, .claude/** |
| Unrecognized file type | — | STOP: Report file path to orchestrator and ask for routing decision |

### Self-Check Before Writing

Before using Write or Edit on any file:

1. Identify the file type from the table above
2. If this is YOUR domain — proceed
3. If this is NOT your domain — STOP and report:

"ROUTING ERROR: [file path] belongs to [correct-agent] domain. I should not write this file."

This check is blocking. Do not write the file. Name the correct agent and return control.

Exception: If the delegation prompt explicitly states you are handling this file as a special case, proceed with a logged note.

**When in Doubt:** If a file type is not listed in the table above (e.g., Dockerfile, *.yaml, *.json config, Makefile, *.toml), do not guess the correct agent. STOP and report to the orchestrator with the file path, asking for a routing decision.

---

## Anti-Patterns

| Anti-Pattern | Problem | Do This Instead |
|--------------|---------|-----------------|
| Using `cat` via Bash | May fail in subagent context | Use Read tool |
| Using relative paths | Working directory differs | Use absolute paths |
| Claiming content without reading | Hallucination risk | Always read first |
| Using `find` via Bash | May fail in subagent context | Use Glob tool |
| Guessing line numbers | Inaccurate citations | Read and count lines |
| Skipping "Files Reviewed" | No audit trail | Always list files read |
| **Making assumptions** | **Findings get invalidated** | **Verify everything** |
| **Guessing file contents** | **Unacceptable** | **Read or report inability** |
| **Assuming patterns exist** | **Creates false findings** | **Search and confirm** |
| **Bash heredoc file creation** | **Bypasses tool tracking, creates unusable permission prompts, pollutes settings.json if allowlisted** | **Use the Write tool to create files** |
| **Inline script complexity** | **Commands with loops/conditionals cause unusable approval dialogs** | **Write as .tmp/scripts/ file first (see ali-bash skill for thresholds)** |
| **Long inline git commands** | **Never construct multi-line git commit/tag messages via heredoc. Same heredoc problems apply; git has native flags.** | **Write message to file, use `git commit -F` / `git tag -F`** |
