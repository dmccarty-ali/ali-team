---
name: ali-clean-code
description: |
  Clean code practices and coding standards for Python and Java. Use when:

  PLANNING: Considering how to structure code for readability, deciding on naming
  conventions, planning error handling strategy, evaluating code organization

  IMPLEMENTATION: Writing any code - variable naming, function structure, error
  handling, logging, comments, formatting, choosing between approaches

  GUIDANCE: Asking about coding best practices, how to write readable code,
  naming conventions, when to add comments, error handling approaches

  REVIEW: Reviewing code quality, checking readability, evaluating naming,
  identifying unclear code, suggesting improvements for maintainability
---

# Clean Code

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Deciding how to name things (variables, functions, classes)
- Planning error handling strategy for a feature
- Considering code organization and file structure
- Evaluating logging needs

**Implementation:**
- Writing any code (this skill applies broadly)
- Choosing between implementation approaches
- Adding error handling or logging
- Deciding whether/how to comment code

**Guidance/Best Practices:**
- Asking about naming conventions
- Asking how to make code more readable
- Asking about error handling best practices
- Asking when to add comments

**Review/Validation:**
- Reviewing code for readability
- Checking if naming is clear
- Evaluating error handling
- Identifying confusing or unclear code

---

## Key Principles

- **Clarity over cleverness**: Write code a tired developer can understand at 2am
- **Names reveal intent**: A name should tell you why it exists and what it does
- **Functions do one thing**: If you need "and" to describe it, split it
- **Fail fast, fail loud**: Detect errors early, make them visible
- **Log for debugging, not for showing off**: Log what helps diagnose problems
- **Comments explain WHY, not WHAT**: Code shows what; comments explain why
- **Consistent style**: Pick conventions and stick to them
- **Delete dead code**: Don't comment it out, delete it (git remembers)
- **No magic values**: Named constants, not mystery numbers
- **Minimize scope**: Declare variables close to where they're used

---

## Naming Conventions

### General Rules

| Guideline | Bad | Good |
|-----------|-----|------|
| Use intention-revealing names | `d` | `elapsed_days` |
| Avoid abbreviations | `calc_ttl_amt` | `calculate_total_amount` |
| Use pronounceable names | `genymdhms` | `generation_timestamp` |
| Use searchable names | `7` | `MAX_RETRY_ATTEMPTS = 7` |
| Avoid mental mapping | `a`, `b`, `c` | `source`, `target`, `result` |
| Don't encode types in names | `strName`, `iCount` | `name`, `count` |
| Be consistent | `fetch`/`get`/`retrieve` mixed | Pick one: `get_` everywhere |

### Python Naming

```python
# Variables and functions: snake_case
user_count = 0
def calculate_total_amount(items: list) -> Decimal:
    pass

# Constants: UPPER_SNAKE_CASE
MAX_RETRY_ATTEMPTS = 3
DEFAULT_TIMEOUT_SECONDS = 30

# Classes: PascalCase
class PaymentProcessor:
    pass

# Private/internal: leading underscore
_internal_cache = {}
def _validate_input(data):
    pass

# "Protected" in classes: single underscore
class BaseParser:
    def _parse_header(self):  # Subclasses may override
        pass

# Name mangling (rare): double underscore
class Widget:
    def __init__(self):
        self.__private_state = {}  # Becomes _Widget__private_state
```

### Java Naming

```java
// Variables and methods: camelCase
int userCount = 0;
public BigDecimal calculateTotalAmount(List<Item> items) { }

// Constants: UPPER_SNAKE_CASE
public static final int MAX_RETRY_ATTEMPTS = 3;
public static final Duration DEFAULT_TIMEOUT = Duration.ofSeconds(30);

// Classes and interfaces: PascalCase
public class PaymentProcessor { }
public interface ClaimRepository { }

// Packages: lowercase, no underscores
package com.aliunde.ingestion.parser;
```

### Boolean Naming

```python
# Use positive names (avoid double negatives)
# Bad
is_not_valid = False
if not is_not_valid:  # Confusing!

# Good
is_valid = True
if is_valid:  # Clear

# Use question-style names
has_permission = True
can_edit = True
should_retry = True
is_active = True

# Avoid meaningless prefixes
# Bad: flag_is_ready, bool_complete
# Good: is_ready, is_complete
```

### Function/Method Naming

```python
# Verb + noun for actions
def send_email(recipient: str, subject: str): ...
def calculate_tax(amount: Decimal): ...
def validate_input(data: dict): ...

# get_ for retrieving (may be expensive)
def get_user_by_id(user_id: str): ...

# find_ when result may not exist (returns Optional)
def find_client_by_email(email: str) -> Optional[Client]: ...

# is_/has_/can_/should_ for boolean returns
def is_valid_email(email: str) -> bool: ...
def has_permission(user: User, action: str) -> bool: ...

# create_/build_ for constructors/factories
def create_payment_intent(amount: Decimal): ...
def build_query(filters: dict) -> str: ...

# Don't use get_ for simple property access
# Bad: def get_name(self): return self._name
# Good: Use @property
@property
def name(self) -> str:
    return self._name
```

---

## Error Handling

### Principles

- **Fail fast**: Validate at entry points, not deep in logic
- **Be specific**: Catch specific exceptions, not bare `except:`
- **Preserve context**: Include useful information in error messages
- **Don't hide errors**: Log and re-raise, or handle completely
- **Use exceptions for exceptional cases**: Not for flow control

### Python Error Handling

```python
# BAD: Bare except hides all errors
try:
    process_data(data)
except:  # Catches SystemExit, KeyboardInterrupt, everything!
    pass

# BAD: Too broad, loses information
try:
    result = external_api_call()
except Exception as e:
    logger.error("API call failed")  # What failed? What was the error?

# GOOD: Specific exceptions, context preserved
try:
    result = external_api_call(client_id=client_id)
except ConnectionError as e:
    logger.error(f"Failed to connect to API for client {client_id}: {e}")
    raise ServiceUnavailableError(f"External API unavailable") from e
except ValidationError as e:
    logger.warning(f"Invalid response from API for client {client_id}: {e}")
    raise

# GOOD: Validate early, fail fast
def process_payment(amount: Decimal, client_id: str) -> PaymentResult:
    if amount <= 0:
        raise ValueError(f"Amount must be positive, got {amount}")
    if not client_id:
        raise ValueError("client_id is required")

    # Now proceed with confidence
    ...
```

### Custom Exceptions

```python
# Define domain-specific exceptions
class PaymentError(Exception):
    """Base for payment-related errors."""
    pass

class InsufficientFundsError(PaymentError):
    """Client doesn't have enough funds."""
    def __init__(self, requested: Decimal, available: Decimal):
        self.requested = requested
        self.available = available
        super().__init__(f"Requested {requested}, but only {available} available")

class PaymentDeclinedError(PaymentError):
    """Payment processor declined the transaction."""
    def __init__(self, reason: str, decline_code: str):
        self.reason = reason
        self.decline_code = decline_code
        super().__init__(f"Payment declined: {reason} (code: {decline_code})")

# Usage
try:
    process_payment(amount, client_id)
except InsufficientFundsError as e:
    notify_client_insufficient_funds(client_id, e.requested, e.available)
except PaymentDeclinedError as e:
    if e.decline_code == "card_expired":
        prompt_update_card(client_id)
    else:
        notify_payment_failed(client_id, e.reason)
```

### Java Error Handling

```java
// BAD: Catching Exception loses specificity
try {
    processData(data);
} catch (Exception e) {
    logger.error("Processing failed");  // Useless
}

// GOOD: Specific exceptions, context preserved
try {
    result = externalApiCall(clientId);
} catch (ConnectException e) {
    logger.error("Failed to connect to API for client {}: {}", clientId, e.getMessage());
    throw new ServiceUnavailableException("External API unavailable", e);
} catch (JsonParseException e) {
    logger.warn("Invalid JSON response for client {}: {}", clientId, e.getMessage());
    throw new InvalidResponseException("API returned invalid JSON", e);
}

// GOOD: Try-with-resources for cleanup
try (var connection = dataSource.getConnection();
     var statement = connection.prepareStatement(sql)) {
    // Use resources
}  // Automatically closed, even on exception
```

---

## Logging

### What to Log

| Log Level | When to Use | Example |
|-----------|-------------|---------|
| ERROR | Something failed that shouldn't | Payment processing failed |
| WARNING | Unexpected but handled | Retry succeeded after failure |
| INFO | Significant business events | Client created, payment processed |
| DEBUG | Diagnostic information | Query parameters, intermediate values |

### Logging Best Practices

```python
import logging
import structlog

logger = structlog.get_logger(__name__)

# BAD: Useless log
logger.info("Processing...")
logger.error("Error occurred")

# GOOD: Context-rich logging
logger.info(
    "payment_processed",
    client_id=client_id,
    amount=str(amount),
    payment_method="card"
)

logger.error(
    "payment_failed",
    client_id=client_id,
    amount=str(amount),
    error_type=type(e).__name__,
    error_message=str(e)
)

# BAD: Logging sensitive data
logger.info(f"Processing payment for SSN {ssn}")  # NEVER!

# GOOD: Log IDs, not values
logger.info(
    "client_lookup",
    client_id=client_id,
    ssn_last_four=ssn[-4:] if ssn else None
)

# BAD: Log and swallow
try:
    risky_operation()
except Exception as e:
    logger.error(f"Failed: {e}")
    # Error is now hidden!

# GOOD: Log and re-raise (or handle completely)
try:
    risky_operation()
except Exception as e:
    logger.error(f"risky_operation failed", error=str(e))
    raise  # Let caller handle it
```

### Structured Logging

```python
# Use structured logging for searchability
# BAD: String interpolation in logs
logger.info(f"User {user_id} created client {client_id} at {timestamp}")

# GOOD: Structured fields (searchable in log aggregators)
logger.info(
    "client_created",
    user_id=user_id,
    client_id=client_id,
    timestamp=timestamp.isoformat()
)
```

---

## Comments

### When to Comment

| Comment Type | Use For | Example |
|--------------|---------|---------|
| WHY comments | Non-obvious business logic | `# IRS requires 7-year retention` |
| WARNING comments | Gotchas, non-obvious behavior | `# NOTE: This API has 5s timeout` |
| TODO comments | Temporary, tracked in issues | `# TODO(#123): Add retry logic` |
| API docs | Public interfaces | Docstrings for public functions |

### When NOT to Comment

```python
# BAD: Comment restates the code
# Increment counter by one
counter += 1

# BAD: Comment explains obvious code
# Check if user is active
if user.is_active:

# BAD: Commented-out code (use git!)
# old_implementation()
# new_implementation()

# BAD: Changelog in comments
# v1.2: Added validation
# v1.1: Fixed bug
# v1.0: Initial version
```

### Good Comments

```python
# GOOD: Explains WHY
# We use a 30-second timeout because the external API
# occasionally takes 20+ seconds under load
EXTERNAL_API_TIMEOUT = 30

# GOOD: Warns about non-obvious behavior
# WARNING: This function modifies the input list in place
def sort_and_dedupe(items: list) -> list:
    ...

# GOOD: Documents public API
def calculate_tax(
    amount: Decimal,
    state: str,
    effective_date: date
) -> Decimal:
    """Calculate sales tax for a given amount.

    Args:
        amount: Pre-tax amount in USD
        state: Two-letter state code (e.g., 'CA')
        effective_date: Date for tax rate lookup

    Returns:
        Tax amount in USD

    Raises:
        ValueError: If state code is not recognized
        TaxRateNotFoundError: If no rate exists for date
    """
    ...

# GOOD: References external documentation
# See IRS Publication 4557 Section 3.2 for retention requirements
RETENTION_YEARS = 7
```

---

## Code Structure

### Function Length

```python
# BAD: Function doing multiple things
def process_order(order):
    # Validate order (10 lines)
    ...
    # Calculate totals (15 lines)
    ...
    # Apply discounts (20 lines)
    ...
    # Process payment (25 lines)
    ...
    # Send confirmation (10 lines)
    ...

# GOOD: Each function does one thing
def process_order(order: Order) -> OrderResult:
    validated = validate_order(order)
    totals = calculate_totals(validated)
    discounted = apply_discounts(totals)
    payment = process_payment(discounted)
    send_confirmation(payment)
    return OrderResult(payment)

def validate_order(order: Order) -> ValidatedOrder:
    """Validates order structure and items."""
    ...

def calculate_totals(order: ValidatedOrder) -> OrderWithTotals:
    """Calculates subtotal, tax, and total."""
    ...
```

### Early Returns

```python
# BAD: Deep nesting
def process_request(request):
    if request:
        if request.user:
            if request.user.is_active:
                if request.has_permission:
                    # Finally, the actual logic (deeply nested)
                    return do_the_thing(request)
                else:
                    return error("No permission")
            else:
                return error("User inactive")
        else:
            return error("No user")
    else:
        return error("No request")

# GOOD: Guard clauses with early returns
def process_request(request):
    if not request:
        return error("No request")
    if not request.user:
        return error("No user")
    if not request.user.is_active:
        return error("User inactive")
    if not request.has_permission:
        return error("No permission")

    # Main logic at base indentation
    return do_the_thing(request)
```

### Avoid Boolean Parameters

```python
# BAD: What does True mean here?
send_email(user, "Welcome", True)

# GOOD: Named parameter or separate functions
send_email(user, "Welcome", include_attachment=True)

# OR: Separate functions for clarity
send_welcome_email(user)
send_welcome_email_with_attachment(user)
```

---

## Performance Awareness

### Common Mistakes

| Mistake | Problem | Solution |
|---------|---------|----------|
| N+1 queries | 100 items = 101 queries | Use JOINs or eager loading |
| String concatenation in loop | O(n²) memory allocations | Use list + join |
| Re-computing in loops | Wasted CPU | Compute once before loop |
| Loading all into memory | OOM on large data | Use generators, streaming |
| Regex compilation in loop | Compiled every iteration | Compile once, reuse |

### Python Performance

```python
# BAD: String concatenation in loop (O(n²))
result = ""
for item in items:
    result += str(item) + ", "

# GOOD: List + join (O(n))
result = ", ".join(str(item) for item in items)

# BAD: Re-computing in loop
for item in items:
    config = load_config()  # Loaded every iteration!
    process(item, config)

# GOOD: Compute once
config = load_config()
for item in items:
    process(item, config)

# BAD: Loading all into memory
all_records = list(huge_query())  # OOM!
for record in all_records:
    process(record)

# GOOD: Stream/iterate
for record in huge_query():  # Processes one at a time
    process(record)

# BAD: Regex compiled every call
def is_valid_email(email):
    return re.match(r'^[^@]+@[^@]+\.[^@]+$', email)  # Compiled each time

# GOOD: Compile once
EMAIL_PATTERN = re.compile(r'^[^@]+@[^@]+\.[^@]+$')
def is_valid_email(email):
    return EMAIL_PATTERN.match(email)
```

### Java Performance

```java
// BAD: String concatenation in loop
String result = "";
for (String item : items) {
    result += item + ", ";  // Creates new String each time
}

// GOOD: StringBuilder
StringBuilder sb = new StringBuilder();
for (String item : items) {
    sb.append(item).append(", ");
}
String result = sb.toString();

// GOOD: String.join (Java 8+)
String result = String.join(", ", items);

// BAD: Boxing in hot path
for (int i = 0; i < 1000000; i++) {
    Integer boxed = i;  // Boxing overhead
}

// GOOD: Use primitives
for (int i = 0; i < 1000000; i++) {
    // Use i directly
}
```

---

## Testing Practices

### Test Naming

```python
# BAD: Unclear test names
def test_1():
def test_validation():
def test_it_works():

# GOOD: Describes scenario and expected outcome
def test_create_client_with_valid_email_succeeds():
def test_create_client_with_duplicate_email_raises_error():
def test_payment_with_insufficient_funds_returns_declined():
```

### Test Structure

```python
def test_client_creation_sends_welcome_email():
    # Arrange (Given)
    email_service = Mock()
    service = ClientService(email_service=email_service)
    request = CreateClientRequest(name="Test", email="test@test.com")

    # Act (When)
    client = service.create_client(request)

    # Assert (Then)
    assert client.email == "test@test.com"
    email_service.send_welcome.assert_called_once_with("test@test.com")
```

### What to Test

| Test Type | Focus | Coverage |
|-----------|-------|----------|
| Unit tests | Single function/class | Business logic, edge cases |
| Integration tests | Component interaction | Database, external services |
| End-to-end tests | Full workflows | Critical user journeys |

---

---

## Type Hints (Python)

### Comprehensive Type Annotations

```python
from typing import Optional, Union, TypeVar, Generic, Callable
from collections.abc import Sequence, Mapping
from dataclasses import dataclass
from datetime import datetime

# Basic types
def process_claim(claim_id: str, amount: float, urgent: bool = False) -> dict:
    ...

# Optional (can be None)
def find_client(email: str) -> Optional[Client]:
    ...

# Union types (multiple possible types)
def parse_id(value: str | int) -> str:  # Python 3.10+
    return str(value)

# Collections with element types
def process_items(items: list[str]) -> dict[str, int]:
    ...

# Callable types
def retry(func: Callable[[str], dict], attempts: int = 3) -> dict:
    ...

# TypeVar for generics
T = TypeVar('T')
def first_or_none(items: Sequence[T]) -> Optional[T]:
    return items[0] if items else None

# Generic classes
class Repository(Generic[T]):
    def get(self, id: str) -> Optional[T]: ...
    def save(self, entity: T) -> T: ...

class ClientRepository(Repository[Client]):
    ...
```

### Dataclass with Types

```python
from dataclasses import dataclass, field
from typing import Optional
from datetime import datetime

@dataclass
class CreateClientRequest:
    name: str
    email: str
    phone: Optional[str] = None
    tags: list[str] = field(default_factory=list)
    created_at: datetime = field(default_factory=datetime.utcnow)

    def __post_init__(self):
        self.email = self.email.lower()
```

---

## Enum Patterns

```python
from enum import Enum, auto, IntEnum
from typing import ClassVar

# String enum (serializes nicely to JSON)
class DocumentStatus(str, Enum):
    PENDING = "pending"
    PROCESSING = "processing"
    COMPLETED = "completed"
    FAILED = "failed"

    @classmethod
    def active_statuses(cls) -> list['DocumentStatus']:
        return [cls.PENDING, cls.PROCESSING]

# Integer enum (for database storage)
class Priority(IntEnum):
    LOW = 1
    MEDIUM = 2
    HIGH = 3
    URGENT = 4

    def is_escalated(self) -> bool:
        return self >= Priority.HIGH

# Auto-numbered enum
class TaskState(Enum):
    QUEUED = auto()
    RUNNING = auto()
    COMPLETED = auto()
    FAILED = auto()

# Usage
status = DocumentStatus.PENDING
print(status.value)  # "pending"
print(status.name)   # "PENDING"
print(status in DocumentStatus.active_statuses())  # True

# In Pydantic models
from pydantic import BaseModel

class Document(BaseModel):
    id: str
    status: DocumentStatus  # Automatically validates
```

---

## Async Code Organization

```python
import asyncio
from contextlib import asynccontextmanager
from typing import AsyncGenerator

# Async context manager
@asynccontextmanager
async def get_connection() -> AsyncGenerator[Connection, None]:
    conn = await pool.acquire()
    try:
        yield conn
    finally:
        await pool.release(conn)

# Usage
async def fetch_data():
    async with get_connection() as conn:
        return await conn.fetch("SELECT ...")

# Gathering concurrent tasks
async def fetch_all_data(client_ids: list[str]) -> list[dict]:
    tasks = [fetch_client_data(cid) for cid in client_ids]
    return await asyncio.gather(*tasks)

# Async comprehension
async def process_items(items: list[Item]) -> list[Result]:
    return [await process(item) async for item in async_generator(items)]

# Timeout handling
async def fetch_with_timeout(url: str, timeout: float = 5.0) -> dict:
    try:
        return await asyncio.wait_for(fetch(url), timeout=timeout)
    except asyncio.TimeoutError:
        raise ServiceTimeoutError(f"Request to {url} timed out")
```

---

## Linting Configuration

### pyproject.toml (Ruff + Black)

```toml
[tool.ruff]
line-length = 100
target-version = "py311"
select = [
    "E",    # pycodestyle errors
    "F",    # pyflakes
    "I",    # isort
    "N",    # pep8-naming
    "UP",   # pyupgrade
    "B",    # bugbear
    "C4",   # comprehensions
    "SIM",  # simplify
]
ignore = [
    "E501",  # line length (black handles this)
]

[tool.ruff.isort]
known-first-party = ["app", "tests"]

[tool.black]
line-length = 100
target-version = ["py311"]

[tool.mypy]
python_version = "3.11"
strict = true
warn_return_any = true
disallow_untyped_defs = true
ignore_missing_imports = true
```

### Pre-commit Configuration

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.1.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.5.0
    hooks:
      - id: mypy
        additional_dependencies: [types-all]
```

---

## Immutability Patterns

```python
from dataclasses import dataclass
from typing import NamedTuple

# Frozen dataclass (immutable)
@dataclass(frozen=True)
class Money:
    amount: int
    currency: str = "USD"

    def add(self, other: 'Money') -> 'Money':
        # Returns new instance, doesn't modify
        return Money(self.amount + other.amount, self.currency)

# NamedTuple (always immutable)
class Point(NamedTuple):
    x: float
    y: float

    def distance_to(self, other: 'Point') -> float:
        return ((self.x - other.x) ** 2 + (self.y - other.y) ** 2) ** 0.5

# Tuple unpacking
p = Point(3, 4)
x, y = p  # Works because it's a tuple

# frozenset for immutable sets
ALLOWED_STATUSES = frozenset({"pending", "approved", "rejected"})
```

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| Python style config | `pyproject.toml` (ruff/black config) |
| Java style config | `.editorconfig`, `pom.xml` (checkstyle) |
| Logging setup | `ali-ai-acctg/app/logging.py` |
| Custom exceptions | `ali-ai-acctg/app/exceptions.py` |
| Test examples | `ali-ai-ingestion-engine/tests/` |
| Naming conventions | Follow existing code patterns |

---

## References

- [Clean Code (Robert Martin)](https://www.oreilly.com/library/view/clean-code-a/9780136083238/)
- [PEP 8 - Python Style Guide](https://peps.python.org/pep-0008/)
- [Google Python Style Guide](https://google.github.io/styleguide/pyguide.html)
- [Google Java Style Guide](https://google.github.io/styleguide/javaguide.html)
- [Effective Python (Slatkin)](https://effectivepython.com/)
