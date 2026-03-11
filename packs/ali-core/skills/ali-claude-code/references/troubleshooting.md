# Troubleshooting Claude Code Skills

Systematic guide for diagnosing and fixing common skill issues.

---

## Skill Won't Upload

### Error: "Could not find SKILL.md in uploaded folder"

**Cause:** File not named exactly SKILL.md (case-sensitive).

**Solution:**
- Rename to SKILL.md
- Verify with: `ls -la` should show SKILL.md exactly

### Error: "Invalid frontmatter"

**Cause:** YAML formatting issue in the frontmatter section.

**Common mistakes:**

```yaml
# Wrong - missing delimiters
name: my-skill
description: Does things

# Wrong - unclosed quotes
name: my-skill
description: "Does things

# Wrong - invalid YAML characters
name: my-skill
description: Does things & stuff < more

# Correct
---
name: my-skill
description: Does things and stuff
---
```

**Solutions:**
- Ensure `---` delimiters on separate lines before and after frontmatter
- Close all quotes properly
- Escape special characters or use `|` for multiline strings
- Use a YAML validator to check syntax

### Error: "Skill name already exists"

**Cause:** Another skill with the same name is already uploaded.

**Solution:**
- Choose a unique name
- Or remove the existing skill with that name before uploading

---

## Skill Doesn't Trigger

### Symptom: Skill never loads automatically

**Fix:** Revise your description field.

**Quick checklist:**
- Is it too generic? ("Helps with projects" won't work)
- Does it include trigger phrases users would actually say?
- Does it mention relevant file types or technologies if applicable?
- Does it cover all four modes (PLANNING, IMPLEMENTATION, GUIDANCE, REVIEW)?

**Debugging approach:**

Ask Claude: "When would you use the {skill-name} skill?"

Claude will quote the description back. Compare what Claude says with what you intended. Adjust description keywords based on gaps.

**Example:**

If your skill is for database migrations but Claude's response doesn't mention "migrations" or "database schema changes," add those phrases to your description.

---

## Skill Triggers Too Often

### Symptom: Skill loads for unrelated queries

**Solutions:**

### 1. Add negative triggers

Include explicit exclusions in your description:

```yaml
description: |
  Advanced data analysis for CSV files. Use for statistical modeling,
  regression, clustering. Do NOT use for simple data exploration
  (use data-viz skill instead).
```

### 2. Be more specific

Narrow your scope with precise language:

```yaml
# Too broad
description: Processes documents

# More specific
description: Processes PDF legal documents for contract review
```

### 3. Clarify scope with examples

```yaml
description: PayFlow payment processing for e-commerce. Use specifically
  for online payment workflows, not for general financial queries.
```

### 4. Add "When NOT to Use" section in SKILL.md body

Immediately after "When I Speak Up," add:

```markdown
## When NOT to Use This Skill

This skill does NOT activate for:
- {Exclusion 1} (use {alternative skill} instead)
- {Exclusion 2}
- {Exclusion 3}
```

---

## Instructions Not Followed

### Symptom: Skill loads but Claude doesn't follow instructions

**Common causes:**

### Cause 1: Instructions too verbose

**Problem:** Long paragraphs get skipped or skimmed.

**Solution:**
- Keep instructions concise
- Use bullet points and numbered lists
- Move detailed reference to separate files in `references/`

**Example:**

```markdown
# Bad - verbose
When creating items, you should make sure to validate that the title
is not empty and that it doesn't exceed 200 characters. You should also
verify that the item type is one of the allowed types before proceeding
with the creation process.

# Good - concise
Before creating items:
- Validate title is non-empty and ≤200 characters
- Verify item type is in allowed list
```

### Cause 2: Instructions buried

**Problem:** Critical instructions appear late in the file or in the middle of other content.

**Solution:**
- Put critical instructions at the top
- Use `## Important` or `## Critical` headers
- Repeat key points if needed

**Example:**

```markdown
## Critical Constraints

**IMPORTANT: Always validate user input before MCP calls.**

1. Check required fields are present
2. Validate data types
3. Verify values are in allowed ranges
```

### Cause 3: Ambiguous language

**Bad:**
```
Make sure to validate things properly
```

**Good:**
```
CRITICAL: Before calling create_project, verify:
- Project name is non-empty
- At least one team member assigned
- Start date is not in the past
```

### Advanced technique: Bundle validation scripts

For critical validations, consider bundling a script that performs the checks programmatically. Code is deterministic; language interpretation isn't.

**Example:**
```markdown
## Validation

Before creating items, run the bundled validation:

```bash
python validate_item.py --title "$TITLE" --type "$TYPE"
```

If exit code is 0, proceed with creation.
```

---

## MCP Connection Issues

### Symptom: Skill loads but MCP calls fail

**Checklist:**

### 1. Verify MCP server is connected

**Claude.ai:**
- Settings > Extensions > [Your Service]
- Should show "Connected" status

**Claude Code:**
```bash
claude mcp list
```
- Your server should appear in the list

### 2. Check authentication

- API keys valid and not expired
- Proper permissions/scopes configured
- OAuth tokens refreshed
- Environment variables set correctly

### 3. Test MCP independently

Ask Claude to call MCP directly (without skill):

```
"Use [Service] MCP to fetch my projects"
```

If this fails, the issue is with MCP configuration, not the skill.

### 4. Verify tool names

- Skill references correct MCP tool names
- Check MCP server documentation for exact names
- Tool names are case-sensitive
- Use `claude mcp list` to see available tools

### 5. Check error messages

MCP errors often include helpful details:
- "Tool not found" → Check tool name spelling
- "Authentication failed" → Check credentials
- "Rate limit exceeded" → Add rate limiting guidance to skill

---

## Large Context Issues

### Symptom: Skill seems slow or responses degraded

**Causes:**
- Skill content too large
- Too many skills enabled simultaneously
- All content loaded upfront instead of using progressive disclosure

**Solutions:**

### 1. Optimize SKILL.md size

Target: Under 5,000 words for SKILL.md body

**How:**
- Move detailed docs to `references/*.md` files
- Use bullet points instead of paragraphs
- Link to external documentation instead of duplicating it

### 2. Reduce enabled skills

Disable skills you're not actively using:
- Keep only domain-relevant skills enabled
- Test with minimal skill set
- Re-enable as needed

### 3. Use 3-level progressive disclosure

**Level 1 (always loaded):** YAML frontmatter
- Name, description, basic metadata

**Level 2 (auto-loaded when relevant):** SKILL.md body
- Core instructions, patterns, anti-patterns
- Target: <5,000 words

**Level 3 (loaded on demand):** references/*.md
- Detailed documentation
- Examples and tutorials
- Reference materials

**Example structure:**
```
my-skill/
├── SKILL.md                    # <5,000 words
└── references/
    ├── api-reference.md        # Detailed API docs
    ├── examples.md             # Usage examples
    └── troubleshooting.md      # This guide
```

### 4. Check for duplicate content

- Don't repeat the same information in multiple places
- Don't duplicate external documentation
- Link to sources instead of copying

---

## Debugging Checklist

Quick-reference table for common issues:

| Symptom | First Thing to Check | Section |
|---------|---------------------|---------|
| Won't upload | SKILL.md filename and frontmatter | [Skill Won't Upload](#skill-wont-upload) |
| Never triggers | Description field keywords and four-mode triggers | [Skill Doesn't Trigger](#skill-doesnt-trigger) |
| Triggers too often | Missing negative triggers | [Skill Triggers Too Often](#skill-triggers-too-often) |
| Ignores instructions | Instruction placement and verbosity | [Instructions Not Followed](#instructions-not-followed) |
| MCP errors | Connection status and tool names | [MCP Connection Issues](#mcp-connection-issues) |
| Slow/degraded | File sizes and skill count | [Large Context Issues](#large-context-issues) |

---

## Iterative Debugging Process

1. **Identify the symptom** - Use the debugging checklist table
2. **Check the obvious** - Follow the "First Thing to Check" column
3. **Read the relevant section** - Click the link for detailed guidance
4. **Make one change at a time** - Don't change multiple things simultaneously
5. **Test after each change** - Verify the fix worked
6. **Document what worked** - Note the solution for future reference

---

**Document Version:** 1.0
**Last Updated:** 2026-02-16
**Maintained By:** ALI AI Team
