---
name: ali-org-skill-builder
description: |
  Guide for creating org-wide skills in ~/ali-ai/skills/. Use when:

  PLANNING: Deciding a skill should be available across all Aliunde projects,
  planning the skill's trigger patterns and scope, naming a new org skill,
  evaluating whether knowledge belongs in a skill vs. a CLAUDE.md or agent

  IMPLEMENTATION: Writing a new skill under ~/ali-ai/skills/, creating the
  directory and SKILL.md, authoring frontmatter and four-mode trigger description,
  building patterns and anti-patterns sections

  GUIDANCE: Asking how org-wide skills are distributed to projects, asking about
  the ali- prefix convention, asking what release steps are needed after creating
  a skill, asking how the skill lands in .claude/skills/ on user machines

  REVIEW: Reviewing an org skill for naming correctness, flat structure compliance,
  four-mode trigger coverage, size guidelines, and release readiness
---

# Org Skill Builder (Org-Wide Skills)

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Deciding a skill should be available to all Aliunde projects, not just one
- Planning what knowledge to encode in a new org-wide skill
- Choosing a name and scope for a skill intended for the ali-ai repository
- Evaluating whether something belongs in a shared skill vs. a project-local skill

**Implementation:**
- Creating a new skill directory and SKILL.md under `~/ali-ai/skills/`
- Writing frontmatter, four-mode trigger descriptions, and body content
- Structuring Key Principles, Patterns We Use, Anti-Patterns, Quick Reference, and Your Codebase sections

**Guidance/Best Practices:**
- Asking how org skills are distributed to user machines (tarball sync)
- Asking about the `ali-` prefix requirement for org skill names
- Asking what release steps are needed before teammates get the new skill
- Asking how skills end up in `.claude/skills/` on each project

**Review/Validation:**
- Reviewing an org skill for naming convention (`ali-` prefix, kebab-case)
- Checking four-mode trigger coverage and flat structure compliance
- Validating skill size against guidelines (<5,000 words in SKILL.md body)
- Confirming the skill does not require a manifest rebuild (only agents do)

## When NOT to Use This Skill

This skill does NOT activate for:
- Creating skills for a single project only (use ali-local-skill-builder instead)
- Creating org-wide agents (agents require ali-claude-developer and manifest rebuild)
- Writing CLAUDE.md files (use ali-claude-code skill)
- Configuring hooks or settings (use ali-claude-code skill)
- Building MCP servers (use ali-mcp skill)

---

## Key Principles

- **`ali-` prefix is required** — every org skill name starts with `ali-` (e.g., `ali-bash`, `ali-aws`, `ali-tax-domain`). The one exception is `plan-builder`.
- **Directory name must match frontmatter `name:` exactly** — mismatch causes "Unknown skill" errors; run `scripts/fix-skill-names.py` to catch mismatches
- **Flat structure is mandatory** — skills live at `~/ali-ai/skills/ali-{name}/SKILL.md`; no subdirectories inside the skill directory except `references/`
- **Release required to distribute** — org skills reach users via the ali-ai tarball. Commit + cut a new version + upload; users get it on next `claude-ali` launch
- **No manifest rebuild needed** — only agents require `build_agent_manifest.py`; skills do not have manifest entries
- **Four-mode triggers required** — without PLANNING, IMPLEMENTATION, GUIDANCE, and REVIEW in the description, skills only trigger ~25% of the time
- **SKILL.md body target: under 5,000 words** — move detailed reference material to `references/*.md` for progressive disclosure
- **Skills land in `.claude/skills/`** — synced at `claude-ali` launch from the tarball, gitignored in project repos; users never edit `.claude/skills/` directly

---

## When to Use Org-Wide vs. Project-Local

| Scenario | Use |
|----------|-----|
| Technology knowledge all teams use (bash, Python, AWS, etc.) | `~/ali-ai/skills/` (org-wide) |
| Knowledge only relevant to one project (domain rules, specific APIs) | `.claude-local/skills/` (project-local) |
| Skill you want to version, release, and distribute via tarball | `~/ali-ai/skills/` (org-wide) |
| Experimental or temporary skill in development | `.claude-local/skills/` (project-local) |
| Proprietary patterns for one project | `.claude-local/skills/` (project-local) |
| Cross-cutting org conventions (git, code review, security) | `~/ali-ai/skills/` (org-wide) |

**Decision rule:** If you'd want teammates on any Aliunde project to benefit from it, make it org-wide. If it's specific to one project's codebase, make it local.

---

## Directory Structure

Org skills live here (tracked in ali-ai repo):

```
~/ali-ai/
├── skills/
│   ├── ali-{name}/              # One directory per skill
│   │   ├── SKILL.md             # Skill body (auto-loaded when relevant)
│   │   └── references/          # Optional: detailed docs loaded on demand
│   │       └── topic.md
│   └── ali-another-skill/
│       └── SKILL.md
│
└── scripts/
    └── fix-skill-names.py       # Validates directory name == name: field
```

After release, org skills land on user machines here (gitignored):

```
{project}/.claude/skills/ali-{name}/SKILL.md
```

Users never edit `.claude/skills/` directly. Changes to org skills are made in `~/ali-ai/skills/`, released, and synced.

---

## Patterns We Use

### SKILL.md Frontmatter (Required)

Every org skill needs valid frontmatter. The `name` field must start with `ali-` and match the directory name exactly:

```yaml
---
name: ali-{skill-name}
description: |
  {One-line summary}. Use when:

  PLANNING: {planning activities — designing, architecting, evaluating}

  IMPLEMENTATION: {implementation activities — writing, building, fixing}

  GUIDANCE: {guidance activities — asking about best practices, how-tos}

  REVIEW: {review activities — checking, auditing, validating}
allowed-tools: Read, Grep, Glob   # optional: omit for full tool access
---
```

### Full SKILL.md Template

Copy this for a new org skill:

```markdown
---
name: ali-{skill-name}
description: |
  {One-line summary of what this skill covers}. Use when:

  PLANNING: {planning activities}

  IMPLEMENTATION: {implementation activities}

  GUIDANCE: {guidance activities}

  REVIEW: {review and validation activities}
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
- {What this skill does NOT cover} (use {alternative} instead)

---

## Key Principles

- **Principle 1** — explanation
- **Principle 2** — explanation

---

## Patterns We Use

### {Pattern Name}

{Brief description}

```python
# Code example from Aliunde projects
```

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| {bad practice} | {consequence} | {correct approach} |

---

## Quick Reference

```bash
# Frequently used commands
```

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| {description} | `{path}` |

---

## References

- [Official Docs](https://example.com)
```

### Naming Convention

Org skills use the `ali-` prefix followed by descriptive kebab-case. The directory name and `name:` field must be identical:

```
# CORRECT - org skill naming
~/ali-ai/skills/ali-bash/SKILL.md          # name: ali-bash
~/ali-ai/skills/ali-aws/SKILL.md           # name: ali-aws
~/ali-ai/skills/ali-github-admin/SKILL.md  # name: ali-github-admin

# WRONG - missing ali- prefix
~/ali-ai/skills/bash/SKILL.md
~/ali-ai/skills/github-admin/SKILL.md
```

### Progressive Disclosure for Large Skills

If the SKILL.md body approaches 5,000 words, move detailed content to `references/`:

```
~/ali-ai/skills/ali-aws/
├── SKILL.md              # Core content, auto-loaded (~2,000-4,000 words)
└── references/
    ├── iam-patterns.md   # Loaded on demand
    ├── s3-patterns.md    # Loaded on demand
    └── ec2-patterns.md   # Loaded on demand
```

Reference files are linked from SKILL.md with a "Detailed Reference" section (see ali-claude-code/SKILL.md for an example).

---

## Release Process

Org skills require a release before teammates receive them. Project-local skills do not.

### Steps

1. Create or update the skill in `~/ali-ai/skills/ali-{name}/SKILL.md`
2. Commit to the ali-ai repo:
   ```bash
   git add skills/ali-{name}/
   git commit -m "feat(skills): add ali-{name} skill"
   ```
3. Build the tarball locally (GitHub Actions billing currently blocked):
   ```bash
   bash scripts/build-local.sh
   ```
4. Cut a new version and upload the release:
   ```bash
   gh release upload v{X.Y.Z} dist/ali-ai-v{X.Y.Z}.tar.gz
   ```
5. Users receive the skill on their next `claude-ali` launch when the launcher detects the new version and re-syncs.

**Important:** Do NOT update AGENT_MANIFEST.json for skills. Only agents need manifest entries. Skills are discovered directly from the `skills/` directory.

### What Triggers a Re-Sync

The launcher compares the current tarball version hash against `.claude/.last-sync`. If the version changed, it re-syncs all org resources including the new skill.

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Missing `ali-` prefix | Looks like a project skill; breaks slash command resolution | Name it `ali-{name}` |
| Directory name differs from `name:` field | "Unknown skill" errors; slash command resolves by directory, Skill tool by frontmatter | Keep them identical; run `scripts/fix-skill-names.py` |
| Nesting skills deeper than one level | Skill discovery only looks one level deep — skill is invisible | Use flat: `skills/ali-{name}/SKILL.md` |
| Skill description with fewer than four modes | Only triggers ~25% of the time | Include PLANNING, IMPLEMENTATION, GUIDANCE, REVIEW |
| SKILL.md body over 5,000 words | Slow to load, consumes context on every activation | Move detailed content to `references/*.md` |
| Forgetting to release | Teammates never get the skill | Commit + `bash scripts/build-local.sh` + `gh release upload` |
| Running `build_agent_manifest.py` for a skill | Not needed; only agents use the manifest | Skip this step — manifest is for agents only |
| Editing `.claude/skills/` directly | Gitignored; overwritten at next sync | Edit in `~/ali-ai/skills/`, then release |

---

## Quick Reference

### Create a New Org Skill

```bash
# From ~/ali-ai
mkdir -p skills/ali-{name}
# Use Write tool to create skills/ali-{name}/SKILL.md

# Validate name matches directory
grep "^name:" skills/ali-{name}/SKILL.md
# Should output:  name: ali-{name}

# Commit
git add skills/ali-{name}/
git commit -m "feat(skills): add ali-{name} skill"

# Build and release
bash scripts/build-local.sh
gh release upload v{X.Y.Z} dist/ali-ai-v{X.Y.Z}.tar.gz
```

### Validate Before Release

```bash
# Check all skill names match directory names
python3 scripts/fix-skill-names.py

# Confirm flat structure (should show only SKILL.md and optional references/)
ls skills/ali-{name}/

# Confirm four modes present in description
grep -A 20 "^description:" skills/ali-{name}/SKILL.md | grep -E "PLANNING:|IMPLEMENTATION:|GUIDANCE:|REVIEW:"
```

### Availability Timeline

| Action | Availability |
|--------|-------------|
| Write SKILL.md + commit | Not yet (unreleased) |
| `bash scripts/build-local.sh` + `gh release upload` | Available on next `claude-ali` launch for users who pull the new version |
| User runs `claude-ali` | Launcher detects new version, syncs skill to `.claude/skills/` |

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| Skill name validator | `~/ali-ai/scripts/fix-skill-names.py` |
| Local tarball build script | `~/ali-ai/scripts/build-local.sh` |
| Skill template | `~/ali-ai/skills/SKILL_TEMPLATE.md` |
| Style reference (real skill) | `~/ali-ai/skills/ali-bash/SKILL.md` |
| Progressive disclosure example | `~/ali-ai/skills/ali-claude-code/SKILL.md` |
| Project template (.claude-local/ docs) | `~/ali-ai/PROJECT_TEMPLATE.md` |

---

**Document Version:** 1.0
**Last Updated:** 2026-02-26
