# ali-ai-development - Error Handling Reference

Detailed error handling and resilience patterns for Claude API integrations.

---

## API Error Handling with Retry

```python
from anthropic import APIError, RateLimitError, APITimeoutError
import time

def call_with_retry(func, max_retries=3):
    """Call API with exponential backoff."""
    for attempt in range(max_retries):
        try:
            return func()
        except RateLimitError:
            wait = 2 ** attempt
            print(f"Rate limited, waiting {wait}s...")
            time.sleep(wait)
        except APITimeoutError:
            if attempt == max_retries - 1:
                raise
            print(f"Timeout on attempt {attempt + 1}, retrying...")
            continue
        except APIError as e:
            # Log and re-raise non-retryable errors
            logger.error(f"API error: {e}")
            raise

    raise RuntimeError(f"Failed after {max_retries} retries")
```

---

## Output Validation

```python
import json
import jsonschema
import re

def validate_json_output(response: str, schema: dict) -> dict:
    """Validate Claude's JSON output against schema."""
    try:
        data = json.loads(response)
        jsonschema.validate(data, schema)
        return data
    except json.JSONDecodeError:
        # Try to extract JSON from response
        match = re.search(r'\{.*\}', response, re.DOTALL)
        if match:
            return json.loads(match.group())
        raise ValueError("No valid JSON in response")
    except jsonschema.ValidationError as e:
        raise ValueError(f"Schema validation failed: {e.message}")
```

---

## Graceful Degradation

```python
def get_completion_with_fallback(prompt: str) -> str:
    """Try multiple models with fallback."""

    models = [
        "claude-sonnet-4-20250514",
        "claude-3-5-haiku-20241022",  # Fallback to cheaper model
    ]

    last_error = None
    for model in models:
        try:
            response = client.messages.create(
                model=model,
                max_tokens=1024,
                messages=[{"role": "user", "content": prompt}]
            )
            return response.content[0].text
        except Exception as e:
            last_error = e
            logger.warning(f"Model {model} failed: {e}")
            continue

    raise RuntimeError(f"All models failed. Last error: {last_error}")
```

---

## Circuit Breaker Pattern

```python
from datetime import datetime, timedelta

class CircuitBreaker:
    """Prevent cascading failures from API issues."""

    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failures = 0
        self.last_failure_time = None
        self.state = "closed"  # closed, open, half-open

    def call(self, func):
        """Execute function with circuit breaker protection."""

        # Check if circuit is open
        if self.state == "open":
            if datetime.now() - self.last_failure_time > timedelta(seconds=self.timeout):
                self.state = "half-open"
            else:
                raise RuntimeError("Circuit breaker is OPEN")

        try:
            result = func()
            # Success - reset if half-open
            if self.state == "half-open":
                self.state = "closed"
                self.failures = 0
            return result

        except Exception as e:
            self.failures += 1
            self.last_failure_time = datetime.now()

            if self.failures >= self.failure_threshold:
                self.state = "open"
                logger.error(f"Circuit breaker opened after {self.failures} failures")

            raise

# Usage
breaker = CircuitBreaker()

def make_api_call():
    return breaker.call(lambda: client.messages.create(...))
```

---

## Input Sanitization

```python
def sanitize_user_input(user_input: str) -> str:
    """Sanitize user input before sending to Claude."""

    # Limit length
    max_length = 50000
    if len(user_input) > max_length:
        user_input = user_input[:max_length] + "\n[truncated]"

    # Remove potentially problematic characters
    # (Be careful - don't break legitimate use cases)
    user_input = user_input.replace('\x00', '')  # Null bytes

    # Escape prompt injection attempts (basic)
    injection_patterns = [
        "Ignore previous instructions",
        "System: ",
        "Assistant: ",
    ]

    for pattern in injection_patterns:
        if pattern.lower() in user_input.lower():
            logger.warning(f"Potential injection attempt: {pattern}")
            # Could reject, sanitize, or just log

    return user_input
```

---

## Response Quality Checks

```python
def validate_response_quality(response: str, expected_length_range: tuple = None) -> bool:
    """Check if response meets quality criteria."""

    # Check for empty response
    if not response or len(response.strip()) < 10:
        logger.warning("Response too short")
        return False

    # Check expected length
    if expected_length_range:
        min_len, max_len = expected_length_range
        if not (min_len <= len(response) <= max_len):
            logger.warning(f"Response length {len(response)} outside expected range")
            return False

    # Check for refusal patterns
    refusal_patterns = [
        "I cannot",
        "I'm not able to",
        "I don't have access to",
    ]

    if any(pattern.lower() in response.lower() for pattern in refusal_patterns):
        logger.info("Claude refused the request")
        return False

    # Check for incomplete responses
    if response.endswith("...") or response.endswith("[truncated]"):
        logger.warning("Response appears incomplete")
        return False

    return True
```

---

## Timeout Handling

```python
import asyncio
from anthropic import AsyncAnthropic

async def call_with_timeout(prompt: str, timeout: int = 30) -> str:
    """Call API with timeout."""

    client = AsyncAnthropic()

    try:
        response = await asyncio.wait_for(
            client.messages.create(
                model="claude-sonnet-4-20250514",
                max_tokens=1024,
                messages=[{"role": "user", "content": prompt}]
            ),
            timeout=timeout
        )
        return response.content[0].text

    except asyncio.TimeoutError:
        logger.error(f"Request timed out after {timeout}s")
        raise
```

---

## Logging Best Practices

```python
import logging

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)

logger = logging.getLogger(__name__)

def log_api_call(model: str, prompt: str, response, error=None):
    """Log API calls for debugging and auditing."""

    log_data = {
        "model": model,
        "prompt_length": len(prompt),
        "timestamp": datetime.now().isoformat(),
    }

    if response:
        log_data.update({
            "response_length": len(response.content[0].text),
            "input_tokens": response.usage.input_tokens,
            "output_tokens": response.usage.output_tokens,
            "stop_reason": response.stop_reason,
        })
        logger.info(f"API call successful: {json.dumps(log_data)}")
    else:
        log_data["error"] = str(error)
        logger.error(f"API call failed: {json.dumps(log_data)}")

    # Don't log full prompts/responses in production (PII, secrets)
    # Only log in debug mode
    if logger.level == logging.DEBUG:
        logger.debug(f"Prompt: {prompt[:200]}...")
        if response:
            logger.debug(f"Response: {response.content[0].text[:200]}...")
```

---

**Last Updated:** 2026-02-16
