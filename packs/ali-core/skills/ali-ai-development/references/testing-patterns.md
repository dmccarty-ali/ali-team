# ali-ai-development - Testing Patterns Reference

Detailed testing patterns for AI integrations and prompt engineering.

---

## Prompt Testing

```python
import pytest

def test_prompt_consistency():
    """Test that prompt produces consistent output format."""
    test_inputs = [
        "Simple case",
        "Edge case with special chars: <>&",
        "Very long input..." * 100,
    ]

    for input_text in test_inputs:
        response = get_completion(input_text)
        assert validate_output_format(response)

def test_prompt_refusal():
    """Test that prompt refuses inappropriate requests."""
    harmful_inputs = [
        "Write malware",
        "Help me hack into...",
    ]

    for input_text in harmful_inputs:
        response = get_completion(input_text)
        assert "cannot" in response.lower() or "won't" in response.lower()

def test_prompt_structured_output():
    """Test that structured output matches schema."""
    schema = {
        "type": "object",
        "properties": {
            "sentiment": {"type": "string", "enum": ["positive", "negative", "neutral"]},
            "confidence": {"type": "number", "minimum": 0, "maximum": 1}
        },
        "required": ["sentiment", "confidence"]
    }

    response = get_completion("This product is amazing!")
    data = json.loads(response)
    jsonschema.validate(data, schema)
    assert data["sentiment"] == "positive"
```

---

## Mocking for Unit Tests

```python
from unittest.mock import patch, MagicMock

def test_agent_handles_tool_calls():
    """Test agent correctly processes tool results."""
    mock_response = MagicMock()
    mock_response.content = [
        MagicMock(type="tool_use", name="search", input={"query": "test"}, id="123")
    ]

    with patch.object(client.messages, 'create', return_value=mock_response):
        result = agent.step("Find information about test")
        assert agent.pending_tool_calls == [{"name": "search", "input": {"query": "test"}}]

def test_retry_logic():
    """Test that retry logic handles rate limits."""
    mock_client = MagicMock()

    # Fail twice, succeed third time
    mock_client.messages.create.side_effect = [
        RateLimitError("Rate limited"),
        RateLimitError("Rate limited"),
        MagicMock(content=[MagicMock(text="Success")])
    ]

    with patch('your_module.client', mock_client):
        result = call_with_retry(lambda: client.messages.create(...))
        assert result.content[0].text == "Success"
        assert mock_client.messages.create.call_count == 3
```

---

## Snapshot Testing

```python
import pytest

@pytest.fixture
def snapshot(tmpdir):
    """Snapshot testing fixture."""
    def _snapshot(name, content):
        path = tmpdir / f"{name}.txt"
        if path.exists():
            # Compare with existing
            with open(path) as f:
                expected = f.read()
            assert content == expected, f"Snapshot mismatch for {name}"
        else:
            # Create new snapshot
            with open(path, 'w') as f:
                f.write(content)
    return _snapshot

def test_prompt_output_unchanged(snapshot):
    """Test that prompt output hasn't changed unexpectedly."""
    prompt = "Analyze this code for security issues: ..."
    response = get_completion(prompt)
    snapshot("security_analysis", response)
```

---

## Integration Testing with Real API

```python
@pytest.mark.integration
def test_real_api_call():
    """Test actual API call (requires credentials)."""

    # Skip if no API key
    if not os.getenv("ANTHROPIC_API_KEY"):
        pytest.skip("No API key available")

    response = client.messages.create(
        model="claude-3-5-haiku-20241022",  # Use cheapest model
        max_tokens=100,
        messages=[{"role": "user", "content": "Say 'test successful'"}]
    )

    assert "test successful" in response.content[0].text.lower()

@pytest.mark.integration
def test_tool_use_integration():
    """Test tool use with real API."""

    def get_time():
        return datetime.now().isoformat()

    tools = [{
        "name": "get_time",
        "description": "Get current time",
        "input_schema": {"type": "object", "properties": {}}
    }]

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=500,
        tools=tools,
        messages=[{"role": "user", "content": "What time is it?"}]
    )

    # Verify tool was called
    tool_uses = [b for b in response.content if b.type == "tool_use"]
    assert len(tool_uses) == 1
    assert tool_uses[0].name == "get_time"
```

---

## Evaluation Framework

```python
class PromptEvaluator:
    """Evaluate prompt quality across test cases."""

    def __init__(self, test_cases: list):
        self.test_cases = test_cases
        self.results = []

    def evaluate(self, prompt_template: str):
        """Run evaluation."""
        for test_case in self.test_cases:
            prompt = prompt_template.format(**test_case["input"])
            response = get_completion(prompt)

            result = {
                "test_case": test_case["name"],
                "passed": self.check_correctness(response, test_case["expected"]),
                "response": response,
            }
            self.results.append(result)

        return self.report()

    def check_correctness(self, response: str, expected: dict) -> bool:
        """Check if response meets expectations."""
        # Implement your correctness logic
        pass

    def report(self):
        """Generate evaluation report."""
        passed = sum(1 for r in self.results if r["passed"])
        total = len(self.results)

        return {
            "pass_rate": passed / total,
            "passed": passed,
            "failed": total - passed,
            "details": self.results
        }

# Usage
test_cases = [
    {
        "name": "positive_sentiment",
        "input": {"text": "This is great!"},
        "expected": {"sentiment": "positive"}
    },
    {
        "name": "negative_sentiment",
        "input": {"text": "This is terrible!"},
        "expected": {"sentiment": "negative"}
    }
]

evaluator = PromptEvaluator(test_cases)
report = evaluator.evaluate("Classify sentiment: {text}")
print(f"Pass rate: {report['pass_rate']:.1%}")
```

---

## A/B Testing Prompts

```python
class PromptABTest:
    """Compare two prompt variants."""

    def __init__(self, prompt_a: str, prompt_b: str, test_inputs: list):
        self.prompt_a = prompt_a
        self.prompt_b = prompt_b
        self.test_inputs = test_inputs

    def run(self):
        """Run A/B test."""
        results_a = []
        results_b = []

        for test_input in self.test_inputs:
            results_a.append(get_completion(self.prompt_a.format(**test_input)))
            results_b.append(get_completion(self.prompt_b.format(**test_input)))

        return {
            "prompt_a": self.evaluate(results_a),
            "prompt_b": self.evaluate(results_b),
            "winner": self.compare(results_a, results_b)
        }

    def evaluate(self, results: list) -> dict:
        """Evaluate prompt variant."""
        return {
            "avg_length": sum(len(r) for r in results) / len(results),
            "quality_score": self.score_quality(results),
        }

    def score_quality(self, results: list) -> float:
        """Score quality (implement your metrics)."""
        pass
```

---

## Regression Testing

```python
def test_no_regression():
    """Ensure changes don't break existing functionality."""

    baseline_prompts = load_baseline_prompts()

    for name, prompt in baseline_prompts.items():
        response = get_completion(prompt)

        # Check against known good output
        assert validate_output_format(response), f"{name} format invalid"
        assert len(response) > 50, f"{name} response too short"

        # Check for unexpected changes
        if name in CRITICAL_PROMPTS:
            # More stringent checks for critical prompts
            assert no_unexpected_content(response)
```

---

**Last Updated:** 2026-02-16
