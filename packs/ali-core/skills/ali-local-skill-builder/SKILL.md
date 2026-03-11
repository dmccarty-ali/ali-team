---
name: ali-local-skill-builder
description: |
  Guide for creating project-specific skills in .claude-local/skills/. Use when:

  PLANNING: Deciding whether a skill should be project-local or org-wide, planning
  what knowledge to encode, designing trigger patterns for a new project skill,
  determining skill scope and naming

  IMPLEMENTATION: Writing a new .claude-local/skills/ skill, creating the SKILL.md
  file and frontmatter, building trigger descriptions, authoring key principles
  and patterns for a single project

  GUIDANCE: Asking how to add a skill to a specific project, asking about the
  difference between project-local and org-wide skills, asking where project
  skills live and how they get loaded

  REVIEW: Reviewing an existing .claude-local/ skill for structure, trigger
  coverage, naming correctness, and flat directory compliance
---

# Local Skill Builder (Project-Specific Skills)

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Deciding whether a new skill belongs to one project or all projects
- Determining what project-specific knowledge to encode in a skill
- Planning trigger patterns for a skill that covers a single project's conventions
- Designing skills that reference project-specific file paths, APIs, or domain rules

**Implementation:**
- Creating a new skill under `.claude-local/skills/`
- Writing frontmatter, trigger descriptions, and body content for a project skill
- Setting up the `.claude-local/` directory structure for the first time

**Guidance/Best Practices:**
- Asking how to add a skill that only applies to one project
- Asking where project-local skills live vs. org-wide skills
- Asking how project skills get loaded at runtime and whether a release is needed

**Review/Validation:**
- Reviewing a `.claude-local/` skill for correct frontmatter, naming, and flat structure
- Checking that trigger descriptions cover all four modes
- Validating that the skill is in git and not in .gitignore

## When NOT to Use This Skill

This skill does NOT activate for:
- Creating org-wide skills available to all projects (use ali-org-skill-builder instead)
- Creating project-specific agents (use `.claude-local/agents/` directly)
- Writing CLAUDE.md content (use ali-claude-code skill)
- Configuring hooks or settings (use ali-claude-code skill)

---

## Key Principles

- **Project-local = .claude-local/skills/**, never `.claude/skills/` — `.claude/` is auto-generated and gitignored; `.claude-local/` is git-tracked
- **No ali- prefix required** — project skill names are kebab-case without the org prefix (e.g., `tax-domain`, `ingestion-patterns`)
- **Directory name must match frontmatter name exactly** — mismatch causes "Unknown skill" errors
- **Flat structure is mandatory** — skills live at `.claude-local/skills/{name}/SKILL.md`, never deeper
- **No release needed** — project skills are available immediately on the next `claude-ali` launch after commit; no tarball needed
- **Git-track the skill** — `.claude-local/` is committed to the project repo; it is not in `.gitignore`
- **Overlays on org skills** — project skills are overlaid on top of `.claude/skills/` at sync time; they can coexist with org skills of different names
- **Four-mode triggers required** — skills without all four modes (PLANNING, IMPLEMENTATION, GUIDANCE, REVIEW) only trigger ~25% of the time

---

## When to Use Project-Local vs. Org-Wide

| Scenario | Use |
|----------|-----|
| Knowledge only relevant to one project (domain rules, specific APIs) | `.claude-local/skills/` |
| Knowledge useful across multiple Aliunde projects | `~/ali-ai/skills/` (org-wide) |
| Proprietary patterns you don't want in the shared org repo | `.claude-local/skills/` |
| Technology skill (bash, Python, AWS, etc.) all teams use | `~/ali-ai/skills/` (org-wide) |
| Temporary or experimental skill in development | `.claude-local/skills/` |
| Skill that references hardcoded paths from one project | `.claude-local/skills/` |

**Decision rule:** If you'd want teammates on other projects to use it too, it's org-wide. Otherwise, it's local.

---

## Directory Structure

Project-local skills live here (git-tracked):

```
{project-root}/
├── .claude-local/               # Git-tracked project resources
│   ├── agents/                  # Project-only agents (optional)
│   └── skills/
│       └── {skill-name}/        # One directory per skill
│           └── SKILL.md         # Skill content (exact filename)
│
├── .claude/                     # Auto-generated at launch (gitignored)
│   └── skills/                  # Org skills land here after sync
│
└── .gitignore                   # Contains: .claude/  (NOT .claude-local/)
```

At `claude-ali` launch, the launcher overlays `.claude-local/skills/` on top of `.claude/skills/`. Both org and project skills are available inside Claude Code.

---

## Patterns We Use

### SKILL.md Frontmatter (Minimal Required)

Every project skill needs valid frontmatter. The `name` field must match the directory name exactly:

```yaml
---
name: tax-domain
description: |
  Tax practice domain knowledge for the tax-ai-chat project. Use when:

  PLANNING: Designing tax workflow features, planning IRS Circular 230 compliance
  controls, evaluating how tax rules affect feature design

  IMPLEMENTATION: Implementing tax calculation logic, writing tax form processing,
  handling tax-period-specific business rules

  GUIDANCE: Asking about tax practice conventions, IRS deadlines, or how our
  system models client-preparer relationships

  REVIEW: Reviewing tax-related features for domain correctness, checking
  compliance requirements are met, validating tax period handling
---
```

**No `allowed-tools` needed** unless you want to restrict the tools this skill activates for (rare for project skills).

### Full SKILL.md Template

Copy this for a new project skill:

```markdown
---
name: {skill-name}
description: |
  {One-line summary of what this skill covers}. Use when:

  PLANNING: {planning activities where this skill adds value}

  IMPLEMENTATION: {implementation activities where this skill adds value}

  GUIDANCE: {questions this skill answers about best practices or how-tos}

  REVIEW: {review and validation activities this skill guides}
---

# {Skill Display Name}

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- {Specific planning scenario}
- {Another planning scenario}

**Implementation:**
- {Specific implementation task}
- {Another implementation task}

**Guidance/Best Practices:**
- {Question this skill answers}
- {Another question}

**Review/Validation:**
- {Review scenario}
- {Another review scenario}

## When NOT to Use This Skill

This skill does NOT activate for:
- {What this skill does NOT cover}

---

## Key Principles

- **Principle 1** — explanation
- **Principle 2** — explanation
- **Principle 3** — explanation

---

## Patterns We Use

### {Pattern Name}

{Brief description of the pattern}

```python
# Code example from this project
```

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| {bad practice} | {consequence} | {correct approach} |

---

## Quick Reference

```bash
# Frequently used commands for this project
```

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| {description} | `{relative/path/to/file}` |
```

### Naming Convention

Project skills do NOT use the `ali-` prefix. Use descriptive kebab-case:

```
# CORRECT - project skill names
.claude-local/skills/tax-domain/SKILL.md
.claude-local/skills/ingestion-patterns/SKILL.md
.claude-local/skills/client-portal-ux/SKILL.md

# WRONG - ali- prefix is for org skills only
.claude-local/skills/ali-tax-domain/SKILL.md
```

The `name:` field in frontmatter must match the directory name exactly:

```
# Directory: .claude-local/skills/tax-domain/
# Frontmatter: name: tax-domain   <-- must match
```

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Placing skill in `.claude/skills/` | Gitignored, overwritten at next sync | Place in `.claude-local/skills/` |
| Nesting deeper than one level | Skill discovery only looks one level deep — skill is invisible | Use flat: `.claude-local/skills/{name}/SKILL.md` |
| Directory name mismatches `name:` field | Skill tool resolves by frontmatter; slash command resolves by directory — mismatch causes "Unknown skill" | Keep them identical |
| Using `ali-` prefix for project skills | Looks like an org skill, creates confusion | Omit prefix, use plain kebab-case |
| Skill description with fewer than four modes | Only triggers ~25% of the time | Include PLANNING, IMPLEMENTATION, GUIDANCE, REVIEW |
| Forgetting to `git add .claude-local/` | Skill exists locally but teammates never get it | Add and commit `.claude-local/skills/` |

---

## Quick Reference

### Create a New Project Skill

```bash
# From your project root
mkdir -p .claude-local/skills/{skill-name}
# Then use Write tool to create .claude-local/skills/{skill-name}/SKILL.md

# Stage and commit
git add .claude-local/skills/{skill-name}/
git commit -m "feat(skills): add {skill-name} project skill"
```

### Verify Skill Is Loadable

```bash
# Confirm it is NOT in .gitignore
grep ".claude-local" .gitignore   # Should return nothing

# Confirm flat structure (no nested directories)
ls .claude-local/skills/{skill-name}/   # Should show only SKILL.md

# Confirm name matches directory
grep "^name:" .claude-local/skills/{skill-name}/SKILL.md
# Output should be:  name: {skill-name}
```

### Availability Timeline

| Action | Availability |
|--------|-------------|
| Write SKILL.md + commit | Available on next `claude-ali` launch |
| No release or tarball needed | Confirmed — project skills sync via git |
| Teammates get it | On their next `git pull` + `claude-ali` launch |

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| Project template (full .claude-local/ reference) | `~/ali-ai/PROJECT_TEMPLATE.md` |
| Sync script (how overlay works) | `~/.claude/bin/sync-org-resources.sh` |
| Example project skill directory | `{any-project}/.claude-local/skills/` |
| Org skill examples to model style from | `~/ali-ai/skills/ali-bash/SKILL.md` |

---

**Document Version:** 1.0
**Last Updated:** 2026-02-26
