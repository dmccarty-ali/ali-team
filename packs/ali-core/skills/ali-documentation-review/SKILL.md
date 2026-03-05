---
name: ali-documentation-review
description: |
  Documentation quality standards and review patterns for project docs. Use when:

  PLANNING: Designing documentation structure, planning what to document,
  choosing documentation format, organizing project knowledge

  IMPLEMENTATION: Writing ARCHITECTURE.md, CLAUDE.md, README.md, updating
  documentation, adding code examples, documenting patterns

  GUIDANCE: Asking about documentation best practices, what to document,
  how to structure docs, when to update docs, documentation standards

  REVIEW: Checking documentation completeness, accuracy, currency, validating
  cross-references, detecting documentation drift, auditing project docs
---

# Documentation Review

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing documentation structure for new projects
- Planning what patterns need to be documented
- Choosing documentation format (markdown, diagrams, code examples)
- Organizing project knowledge and decision history

**Implementation:**
- Writing or updating ARCHITECTURE.md
- Writing or updating CLAUDE.md
- Writing or updating README.md
- Adding code examples to documentation
- Documenting new patterns or design decisions
- Creating migration guides or handoff documents

**Guidance/Best Practices:**
- Asking about documentation best practices
- Asking what should be documented vs what should be in code comments
- Asking how to structure project documentation
- Asking when documentation should be updated
- Asking about documentation standards for compliance projects

**Review/Validation:**
- Reviewing documentation for completeness
- Checking documentation accuracy (does it match code?)
- Validating cross-references (do file:line refs exist?)
- Detecting documentation drift (code changed, docs didn't)
- Auditing project documentation before releases

---

## Key Principles

- **Documentation has expiration dates**: Flag docs older than 6 months for review
- **File:line references must exist**: Always verify cross-references point to actual code
- **ARCHITECTURE.md is WHAT and WHY**: Not HOW (that's in code comments)
- **CLAUDE.md is PATTERNS and DECISIONS**: Project-specific rules and conventions
- **README.md is SETUP only**: Get developer running in 5 minutes max
- **Document decisions, not just code**: Why was this choice made? What alternatives were considered?
- **Code examples must work**: Never document pseudocode - always real, tested examples
- **Update docs immediately**: If pattern changes in code, update docs in same PR
- **Documentation drift is technical debt**: Stale docs are worse than no docs

---

## Documentation Structure Standards

### ARCHITECTURE.md

**Purpose:** Document the system's structure, key patterns, and design decisions.

**Required Sections:**

1. **System Overview** (2-3 paragraphs)
   - What problem does this system solve?
   - High-level architecture (components, data flow)
   - Technology stack (languages, frameworks, databases)

2. **Directory Structure** (annotated tree)
   ```
   src/
     api/           # FastAPI routes and schemas
     domain/        # Domain models and business logic
     services/      # External service integrations
     workflows/     # LangGraph flows and state machines
   ```

3. **Key Patterns** (with file:line references)
   - Authentication flow (auth.py:119-180)
   - File upload validation (file_validators.py:66-117)
   - Async operation handling (streamlit_bridge.py:36-79)
   - Include WHY pattern was chosen

4. **Integration Points**
   - External services (APIs, databases, S3, etc.)
   - Authentication method
   - Data storage strategy

5. **Design Decisions** (with dates and rationale)
   ```markdown
   ### Event Loop Management (2025-12-15)

   **Decision:** Use StreamlitAsyncBridge pattern for async operations
   **Rationale:** Streamlit reruns create new event loops, breaking asyncpg pools
   **Alternatives Considered:** Direct asyncio.run() (causes "Event loop closed" errors)
   **Implementation:** src/workflows/chat/streamlit_bridge.py:36-79
   ```

**Update Frequency:** Immediately when architecture changes, review quarterly

---

### architecture_stack.txt

**Purpose:** Machine-readable tech stack definition for automated agent/skill context detection.

**Required File:** All projects must have `architecture_stack.txt` in project root.

**Format:** YAML-style key-value structure (see PROJECT_TEMPLATE_architecture_stack.txt)

**Required Sections:**

1. **project** - Name, type, description
2. **databases** - Operational, warehouse, cache (with versions, ORM, migrations)
3. **languages** - Python, TypeScript, SQL dialect (with versions)
4. **frameworks** - Backend, frontend, orchestration, UI (with versions)
5. **infrastructure** - Cloud, compute, storage, IaC
6. **testing** - Backend, frontend, integration, BDD frameworks
7. **integrations** - External services (Stripe, Twilio, etc.)
8. **security** - Authentication, authorization, secrets, compliance
9. **monitoring** - APM, logging, errors

**Usage by Agents:**
- `database-developer` reads to auto-detect database type and patterns
- `expert-panel` reads to boost relevant expert scores based on stack
- `backend-developer`/`frontend-developer` verify using approved frameworks
- `documentation-review` validates consistency with ARCHITECTURE.md

**Validation Checklist:**

When reviewing documentation, verify:
- [ ] `architecture_stack.txt` exists in project root
- [ ] All technologies in ARCHITECTURE.md are listed in architecture_stack.txt
- [ ] All technologies in architecture_stack.txt are documented in ARCHITECTURE.md
- [ ] Versions match between files (if specified in ARCHITECTURE.md)
- [ ] Deprecated frameworks marked in both places
- [ ] New stack additions have rationale in ARCHITECTURE.md "Design Decisions"
- [ ] File is valid YAML syntax (no syntax errors)
- [ ] All required sections present (project, databases, languages, frameworks, infrastructure, testing)

**Example Consistency Check:**

```markdown
# ARCHITECTURE.md says:
"PostgreSQL Aurora 15.4 with SQLAlchemy 2.0 ORM"

# architecture_stack.txt must have:
databases:
  operational:
    - type: postgresql-aurora
      version: "15.4"
      orm: sqlalchemy
      migrations: alembic
```

**Update Frequency:** Immediately when stack changes (new framework, version upgrade, database change)

---

### CLAUDE.md

**Purpose:** Instructions for Claude Code to follow project-specific patterns and conventions.

**Required Sections:**

1. **Project Overview** (1-2 paragraphs)
   - What is this project?
   - What is unique about this codebase?

2. **Project-Specific Rules**
   - Patterns this project follows
   - Anti-patterns to avoid
   - Mandatory checks before code changes
   - Testing requirements

3. **Key Files** (where to find things)
   ```markdown
   | Purpose | Location |
   |---------|----------|
   | Authentication | src/auth/authentication.py |
   | File validation | src/utils/file_validators.py |
   ```

4. **Compliance Requirements** (if applicable)
   - IRS Circular 230 (tax applications)
   - HIPAA (healthcare applications)
   - SOC 2 (security compliance)
   - What must be logged, validated, documented

5. **Technology-Specific Patterns**
   - Streamlit: Use StreamlitAsyncBridge for async operations
   - LangGraph: Checkpoint configuration and state management
   - Database: Connection pooling and migration patterns

**Warning Signs CLAUDE.md Needs Update:**
- Recent PR added new pattern not mentioned in CLAUDE.md
- Instructions conflict with actual code
- Developer says "Claude keeps doing X wrong" (pattern not documented)
- New technology added but not covered in instructions

**Update Frequency:** Immediately when patterns change

---

### README.md

**Purpose:** Get a new developer running in 5 minutes or less.

**Required Sections:**

1. **Quick Start** (minimal steps)
   ```bash
   git clone <repo>
   cd <project>
   python -m venv .venv && source .venv/bin/activate
   pip install -r requirements.txt
   streamlit run src/workflows/chat_ui.py
   ```

2. **Prerequisites**
   - Python 3.11+
   - PostgreSQL 15+
   - AWS CLI (if applicable)

3. **Installation** (step-by-step)
   - Clone repository
   - Install dependencies
   - Setup database
   - Configure environment variables
   - Run migrations

4. **Configuration** (environment variables)
   ```bash
   # .env.example
   DATABASE_URL=postgresql://localhost/myapp
   AWS_REGION=us-east-1
   BEDROCK_MODEL_ID=anthropic.claude-sonnet-4-5-v1
   ```

5. **Testing**
   ```bash
   pytest tests/unit
   pytest tests/integration
   ```

6. **Common Issues** (troubleshooting)
   - PostgreSQL not running → `docker compose up -d postgres`
   - Module not found → `pip install -r requirements.txt`

**What NOT to Put in README.md:**
- Detailed architecture (that's ARCHITECTURE.md)
- Code patterns (that's CLAUDE.md)
- API documentation (that's separate docs/)
- Long explanations (keep it action-oriented)

**Update Frequency:** When setup process changes

---

### Optional Documentation Files

**MIGRATION_PLAN.md**
- Use when migrating from another codebase
- Track which files have been copied/refactored
- Note dependencies on old system

**HANDOFF.md**
- Created after major feature or bug fix
- Documents what was done and why
- Includes testing checklist
- Helps next developer maintain the code

**DECISIONS.md** (or ADR - Architecture Decision Records)
- One entry per major decision
- Date, context, decision, alternatives, consequences
- Immutable (never edit, only append)

---

## Documentation Quality Checklist

### Completeness

- [ ] ARCHITECTURE.md exists and documents system structure
- [ ] ARCHITECTURE.md has "System Overview" section
- [ ] ARCHITECTURE.md has "Directory Structure" section
- [ ] ARCHITECTURE.md has "Key Patterns" section with file:line refs
- [ ] ARCHITECTURE.md has "Design Decisions" section with dates
- [ ] CLAUDE.md exists and has project-specific instructions
- [ ] CLAUDE.md covers all major patterns
- [ ] CLAUDE.md documents technology-specific conventions
- [ ] README.md exists and has working Quick Start
- [ ] README.md prerequisites match actual dependencies
- [ ] README.md configuration section lists all env vars

### Accuracy

- [ ] File:line references point to existing code
- [ ] Code examples in docs actually work
- [ ] Setup instructions in README.md work (tested recently)
- [ ] Technology versions match requirements.txt/package.json
- [ ] No conflicting instructions in CLAUDE.md

### Currency

- [ ] ARCHITECTURE.md last updated within 6 months
- [ ] Recent patterns (last 3 months) are documented
- [ ] Deprecated patterns are marked as such
- [ ] No references to deleted files or moved code

### Cross-Reference Validation

Use these checks to validate documentation:

```bash
# Check if file paths in docs exist
grep -E "src/[^:]+\\.py" ARCHITECTURE.md | while read path; do
  [ -f "$path" ] || echo "MISSING: $path"
done

# Check if line numbers in docs are current
# Example: ARCHITECTURE.md says "auth.py:119-180"
# Verify those lines still contain authentication code
```

### Requirements-DB Cross-Reference Validation

When reviewing a requirements-db structure, validate that all cross-references are complete and functional.

**Anchor Targets Checklist:**

- [ ] Every REQ-F-NNN in requirements-catalog.md has an HTML anchor (e.g., `<a id="req-f-001"></a>`)
- [ ] Every REQ-NF-NNN in requirements-catalog.md has an HTML anchor
- [ ] Every CON-NNN in requirements-catalog.md has an HTML anchor
- [ ] Every ASM-NNN in assumptions-dependencies.md has an HTML anchor
- [ ] Every DEP-NNN in assumptions-dependencies.md has an HTML anchor
- [ ] Every DEC-NNN in decisions.md has a heading anchor (## or ###)
- [ ] Every RISK-NNN in risks.md has a heading anchor (## or ###)
- [ ] Every DOC-NNN link points to an existing file in documents/
- [ ] Every MTG-NNN link points to an existing file in meetings/

**Hyperlink Completeness Checklist:**

- [ ] No bare ID references in tables (every REQ-F-001 must be `[REQ-F-001](path#anchor)`)
- [ ] Source columns in catalogs link to extraction files (not bare DOC-NNN)
- [ ] Extraction files link back to catalog (ID column has hyperlinks, not bare IDs)
- [ ] Extraction files have "Source" metadata linking to original document URLs
- [ ] Traceability matrix uses range linking (links first and last ID in ranges)

**Relative Path Correctness Checklist:**

- [ ] Links from documents/ to requirements/ use `../requirements/`
- [ ] Links from requirements/ to documents/ use `../documents/`
- [ ] Links from cross-references/ to other directories use correct `../` prefix
- [ ] Same-directory links use filename only (no path prefix)

**Automated Validation Patterns:**

Find bare IDs that should be hyperlinked:

```bash
# Find REQ-F-NNN not preceded by [ (not a hyperlink)
grep -E 'REQ-F-[0-9]{3}' requirements-catalog.md | grep -v '\[REQ-F'

# Find DOC-NNN not preceded by [ (not a hyperlink)
grep -E 'DOC-[0-9]{3}' requirements-catalog.md | grep -v '\[DOC-'

# Find DEC-NNN not preceded by [ (not a hyperlink)
grep -E 'DEC-[0-9]{3}' decisions.md | grep -v '\[DEC-'

# Find ASM-NNN not preceded by [ (not a hyperlink)
grep -E 'ASM-[0-9]{3}' assumptions-dependencies.md | grep -v '\[ASM-'

# Find CON-NNN not preceded by [ (not a hyperlink)
grep -E 'CON-[0-9]{3}' requirements-catalog.md | grep -v '\[CON-'

# Find RISK-NNN not preceded by [ (not a hyperlink)
grep -E 'RISK-[0-9]{3}' risks.md | grep -v '\[RISK-'
```

**Note:** These grep patterns will find bare ID references in markdown. Review each match to determine if it should be a hyperlink.

---

## Documentation Drift Detection

**Symptoms of Documentation Drift:**

1. **New Pattern Not Documented**
   - Developer added StreamlitAsyncBridge 3 months ago
   - ARCHITECTURE.md doesn't mention it
   - Next developer reinvents the pattern

2. **Outdated Instructions**
   - CLAUDE.md says "use asyncio.run()"
   - Code now uses StreamlitAsyncBridge
   - Claude keeps suggesting wrong pattern

3. **Stale File References**
   - ARCHITECTURE.md references old_service.py:145
   - File was refactored to new_service.py
   - Developer can't find referenced code

4. **Missing Design Decisions**
   - Code has complex workaround
   - No documentation of WHY
   - Looks like tech debt, gets "fixed" incorrectly

**How to Detect Drift:**

```bash
# Find docs older than 6 months
find . -name "ARCHITECTURE.md" -mtime +180

# Find TODO comments mentioning documentation
git grep "TODO.*document"

# Find recent commits without doc updates
git log --since="3 months ago" --name-only | grep -v "\.md$"

# Compare documented patterns with actual code
# (requires manual review)
```

**Fix Documentation Drift Immediately:**
- Create GitHub Issue with "documentation" label
- Update docs in same PR that changes pattern
- Add pre-commit hook to remind about docs

---

## Code Examples in Documentation

**Rules for Code Examples:**

1. **Real Code Only** - Never pseudocode
   ```markdown
   ❌ BAD:
   ```python
   def validate_file(file):
       # Validate file here
       pass
   ```

   ✅ GOOD:
   ```python
   def validate_file(content: bytes, filename: str) -> tuple[bool, str]:
       if len(content) > MAX_FILE_SIZE:
           return False, "File too large"
       return True, ""
   ```
   ```

2. **Include File:Line References**
   ```markdown
   File upload validation (file_validators.py:66-117):
   ```python
   def validate_file(content, filename, content_type):
       # ... actual implementation
   ```
   ```

3. **Show Context** - Include surrounding code if needed
   ```python
   # From chat_ui.py:502-514
   uploaded_file = st.file_uploader("Upload", type=["pdf", "png"])
   if uploaded_file:
       content = uploaded_file.read()
       is_valid, error = validate_file(content, uploaded_file.name, uploaded_file.type)
       if not is_valid:
           st.error(error)
           st.stop()
   ```

4. **Test Examples** - Ensure examples work
   - Run code snippets before committing
   - Use doctest for inline examples
   - Include in integration tests

---

## Documentation for Compliance Projects

**Tax/Healthcare/Financial applications have special documentation requirements:**

### Audit Trail Documentation

**REQUIRED in ARCHITECTURE.md:**
- What events are logged (login, logout, data access, modifications)
- Log format (JSON, structured, append-only)
- Log retention policy (7 years for tax, 6 years for HIPAA)
- Where logs are stored (file, database, CloudWatch)

**Example:**
```markdown
### Audit Logging (IRS Circular 230 Compliance)

**Events Logged:**
- staff_login, staff_logout (authentication)
- account_lockout (security)
- session_timeout (security)
- document_upload, document_view (data access)

**Log Format:** JSON, append-only file (logs/audit.log)
**Retention:** 7 years (IRS requirement)
**Implementation:** audit_log() function in chat_ui.py:69-95
```

### Security Pattern Documentation

**REQUIRED in ARCHITECTURE.md:**
- Authentication method (database, OAuth, SSO)
- Session management (timeouts, lockout protection)
- File upload validation (size, type, magic bytes)
- PII handling (encryption, masking)

### Compliance Checklist Documentation

**REQUIRED in CLAUDE.md or separate COMPLIANCE.md:**
- [ ] All user actions logged
- [ ] Session timeout enforced (12h absolute, 30m idle)
- [ ] Lockout protection (5 attempts, 15 min)
- [ ] File uploads validated (size, type, content)
- [ ] PII encrypted at rest
- [ ] Audit logs append-only
- [ ] 7-year retention documented

---

## Documentation Anti-Patterns

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Documentation in comments only | Gets out of sync, not searchable | Put in ARCHITECTURE.md with file:line refs |
| "See code for details" | Not helpful, documentation should add context | Explain WHY and WHAT, reference code for HOW |
| Copy-paste from other projects | Context doesn't match | Write project-specific docs |
| No file:line references | Can't find implementation | Always include src/file.py:line |
| Pseudocode examples | Doesn't help, might be wrong | Use real, tested code |
| README.md = everything | Too long, unmaintainable | Split: README=setup, ARCHITECTURE=structure, CLAUDE=patterns |
| No update date | Can't tell if stale | Add "Last Updated: YYYY-MM-DD" to each doc |
| No design decisions | Future devs don't know WHY | Document decisions with date, rationale, alternatives |
| Requirements IDs without hyperlinks | Traceability is visual only; can't navigate the document web | Every ID reference should be a clickable markdown link |

---

## Documentation Maintenance Schedule

**Immediate (Same PR):**
- New pattern added → Update CLAUDE.md
- Architecture changed → Update ARCHITECTURE.md
- Setup process changed → Update README.md

**Quarterly (Every 3 Months):**
- Review ARCHITECTURE.md for currency
- Verify all file:line references
- Check for documentation drift
- Update outdated examples

**Before Release:**
- Full documentation audit
- Test README.md setup instructions
- Verify compliance documentation (if applicable)
- Update version references

**After Incident:**
- Document pattern that caused incident
- Add to CLAUDE.md anti-patterns
- Create HANDOFF.md with fix details

---

## Documentation Review Process

**Self-Review Checklist (Before PR):**

1. **Did I add new pattern?** → Update CLAUDE.md
2. **Did I change architecture?** → Update ARCHITECTURE.md
3. **Did I change setup?** → Update README.md
4. **Did I reference code?** → Include file:line refs
5. **Did I add code example?** → Test it works
6. **Did I make design decision?** → Document WHY with date

**Peer Review Checklist (During PR Review):**

- [ ] PR includes documentation updates (if needed)
- [ ] File:line references are accurate
- [ ] Code examples match actual code
- [ ] Design decisions explained with rationale
- [ ] No conflicts with existing CLAUDE.md instructions

**Automated Checks (Pre-commit Hook):**

```bash
# .git/hooks/pre-commit
#!/bin/bash

# Check if code changed but no docs updated
CODE_CHANGED=$(git diff --cached --name-only | grep -E "\.(py|ts|tsx)$")
DOCS_CHANGED=$(git diff --cached --name-only | grep -E "\.(md)$")

if [ -n "$CODE_CHANGED" ] && [ -z "$DOCS_CHANGED" ]; then
  echo "⚠️  Code changed but no documentation updated."
  echo "    Consider updating ARCHITECTURE.md, CLAUDE.md, or README.md"
  echo "    (This is a reminder, not a blocker)"
fi
```

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| Skills README | `skills/README.md` |
| Skill template | `skills/SKILL_TEMPLATE.md` |
| Project template | `PROJECT_TEMPLATE.md` |
| Example ARCHITECTURE.md | `projects/tax-ai-chat/ARCHITECTURE.md` |
| Example CLAUDE.md | `projects/tax-ai-chat/CLAUDE.md` |

---

## Quick Reference

### ARCHITECTURE.md Sections

1. System Overview
2. Directory Structure
3. Key Patterns (with file:line refs)
4. Integration Points
5. Design Decisions (with dates)

### CLAUDE.md Sections

1. Project Overview
2. Project-Specific Rules
3. Key Files
4. Compliance Requirements
5. Technology-Specific Patterns

### README.md Sections

1. Quick Start
2. Prerequisites
3. Installation
4. Configuration
5. Testing
6. Common Issues

### File:Line Reference Format

```markdown
Authentication flow (auth.py:119-180)
File validation (file_validators.py:66-117)
Async bridge pattern (streamlit_bridge.py:36-79)
```

### Design Decision Template

```markdown
### [Pattern Name] (YYYY-MM-DD)

**Decision:** [What was decided]
**Rationale:** [Why this decision was made]
**Alternatives Considered:** [What else was considered and why rejected]
**Implementation:** [File:line references]
**Trade-offs:** [What we gave up for this decision]
```

---

## References

- [Documentation Best Practices](https://documentation.divio.com/)
- [Architecture Decision Records](https://adr.github.io/)
- [Writing Documentation](https://www.writethedocs.org/guide/)
- [README Best Practices](https://github.com/RichardLitt/standard-readme)
