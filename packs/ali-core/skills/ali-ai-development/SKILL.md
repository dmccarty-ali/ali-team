---
name: ali-ai-development
description: |
  AI development patterns with focus on Claude and Anthropic APIs. Use when:

  PLANNING: Designing AI-powered features, planning agent architectures,
  considering model selection, evaluating AI integration approaches

  IMPLEMENTATION: Building Claude API integrations, implementing tool use,
  creating prompts, building agents, handling streaming responses

  GUIDANCE: Asking about prompt engineering, model capabilities, cost optimization,
  best practices for AI development

  REVIEW: Checking prompt quality, reviewing AI integration code, validating
  cost efficiency, auditing agent designs

  Do NOT use for:
  - Claude Code configuration/setup (use ali-claude-code skill)
  - MCP server development (use ali-mcp skill)
  - Project-specific implementations (keep in project docs)
---

# AI Development

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing AI-powered features
- Planning multi-agent architectures
- Evaluating model selection (Opus vs Sonnet vs Haiku)
- Architecting AI integrations

**Implementation:**
- Building Claude API integrations
- Implementing tool use (function calling)
- Creating system prompts and user prompts
- Building AI agents and workflows
- Handling streaming responses

**Guidance/Best Practices:**
- Asking about prompt engineering patterns
- Asking about Claude capabilities
- Asking about cost optimization

**Review/Validation:**
- Reviewing prompt quality
- Checking AI integration patterns
- Validating cost efficiency
- Auditing agent architectures

---

## Key Principles

- **Model selection matters**: Use the right model for the task - Haiku for simple/fast, Sonnet for balanced, Opus for complex reasoning
- **Prompts are code**: Version control, test, and iterate on prompts like any other code
- **Context is precious**: Manage context window carefully - it's your primary resource constraint
- **Tools extend capabilities**: Use tool use for actions, structured output, and grounding
- **Stream for UX**: Stream responses for better user experience on longer outputs
- **Cache aggressively**: Use prompt caching for repeated context (system prompts, documents)
- **Fail gracefully**: AI outputs are probabilistic - handle edge cases and validate outputs
- **Cost = tokens**: Track token usage, optimize for cost without sacrificing quality

---

## Model Selection

### Claude Model Lineup

| Model | Best For | Speed | Cost | Context |
|-------|----------|-------|------|---------|
| **Opus 4** | Complex reasoning, nuanced analysis, creative work | Slower | $$$ | 200K |
| **Sonnet 4** | Balanced performance, most tasks, coding | Fast | $$ | 200K |
| **Haiku 3.5** | Simple tasks, classification, extraction, high volume | Fastest | $ | 200K |

### Selection Guidelines

```python
# Decision framework for model selection
def select_model(task: str) -> str:
    # Haiku: Simple, fast, cheap
    if task in ["classification", "extraction", "simple_qa", "formatting"]:
        return "claude-3-5-haiku-20241022"

    # Opus: Complex reasoning required
    if task in ["architecture", "ambiguous_requirements", "creative", "nuanced_analysis"]:
        return "claude-opus-4-20250514"

    # Sonnet: Default for most work
    return "claude-sonnet-4-20250514"
```

### Cost Comparison (per 1M tokens)

| Model | Input | Output |
|-------|-------|--------|
| Opus 4 | $15 | $75 |
| Sonnet 4 | $3 | $15 |
| Haiku 3.5 | $0.80 | $4 |

**Rule of thumb**: If you can get acceptable quality from Haiku, use it. The cost difference is 10-20x.

---

## Prompt Engineering

### Prompt Structure

```
[System Prompt]
- Role/persona
- Capabilities and constraints
- Output format requirements
- Examples (if few-shot)

[User Message]
- Context/background
- Specific task/question
- Any additional constraints
```

### Effective System Prompts

```python
# Good: Specific, constrained, actionable
system = """You are a code reviewer for a Python FastAPI application.

Your task:
- Review code for security vulnerabilities, bugs, and style issues
- Focus on OWASP Top 10 vulnerabilities
- Be concise - one sentence per issue

Output format:
- CRITICAL: [issue] at [location]
- WARNING: [issue] at [location]
- SUGGESTION: [improvement] at [location]

Do NOT:
- Explain basic Python syntax
- Suggest purely stylistic changes unless egregious
- Add disclaimers or caveats
"""

# Bad: Vague, no constraints
system = "You are a helpful code reviewer. Review the code and provide feedback."
```

### Few-Shot Learning

```python
# Provide examples for consistent output
system = """Extract entities from text in JSON format.

Examples:
Input: "John Smith works at Acme Corp in New York"
Output: {"people": ["John Smith"], "companies": ["Acme Corp"], "locations": ["New York"]}

Input: "The meeting between Apple and Microsoft is in Seattle"
Output: {"people": [], "companies": ["Apple", "Microsoft"], "locations": ["Seattle"]}
"""
```

### Chain of Thought

```python
# For complex reasoning, encourage step-by-step thinking
user_message = """Analyze this code for potential race conditions.

Think through this step by step:
1. Identify shared state
2. Find concurrent access points
3. Check for synchronization mechanisms
4. Evaluate potential race windows

Code:
```python
{code}
```
"""
```

### Output Formatting

```python
# XML tags for structured sections
user_message = """Analyze this architecture proposal.

Provide your analysis in the following format:
<summary>One paragraph overview</summary>
<strengths>
- Bullet points of strengths
</strengths>
<weaknesses>
- Bullet points of weaknesses
</weaknesses>
<recommendation>Your recommendation</recommendation>
"""
```

---

## Claude Code Integration

For Claude Code configuration patterns (skills, agents, settings), see the **ali-claude-code** skill.

This skill focuses on direct API integration and programmatic Claude usage.

---

## Anti-Patterns

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Opus for everything | 10x cost, slower | Route by complexity |
| No system prompt | Inconsistent behavior | Always define role/constraints |
| Ignoring stop_reason | Miss tool calls, truncation | Check and handle all stop reasons |
| Huge single prompts | Context limits, cost | Chunk and summarize |
| No output validation | Unreliable downstream | Validate format and content |
| Synchronous for long tasks | Poor UX, timeouts | Stream responses |
| Hardcoded prompts | Can't iterate | Store prompts as versioned files |
| No token tracking | Cost surprises | Monitor usage per request |

---

## Detailed Reference

For detailed implementation patterns, consult the reference files covering:

### Claude API Patterns
**File:** references/claude-api-patterns.md

Covers:
- Basic message API boilerplate
- System prompt examples
- Streaming response implementation
- Multi-turn conversation management
- Vision API (multi-modal) usage
- Document analysis patterns

### Tool Use Patterns
**File:** references/tool-use-patterns.md

Covers:
- Tool schema definition with JSON Schema
- Tool call handling and execution loops
- Structured output via tool use
- Multi-tool coordination patterns
- Best practices for tool design

### Cost Optimization
**File:** references/cost-optimization.md

Covers:
- Prompt caching implementation (90% input cost reduction)
- Token estimation and truncation utilities
- Batching strategies for high-volume processing
- Model routing based on complexity
- Token usage tracking and cost reporting

### Agent Patterns
**File:** references/agent-patterns.md

Covers:
- Single agent with tools implementation
- Multi-agent orchestration patterns
- Delegation to specialized sub-agents
- Stateful agent patterns
- Feedback loop and self-correction
- ReAct (Reasoning + Acting) pattern

### Error Handling
**File:** references/error-handling.md

Covers:
- API error handling with retry logic
- Output validation and quality checks
- Graceful degradation strategies
- Circuit breaker pattern for resilience
- Input sanitization for security
- Timeout handling patterns
- Logging best practices

### Testing Patterns
**File:** references/testing-patterns.md

Covers:
- Prompt testing for consistency and refusal
- Mocking Claude API for unit tests
- Snapshot testing for prompt outputs
- Integration testing with real API
- Evaluation frameworks for prompt quality
- A/B testing prompts
- Regression testing strategies

---

## References

- [Anthropic API Documentation](https://docs.anthropic.com/)
- [Claude Model Card](https://www.anthropic.com/claude)
- [Prompt Engineering Guide](https://docs.anthropic.com/claude/docs/prompt-engineering)
- [Tool Use Documentation](https://docs.anthropic.com/claude/docs/tool-use)
- [Claude Code Documentation](https://docs.anthropic.com/claude-code)

---

**Document Version:** 3.0
**Last Updated:** 2026-02-16
**Maintained By:** ALI AI Team

**Change Log:**
- v3.0 (2026-02-16): Progressive disclosure refactoring - moved implementation details to references/, condensed to core decision frameworks
- v2.1 (2026-02-16): Removed project-specific Bedrock implementation code (650 lines) and duplicate Claude Code content
- v2.0 (2026-01-02): Added production implementation patterns
- v1.0 (2025-12-14): Initial Claude API patterns
