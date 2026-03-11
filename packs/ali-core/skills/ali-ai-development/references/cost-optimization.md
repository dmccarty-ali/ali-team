# ali-ai-development - Cost Optimization Reference

Detailed cost optimization techniques for Claude API usage.

---

## Prompt Caching

```python
# Cache repeated context (system prompts, documents)
# Cached tokens cost 90% less on input

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": large_system_prompt,
            "cache_control": {"type": "ephemeral"}  # Cache this
        }
    ],
    messages=[{"role": "user", "content": "Question about the document..."}]
)

# Check cache usage
print(f"Cache read: {response.usage.cache_read_input_tokens}")
print(f"Cache write: {response.usage.cache_creation_input_tokens}")
```

### Caching Multi-Part System Prompts

```python
# Cache different sections with different lifetimes
system = [
    {
        "type": "text",
        "text": "You are a code reviewer...",
        # No cache - short, changes frequently
    },
    {
        "type": "text",
        "text": read_file("coding_standards.md"),  # Large, stable
        "cache_control": {"type": "ephemeral"}
    },
    {
        "type": "text",
        "text": read_file("project_context.md"),  # Large, stable
        "cache_control": {"type": "ephemeral"}
    }
]
```

### Document Caching Pattern

```python
# Cache large documents for repeated queries
document = read_large_document()

# First request creates cache
response1 = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    system=[{
        "type": "text",
        "text": f"Document to analyze:\n\n{document}",
        "cache_control": {"type": "ephemeral"}
    }],
    messages=[{"role": "user", "content": "What are the key points?"}]
)

# Subsequent requests use cache (90% cheaper input)
response2 = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    system=[{
        "type": "text",
        "text": f"Document to analyze:\n\n{document}",
        "cache_control": {"type": "ephemeral"}
    }],
    messages=[{"role": "user", "content": "What are the risks mentioned?"}]
)
```

---

## Token Management

```python
# Estimate tokens before sending
def estimate_tokens(text: str) -> int:
    """Rough estimate: ~4 chars per token for English."""
    return len(text) // 4

# Truncate context if needed
def truncate_to_tokens(text: str, max_tokens: int) -> str:
    """Truncate text to approximate token limit."""
    max_chars = max_tokens * 4
    if len(text) > max_chars:
        return text[:max_chars] + "\n[truncated]"
    return text

# Smart truncation - keep beginning and end
def smart_truncate(text: str, max_tokens: int) -> str:
    """Truncate middle, keep start and end."""
    estimated = estimate_tokens(text)
    if estimated <= max_tokens:
        return text

    # Keep 40% from start, 40% from end
    keep_tokens = int(max_tokens * 0.4)
    keep_chars = keep_tokens * 4

    start = text[:keep_chars]
    end = text[-keep_chars:]

    return f"{start}\n\n... [middle truncated] ...\n\n{end}"
```

---

## Batching

```python
# Batch multiple items in single request when possible
items_to_process = ["item1", "item2", "item3"]

# Instead of 3 separate calls:
response = client.messages.create(
    model="claude-3-5-haiku-20241022",  # Use Haiku for batch processing
    max_tokens=2048,
    messages=[{
        "role": "user",
        "content": f"""Process each item and return results as JSON array:

Items:
{json.dumps(items_to_process)}

Return: [{{"item": "...", "result": "..."}}, ...]"""
    }]
)

# Parse batch results
results = json.loads(response.content[0].text)
```

---

## Model Routing

```python
# Route to appropriate model based on task complexity
def route_request(task: dict) -> str:
    """Route to most cost-effective model."""

    complexity = estimate_complexity(task)

    if complexity < 0.3:
        return "claude-3-5-haiku-20241022"  # Simple tasks
    elif complexity < 0.7:
        return "claude-sonnet-4-20250514"   # Most tasks
    else:
        return "claude-opus-4-20250514"     # Complex reasoning

def estimate_complexity(task: dict) -> float:
    """Estimate task complexity (0.0 to 1.0)."""
    signals = 0

    # Simple signals → Haiku
    if task.get("type") in ["classification", "extraction", "formatting"]:
        return 0.2

    # Complex signals → Opus
    if any(word in task.get("description", "").lower()
           for word in ["architecture", "design", "ambiguous", "creative"]):
        signals += 0.5

    # Length signal
    if len(task.get("context", "")) > 10000:
        signals += 0.2

    return min(1.0, signals)
```

---

## Token Usage Tracking

```python
class TokenTracker:
    """Track token usage and costs."""

    def __init__(self):
        self.usage = []

    def track(self, response):
        """Track usage from API response."""
        usage = {
            "timestamp": datetime.now().isoformat(),
            "model": response.model,
            "input_tokens": response.usage.input_tokens,
            "output_tokens": response.usage.output_tokens,
            "cache_read": getattr(response.usage, "cache_read_input_tokens", 0),
            "cache_write": getattr(response.usage, "cache_creation_input_tokens", 0),
        }

        # Calculate cost
        costs = {
            "claude-opus-4-20250514": {"input": 15, "output": 75},
            "claude-sonnet-4-20250514": {"input": 3, "output": 15},
            "claude-3-5-haiku-20241022": {"input": 0.8, "output": 4},
        }

        model_costs = costs[response.model]
        usage["cost"] = (
            (usage["input_tokens"] / 1_000_000) * model_costs["input"] +
            (usage["output_tokens"] / 1_000_000) * model_costs["output"] +
            (usage["cache_read"] / 1_000_000) * (model_costs["input"] * 0.1) +  # 90% discount
            (usage["cache_write"] / 1_000_000) * (model_costs["input"] * 1.25)  # 25% premium
        )

        self.usage.append(usage)
        return usage

    def report(self):
        """Generate cost report."""
        total_cost = sum(u["cost"] for u in self.usage)
        total_tokens = sum(u["input_tokens"] + u["output_tokens"] for u in self.usage)

        return {
            "total_requests": len(self.usage),
            "total_tokens": total_tokens,
            "total_cost": f"${total_cost:.4f}",
            "by_model": self._by_model_report()
        }
```

---

**Last Updated:** 2026-02-16
