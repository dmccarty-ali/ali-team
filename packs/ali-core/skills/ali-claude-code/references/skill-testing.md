# Testing Skills

Testing ensures skills trigger correctly, produce quality output, and improve on baseline.

## 1. Triggering Tests

**Goal:** Ensure your skill loads at the right times and does not load at wrong times.

### Test Case Structure

For each skill, define:

**Should trigger (positive tests):**
- 3-5 obvious task descriptions
- 3-5 paraphrased/indirect requests
- 1-2 edge cases that should still trigger

**Should NOT trigger (negative tests):**
- 3-5 unrelated topics
- 2-3 topics handled by competing skills
- 1-2 edge cases that should NOT trigger

### Example: ali-claude-code Skill

Should trigger:
- "Help me set up CLAUDE.md for my project"
- "I want to create a new skill for our team"
- "How should I structure my hooks configuration?"
- "Review my agent definition for quality"
- "What's the best way to manage Claude Code permissions?"

Should NOT trigger:
- "Write a Python function to sort a list"
- "Help me debug my React component"
- "What's the weather today?"
- "Set up an MCP server" (ali-mcp skill territory)

### Debugging Trigger Issues

Ask Claude: "When would you use the {skill-name} skill?"
- Claude will quote the description back
- Compare what Claude says with what you intended
- Adjust description keywords based on gaps

## 2. Functional Tests

**Goal:** Verify the skill produces correct, useful output.

### Test Methodology

For each use case defined in your skill:
1. Provide a realistic prompt that should trigger the skill
2. Verify the output follows the skill's instructions
3. Check that patterns, anti-patterns, and references are applied correctly
4. Confirm error handling works for edge cases

### Example Test

```
Test: Create a new skill using ali-claude-code guidance
Given: User asks "Help me create a skill for database migrations"
When: ali-claude-code skill activates
Then:
  - Response references SKILL_TEMPLATE.md structure
  - Four-mode trigger pattern is recommended
  - Size guidelines are mentioned
  - Progressive disclosure is suggested for large skills
  - No contradictions with existing reference material
```

### Functional Test Checklist

- [ ] Core use case produces correct output
- [ ] Edge cases handled gracefully
- [ ] Error scenarios provide helpful guidance
- [ ] Output follows skill's own patterns/anti-patterns
- [ ] References are accurate and accessible

## 3. Performance Comparison

**Goal:** Demonstrate measurable improvement over baseline (no skill).

### Baseline Methodology

Run the same task twice:
1. **Without skill:** Disable the skill, provide only the user prompt
2. **With skill:** Enable the skill, provide the same user prompt

### Metrics to Compare

| Metric | Without Skill | With Skill | Target |
|--------|---------------|------------|--------|
| Accuracy | % correct recommendations | % correct | Higher |
| Completeness | Topics covered | Topics covered | More complete |
| Back-and-forth | Messages to completion | Messages | Fewer |
| Token usage | Total tokens | Total tokens | Lower or comparable |
| Time to useful answer | Seconds/messages | Seconds/messages | Faster |

### Example Comparison

Task: "Help me structure a CLAUDE.md for a new Python project"

Without ali-claude-code skill:
- Generic advice about project documentation
- No mention of hierarchy or precedence
- No pruning guidance
- 4 back-and-forth messages to get specific

With ali-claude-code skill:
- Immediately references CLAUDE.md hierarchy
- Provides size target (<200 lines)
- Mentions quarterly pruning
- Suggests import syntax for shared docs
- 1 message to get specific, actionable guidance

## Testing Tools

### skill-creator Skill

If available, use skill-creator to:
- Generate test cases based on skill description
- Flag common issues (vague descriptions, missing triggers)
- Identify potential over/under-triggering risks
- Suggest improvements

### Iterative Improvement

After real-world usage:
1. Collect examples where the skill under-triggered (missed activation)
2. Collect examples where the skill over-triggered (false activation)
3. Collect examples where output was wrong or incomplete
4. Use findings to refine description, triggers, and content
