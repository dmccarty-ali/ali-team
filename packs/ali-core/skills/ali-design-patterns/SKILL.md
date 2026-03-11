---
name: ali-design-patterns
description: |
  Software design patterns and anti-patterns for Python and Java. Use when:

  PLANNING: Designing classes, modules, services, or system architecture;
  evaluating how to structure code; considering trade-offs between patterns;
  architecting new features or refactoring existing ones

  IMPLEMENTATION: Writing new classes or modules, structuring services,
  implementing dependency injection, building factories or repositories,
  adding resilience patterns (retry, circuit breaker)

  GUIDANCE: Asking about best practices for code structure, how to apply SOLID,
  what pattern fits a situation, how others organize code, Clean Architecture

  REVIEW: Reviewing code structure for design issues, identifying anti-patterns,
  checking if SOLID principles are followed, evaluating code maintainability
---

# Design Patterns

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing how to structure a new feature or service
- Evaluating which design pattern fits a problem
- Planning refactoring of existing code
- Considering dependency injection approaches
- Architecting service layers or module boundaries

**Implementation:**
- Writing factory or builder patterns
- Implementing repository patterns
- Building resilience patterns (retry, circuit breaker)
- Structuring Python classes with dataclasses
- Using Java records and streams effectively

**Guidance/Best Practices:**
- Asking about SOLID principles and their application
- Asking which pattern to use for a specific situation
- Asking how to organize code for maintainability
- Asking about Clean Architecture or DDD concepts

**Review/Validation:**
- Reviewing code for design anti-patterns
- Checking if code violates SOLID principles
- Identifying code smells and suggesting improvements
- Evaluating whether abstractions are appropriate

---

## Key Principles

### SOLID

- **S - Single Responsibility**: A class should have only one reason to change
- **O - Open/Closed**: Open for extension, closed for modification
- **L - Liskov Substitution**: Subtypes must be substitutable for base types
- **I - Interface Segregation**: Many specific interfaces > one general interface
- **D - Dependency Inversion**: Depend on abstractions, not concretions

### Additional Principles

- **YAGNI**: You Aren't Gonna Need It - don't build for hypothetical futures
- **DRY**: Don't Repeat Yourself - but 3 similar lines beats premature abstraction
- **Composition over inheritance**: Prefer has-a relationships over is-a
- **Fail fast**: Detect errors early, at system boundaries
- **Explicit over implicit**: Make dependencies and behavior visible

---

## Patterns We Use

### Repository Pattern (Python)

Abstracts data access, enables testing without database:

```python
from abc import ABC, abstractmethod
from typing import Optional
from dataclasses import dataclass

@dataclass
class Client:
    id: str
    name: str
    email: str

class ClientRepository(ABC):
    """Abstract repository - depend on this interface."""

    @abstractmethod
    def get_by_id(self, client_id: str) -> Optional[Client]:
        pass

    @abstractmethod
    def save(self, client: Client) -> None:
        pass

class PostgresClientRepository(ClientRepository):
    """Concrete implementation for production."""

    def __init__(self, session):
        self._session = session

    def get_by_id(self, client_id: str) -> Optional[Client]:
        return self._session.query(ClientModel).filter_by(id=client_id).first()

    def save(self, client: Client) -> None:
        self._session.merge(ClientModel(**vars(client)))
        self._session.commit()

class InMemoryClientRepository(ClientRepository):
    """Concrete implementation for testing."""

    def __init__(self):
        self._data: dict[str, Client] = {}

    def get_by_id(self, client_id: str) -> Optional[Client]:
        return self._data.get(client_id)

    def save(self, client: Client) -> None:
        self._data[client.id] = client
```

### Factory Pattern (Python)

Creates objects without exposing instantiation logic:

```python
from enum import Enum
from typing import Protocol

class Parser(Protocol):
    def parse(self, content: bytes) -> dict: ...

class EDI835Parser:
    def parse(self, content: bytes) -> dict:
        # 835 payment parsing logic
        ...

class EDI837IParser:
    def parse(self, content: bytes) -> dict:
        # 837I institutional claim parsing logic
        ...

class ParserType(Enum):
    EDI_835 = "835"
    EDI_837I = "837I"

def create_parser(parser_type: ParserType) -> Parser:
    """Factory function - creates appropriate parser."""
    parsers = {
        ParserType.EDI_835: EDI835Parser,
        ParserType.EDI_837I: EDI837IParser,
    }
    parser_class = parsers.get(parser_type)
    if not parser_class:
        raise ValueError(f"Unknown parser type: {parser_type}")
    return parser_class()
```

### Builder Pattern (Java)

Constructs complex objects step by step:

```java
public record ClaimRecord(
    String claimId,
    String patientId,
    BigDecimal amount,
    LocalDate serviceDate,
    List<String> diagnosisCodes
) {
    public static Builder builder() {
        return new Builder();
    }

    public static class Builder {
        private String claimId;
        private String patientId;
        private BigDecimal amount = BigDecimal.ZERO;
        private LocalDate serviceDate;
        private List<String> diagnosisCodes = new ArrayList<>();

        public Builder claimId(String claimId) {
            this.claimId = claimId;
            return this;
        }

        public Builder patientId(String patientId) {
            this.patientId = patientId;
            return this;
        }

        public Builder amount(BigDecimal amount) {
            this.amount = amount;
            return this;
        }

        public Builder serviceDate(LocalDate date) {
            this.serviceDate = date;
            return this;
        }

        public Builder addDiagnosis(String code) {
            this.diagnosisCodes.add(code);
            return this;
        }

        public ClaimRecord build() {
            Objects.requireNonNull(claimId, "claimId required");
            Objects.requireNonNull(patientId, "patientId required");
            return new ClaimRecord(claimId, patientId, amount, serviceDate,
                                   List.copyOf(diagnosisCodes));
        }
    }
}

// Usage
var claim = ClaimRecord.builder()
    .claimId("CLM001")
    .patientId("PAT123")
    .amount(new BigDecimal("1500.00"))
    .serviceDate(LocalDate.now())
    .addDiagnosis("J06.9")
    .build();
```

### Strategy Pattern (Python)

Encapsulates algorithms, makes them interchangeable:

```python
from typing import Protocol
from decimal import Decimal

class PricingStrategy(Protocol):
    def calculate_fee(self, return_complexity: int) -> Decimal: ...

class StandardPricing:
    def calculate_fee(self, return_complexity: int) -> Decimal:
        return Decimal("200") + Decimal(return_complexity * 25)

class VolumePricing:
    def __init__(self, discount_pct: Decimal):
        self._discount = discount_pct

    def calculate_fee(self, return_complexity: int) -> Decimal:
        base = Decimal("200") + Decimal(return_complexity * 25)
        return base * (1 - self._discount)

class FeeCalculator:
    def __init__(self, strategy: PricingStrategy):
        self._strategy = strategy

    def calculate(self, return_complexity: int) -> Decimal:
        return self._strategy.calculate_fee(return_complexity)

# Usage - strategy injected at runtime
calculator = FeeCalculator(VolumePricing(Decimal("0.10")))
fee = calculator.calculate(return_complexity=3)
```

### Service Layer Pattern (Python)

Thin orchestration layer between API and domain:

```python
from dataclasses import dataclass

@dataclass
class CreateClientRequest:
    name: str
    email: str
    phone: Optional[str] = None

class ClientService:
    """Service layer - orchestrates domain operations."""

    def __init__(
        self,
        client_repo: ClientRepository,
        email_service: EmailService,
        audit_log: AuditLog
    ):
        self._clients = client_repo
        self._email = email_service
        self._audit = audit_log

    def create_client(self, request: CreateClientRequest, user_id: str) -> Client:
        # Validate
        if self._clients.get_by_email(request.email):
            raise DuplicateClientError(f"Email already registered: {request.email}")

        # Create domain object
        client = Client(
            id=str(uuid.uuid4()),
            name=request.name,
            email=request.email,
            phone=request.phone
        )

        # Persist
        self._clients.save(client)

        # Side effects
        self._email.send_welcome(client.email)
        self._audit.log("client.created", user_id, client.id)

        return client
```

### Circuit Breaker Pattern (Python)

Prevents cascading failures from external service outages:

```python
import time
from enum import Enum
from dataclasses import dataclass, field

class CircuitState(Enum):
    CLOSED = "closed"      # Normal operation
    OPEN = "open"          # Failing, reject requests
    HALF_OPEN = "half_open"  # Testing if recovered

@dataclass
class CircuitBreaker:
    """Wraps external calls with failure protection."""
    failure_threshold: int = 5
    recovery_timeout: float = 30.0
    _state: CircuitState = field(default=CircuitState.CLOSED, init=False)
    _failures: int = field(default=0, init=False)
    _last_failure: float = field(default=0.0, init=False)

    def call(self, func, *args, **kwargs):
        if self._state == CircuitState.OPEN:
            if time.time() - self._last_failure >= self.recovery_timeout:
                self._state = CircuitState.HALF_OPEN
            else:
                raise CircuitOpenError("Circuit is open")

        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise

    def _on_success(self):
        self._failures = 0
        self._state = CircuitState.CLOSED

    def _on_failure(self):
        self._failures += 1
        self._last_failure = time.time()
        if self._failures >= self.failure_threshold:
            self._state = CircuitState.OPEN

# Usage
stripe_circuit = CircuitBreaker(failure_threshold=3, recovery_timeout=60)

def charge_card(amount: Decimal) -> str:
    return stripe_circuit.call(stripe.PaymentIntent.create, amount=amount)
```

### Retry with Exponential Backoff (Python)

```python
import time
import random
from functools import wraps

def retry(max_attempts: int = 3, base_delay: float = 1.0, max_delay: float = 60.0):
    """Decorator for retry with exponential backoff and jitter."""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts - 1:
                        raise
                    delay = min(base_delay * (2 ** attempt), max_delay)
                    delay *= (0.5 + random.random())  # Add jitter
                    time.sleep(delay)
        return wrapper
    return decorator

@retry(max_attempts=3, base_delay=1.0)
def call_external_api(payload: dict) -> dict:
    return requests.post(api_url, json=payload).json()
```

---

## Python-Specific Patterns

### Dataclasses for Value Objects

```python
from dataclasses import dataclass, field
from decimal import Decimal

@dataclass(frozen=True)  # Immutable
class Money:
    amount: Decimal
    currency: str = "USD"

    def __add__(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise ValueError("Currency mismatch")
        return Money(self.amount + other.amount, self.currency)
```

### Context Managers for Resource Safety

```python
from contextlib import contextmanager

@contextmanager
def database_transaction(session):
    """Ensures commit or rollback on any exit."""
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()

# Usage
with database_transaction(get_session()) as session:
    session.add(record)
    # Auto-commits on success, rolls back on exception
```

### Protocol for Structural Typing

```python
from typing import Protocol

class Notifier(Protocol):
    """Duck typing with type hints."""
    def notify(self, message: str) -> None: ...

class EmailNotifier:
    def notify(self, message: str) -> None:
        send_email(message)

class SlackNotifier:
    def notify(self, message: str) -> None:
        post_to_slack(message)

def send_notification(notifier: Notifier, message: str):
    # Works with any object that has notify() method
    notifier.notify(message)
```

---

## Java-Specific Patterns

### Records for Immutable Data

```java
// Concise immutable data carrier (Java 16+)
public record PaymentResult(
    String transactionId,
    BigDecimal amount,
    PaymentStatus status,
    Instant timestamp
) {
    // Compact constructor for validation
    public PaymentResult {
        Objects.requireNonNull(transactionId);
        Objects.requireNonNull(status);
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Amount cannot be negative");
        }
    }
}
```

### Try-with-Resources

```java
// Ensures resources are closed even on exception
try (var connection = dataSource.getConnection();
     var statement = connection.prepareStatement(sql);
     var results = statement.executeQuery()) {

    while (results.next()) {
        process(results);
    }
}  // All resources auto-closed
```

### Stream Processing

```java
// Functional processing pipeline
List<ClaimSummary> summaries = claims.stream()
    .filter(c -> c.status() == ClaimStatus.APPROVED)
    .map(c -> new ClaimSummary(c.id(), c.amount()))
    .sorted(Comparator.comparing(ClaimSummary::amount).reversed())
    .limit(100)
    .toList();
```

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| God Class | Does too much, hard to test/maintain | Split by responsibility |
| Primitive Obsession | Uses strings/ints for domain concepts | Create value objects |
| Feature Envy | Method uses another class's data heavily | Move method to that class |
| Shotgun Surgery | One change requires many file edits | Consolidate related code |
| Inheritance for code reuse | Creates tight coupling | Use composition |
| Singleton everywhere | Hidden dependencies, hard to test | Use dependency injection |
| Null returns | Forces null checks everywhere | Return Optional or empty collection |
| Magic numbers/strings | Unclear meaning | Use named constants or enums |
| Deep nesting | Hard to read and test | Extract methods, early returns |
| Boolean parameters | Unclear at call site | Use enums or separate methods |
| Catch-all exception | Hides specific errors | Catch specific exceptions |
| Service locator | Hides dependencies | Use constructor injection |

---

## Code Smells Quick Reference

| Smell | Symptom | Remedy |
|-------|---------|--------|
| Long Method | > 20 lines, does multiple things | Extract methods |
| Long Parameter List | > 3-4 parameters | Introduce parameter object |
| Duplicate Code | Same logic in multiple places | Extract to shared function |
| Data Clumps | Same group of fields appear together | Create class for the group |
| Switch on Type | Switch statement selecting behavior | Use polymorphism/strategy |
| Temporary Field | Field only used in some scenarios | Extract to separate class |
| Message Chains | a.b().c().d() | Introduce intermediary method |
| Middle Man | Class that only delegates | Remove or inline |
| Inappropriate Intimacy | Class knows too much about another | Move methods, extract interface |
| Comments explaining "what" | Code needs comments to understand | Refactor to self-documenting |

---

## Testing Patterns

### Testing Frameworks

| Framework | Purpose | Language |
|-----------|---------|----------|
| pytest | Unit/integration testing | Python |
| pytest-cov | Code coverage | Python |
| hypothesis | Property-based testing | Python |
| pytest-asyncio | Async code testing | Python |
| JUnit 5 | Unit testing | Java |
| Mockito | Mocking | Java |
| AssertJ | Fluent assertions | Java |
| ArchUnit | Architecture testing | Java |

### Test Structure (Arrange-Act-Assert)

```python
def test_client_creation_sends_welcome_email():
    # Arrange
    mock_email = Mock(spec=EmailService)
    service = ClientService(
        client_repo=InMemoryClientRepository(),
        email_service=mock_email,
        audit_log=Mock()
    )
    request = CreateClientRequest(name="Test", email="test@example.com")

    # Act
    client = service.create_client(request, user_id="user123")

    # Assert
    assert client.email == "test@example.com"
    mock_email.send_welcome.assert_called_once_with("test@example.com")
```

### Testing with Dependency Injection

```python
# Production wiring
def create_app():
    session = create_database_session()
    return ClientService(
        client_repo=PostgresClientRepository(session),
        email_service=SESEmailService(),
        audit_log=CloudWatchAuditLog()
    )

# Test wiring - swap implementations
def test_client_service():
    service = ClientService(
        client_repo=InMemoryClientRepository(),  # No database
        email_service=Mock(),                     # No emails sent
        audit_log=Mock()                          # No cloud logging
    )
    # Test business logic in isolation
```

### Property-Based Testing (Python)

```python
from hypothesis import given, strategies as st

@given(st.lists(st.integers()))
def test_sort_preserves_elements(items):
    """Property: sorting should not add or remove elements."""
    sorted_items = sort_function(items)
    assert sorted(items) == sorted(sorted_items)

@given(st.text(min_size=1))
def test_email_validation_never_crashes(text):
    """Property: validation handles any input without crashing."""
    try:
        validate_email(text)
    except ValidationError:
        pass  # Expected for invalid input
    # No unhandled exceptions
```

### Architecture Tests (Java)

```java
@AnalyzeClasses(packages = "com.aliunde")
public class ArchitectureTest {

    @ArchTest
    static final ArchRule services_should_not_access_controllers =
        noClasses().that().resideInAPackage("..service..")
            .should().accessClassesThat().resideInAPackage("..controller..");

    @ArchTest
    static final ArchRule repositories_should_only_be_accessed_by_services =
        classes().that().resideInAPackage("..repository..")
            .should().onlyBeAccessed().byAnyPackage("..service..", "..repository..");
}
```

### Test Coverage Guidelines

| Coverage Type | Target | Notes |
|---------------|--------|-------|
| Line coverage | 80%+ | Focus on business logic, not boilerplate |
| Branch coverage | 70%+ | Cover both sides of conditionals |
| Critical paths | 100% | Payment, auth, data processing |
| Happy path | Required | Basic success scenarios |
| Error paths | Required | Exception handling, edge cases |
| Integration | Key flows | End-to-end critical paths |

---

## FastAPI Dependency Injection

FastAPI's Depends() provides powerful DI without a framework:

```python
from fastapi import Depends, FastAPI
from functools import lru_cache

# Configuration dependency
class Settings:
    db_url: str
    api_key: str

@lru_cache
def get_settings() -> Settings:
    return Settings()

# Database session dependency
async def get_db_session(settings: Settings = Depends(get_settings)):
    engine = create_async_engine(settings.db_url)
    async with AsyncSession(engine) as session:
        yield session

# Repository dependency (depends on session)
async def get_client_repo(
    session: AsyncSession = Depends(get_db_session)
) -> ClientRepository:
    return PostgresClientRepository(session)

# Service dependency (depends on repository)
async def get_client_service(
    repo: ClientRepository = Depends(get_client_repo),
    settings: Settings = Depends(get_settings)
) -> ClientService:
    return ClientService(repo=repo, api_key=settings.api_key)

# Use in endpoint
@app.get("/clients/{client_id}")
async def get_client(
    client_id: str,
    service: ClientService = Depends(get_client_service)
):
    return await service.get_client(client_id)
```

### Testing with Dependency Overrides

```python
from fastapi.testclient import TestClient

def test_get_client():
    # Mock the repository
    mock_repo = Mock(spec=ClientRepository)
    mock_repo.get.return_value = Client(id="123", name="Test")

    # Override dependency
    app.dependency_overrides[get_client_repo] = lambda: mock_repo

    client = TestClient(app)
    response = client.get("/clients/123")

    assert response.status_code == 200
    assert response.json()["name"] == "Test"

    # Clean up
    app.dependency_overrides.clear()
```

---

## Decorator Patterns

Python decorators for cross-cutting concerns:

### Function Decorators

```python
import functools
import time
import logging
from typing import TypeVar, Callable

T = TypeVar('T')

def retry(max_attempts: int = 3, delay: float = 1.0):
    """Retry decorator with exponential backoff."""
    def decorator(func: Callable[..., T]) -> Callable[..., T]:
        @functools.wraps(func)
        def wrapper(*args, **kwargs) -> T:
            last_exception = None
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    last_exception = e
                    if attempt < max_attempts - 1:
                        time.sleep(delay * (2 ** attempt))
            raise last_exception
        return wrapper
    return decorator

def log_execution(logger: logging.Logger = None):
    """Log function entry, exit, and duration."""
    def decorator(func: Callable[..., T]) -> Callable[..., T]:
        nonlocal logger
        logger = logger or logging.getLogger(func.__module__)

        @functools.wraps(func)
        def wrapper(*args, **kwargs) -> T:
            logger.info(f"Entering {func.__name__}")
            start = time.perf_counter()
            try:
                result = func(*args, **kwargs)
                duration = time.perf_counter() - start
                logger.info(f"Exiting {func.__name__} ({duration:.3f}s)")
                return result
            except Exception as e:
                logger.error(f"Error in {func.__name__}: {e}")
                raise
        return wrapper
    return decorator

# Usage
@retry(max_attempts=3)
@log_execution()
def call_external_api(url: str) -> dict:
    return requests.get(url).json()
```

### Async Decorators

```python
import asyncio
import functools

def async_retry(max_attempts: int = 3):
    """Async-compatible retry decorator."""
    def decorator(func):
        @functools.wraps(func)
        async def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return await func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts - 1:
                        raise
                    await asyncio.sleep(2 ** attempt)
        return wrapper
    return decorator

def cache_result(ttl_seconds: int = 300):
    """Simple cache decorator for async functions."""
    cache = {}

    def decorator(func):
        @functools.wraps(func)
        async def wrapper(*args, **kwargs):
            key = (args, tuple(sorted(kwargs.items())))
            now = time.time()

            if key in cache:
                result, timestamp = cache[key]
                if now - timestamp < ttl_seconds:
                    return result

            result = await func(*args, **kwargs)
            cache[key] = (result, now)
            return result
        return wrapper
    return decorator
```

### Class Decorators

```python
from dataclasses import dataclass

def singleton(cls):
    """Make a class a singleton."""
    instances = {}

    @functools.wraps(cls)
    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]

    return get_instance

def register_handler(event_type: str):
    """Register class as handler for event type."""
    def decorator(cls):
        EVENT_HANDLERS[event_type] = cls
        return cls
    return decorator

# Usage
@register_handler('payment.completed')
class PaymentCompletedHandler:
    def handle(self, event: dict):
        ...
```

---

## State Machine Pattern

Model workflows with explicit states and transitions:

```python
from enum import Enum, auto
from dataclasses import dataclass, field
from typing import Callable, Optional
from datetime import datetime

class DocumentState(Enum):
    PENDING_UPLOAD = auto()
    UPLOADED = auto()
    PROCESSING = auto()
    PROCESSED = auto()
    APPROVED = auto()
    REJECTED = auto()
    ARCHIVED = auto()

@dataclass
class Transition:
    from_state: DocumentState
    to_state: DocumentState
    guard: Optional[Callable] = None  # Condition for transition
    action: Optional[Callable] = None  # Side effect on transition

class DocumentStateMachine:
    """State machine for document processing workflow."""

    TRANSITIONS = [
        Transition(DocumentState.PENDING_UPLOAD, DocumentState.UPLOADED),
        Transition(DocumentState.UPLOADED, DocumentState.PROCESSING),
        Transition(DocumentState.PROCESSING, DocumentState.PROCESSED),
        Transition(DocumentState.PROCESSED, DocumentState.APPROVED,
                   guard=lambda doc: doc.validation_passed),
        Transition(DocumentState.PROCESSED, DocumentState.REJECTED,
                   guard=lambda doc: not doc.validation_passed),
        Transition(DocumentState.APPROVED, DocumentState.ARCHIVED),
        Transition(DocumentState.REJECTED, DocumentState.ARCHIVED),
    ]

    def __init__(self, document):
        self.document = document

    def can_transition(self, to_state: DocumentState) -> bool:
        """Check if transition is allowed."""
        for t in self.TRANSITIONS:
            if t.from_state == self.document.state and t.to_state == to_state:
                if t.guard is None or t.guard(self.document):
                    return True
        return False

    def transition(self, to_state: DocumentState) -> bool:
        """Execute state transition."""
        for t in self.TRANSITIONS:
            if t.from_state == self.document.state and t.to_state == to_state:
                if t.guard and not t.guard(self.document):
                    raise ValueError(f"Guard failed for {t.from_state} -> {t.to_state}")

                # Execute action if defined
                if t.action:
                    t.action(self.document)

                self.document.state = to_state
                self.document.state_changed_at = datetime.utcnow()
                return True

        raise ValueError(f"Invalid transition: {self.document.state} -> {to_state}")

    def available_transitions(self) -> list[DocumentState]:
        """Get all valid next states."""
        return [
            t.to_state for t in self.TRANSITIONS
            if t.from_state == self.document.state
            and (t.guard is None or t.guard(self.document))
        ]

# Usage
doc = Document(state=DocumentState.PROCESSED, validation_passed=True)
sm = DocumentStateMachine(doc)

if sm.can_transition(DocumentState.APPROVED):
    sm.transition(DocumentState.APPROVED)
```

---

## Domain-Driven Design Patterns

### Value Objects

Immutable objects defined by their attributes, not identity:

```python
from dataclasses import dataclass
from typing import Optional

@dataclass(frozen=True)  # Immutable
class Money:
    """Value object for monetary amounts."""
    amount: int  # Store as cents to avoid float issues
    currency: str = "USD"

    def __post_init__(self):
        if self.amount < 0:
            raise ValueError("Amount cannot be negative")

    def add(self, other: 'Money') -> 'Money':
        if self.currency != other.currency:
            raise ValueError("Cannot add different currencies")
        return Money(self.amount + other.amount, self.currency)

    def to_decimal(self) -> float:
        return self.amount / 100

@dataclass(frozen=True)
class EmailAddress:
    """Value object for email addresses."""
    value: str

    def __post_init__(self):
        if '@' not in self.value:
            raise ValueError(f"Invalid email: {self.value}")
        # Normalize to lowercase
        object.__setattr__(self, 'value', self.value.lower())

    @property
    def domain(self) -> str:
        return self.value.split('@')[1]

@dataclass(frozen=True)
class SSN:
    """Value object for Social Security Numbers."""
    _digits: str  # Store only digits

    def __init__(self, value: str):
        import re
        digits = re.sub(r'\D', '', value)
        if len(digits) != 9:
            raise ValueError("SSN must be 9 digits")
        object.__setattr__(self, '_digits', digits)

    @property
    def masked(self) -> str:
        return f"XXX-XX-{self._digits[-4:]}"

    @property
    def last_four(self) -> str:
        return self._digits[-4:]
```

### Aggregates

Cluster of domain objects treated as a unit:

```python
from dataclasses import dataclass, field
from typing import List
from datetime import datetime

@dataclass
class TaxReturn:
    """Aggregate root for tax return processing."""
    id: str
    client_id: str
    tax_year: int
    status: str = "draft"
    documents: List['Document'] = field(default_factory=list)
    line_items: List['LineItem'] = field(default_factory=list)
    created_at: datetime = field(default_factory=datetime.utcnow)

    # Invariants enforced by aggregate root
    def add_document(self, doc: 'Document'):
        """Add document with validation."""
        if self.status == "submitted":
            raise ValueError("Cannot add documents to submitted return")
        if doc.document_type not in ALLOWED_DOCUMENT_TYPES:
            raise ValueError(f"Invalid document type: {doc.document_type}")
        self.documents.append(doc)

    def submit(self):
        """Submit return with all validations."""
        if not self.documents:
            raise ValueError("Cannot submit without documents")
        if not self._has_required_documents():
            raise ValueError("Missing required documents")
        if self._calculate_balance_due() < Money(0):
            raise ValueError("Balance due calculation error")

        self.status = "submitted"
        self.submitted_at = datetime.utcnow()

    def _has_required_documents(self) -> bool:
        required = {"W2", "1099"}
        present = {d.document_type for d in self.documents}
        return required.issubset(present)

    def _calculate_balance_due(self) -> Money:
        income = sum(li.amount for li in self.line_items if li.is_income)
        deductions = sum(li.amount for li in self.line_items if li.is_deduction)
        return income.add(Money(-deductions.amount))
```

### Domain Events

Capture significant occurrences in the domain:

```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import List, Callable
import uuid

@dataclass(frozen=True)
class DomainEvent:
    """Base class for domain events."""
    event_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    occurred_at: datetime = field(default_factory=datetime.utcnow)

@dataclass(frozen=True)
class ClientCreatedEvent(DomainEvent):
    client_id: str
    email: str
    created_by: str

@dataclass(frozen=True)
class PaymentReceivedEvent(DomainEvent):
    client_id: str
    amount: Money
    payment_method: str

class EventDispatcher:
    """Simple event dispatcher."""

    def __init__(self):
        self._handlers: dict[type, List[Callable]] = {}

    def register(self, event_type: type, handler: Callable):
        if event_type not in self._handlers:
            self._handlers[event_type] = []
        self._handlers[event_type].append(handler)

    def dispatch(self, event: DomainEvent):
        handlers = self._handlers.get(type(event), [])
        for handler in handlers:
            handler(event)

# Usage
dispatcher = EventDispatcher()
dispatcher.register(ClientCreatedEvent, send_welcome_email)
dispatcher.register(ClientCreatedEvent, create_default_folders)
dispatcher.register(PaymentReceivedEvent, update_account_balance)
```

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| Repository pattern examples | `ali-ai-acctg/app/repositories/` |
| Service layer | `ali-ai-acctg/app/services/` |
| Factory patterns | `ali-ai-ingestion-engine/parsers/` |
| Builder patterns (Java) | `ali-ai-ingestion-engine/parsers/edi-common/` |
| Dependency injection setup | `ali-ai-acctg/app/dependencies.py` |
| Dataclass models | `ali-ai-ingestion-engine/sql_gen_build/data_models.py` |
| Test fixtures | `ali-ai-acctg/tests/conftest.py` |
| Test examples | `ali-ai-ingestion-engine/tests/` |

---

## References

- [Refactoring Guru - Design Patterns](https://refactoring.guru/design-patterns)
- [Python Design Patterns](https://python-patterns.guide/)
- [Effective Java (Bloch)](https://www.oreilly.com/library/view/effective-java/9780134686097/)
- [Clean Architecture (Martin)](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [pytest Documentation](https://docs.pytest.org/)
- [JUnit 5 User Guide](https://junit.org/junit5/docs/current/user-guide/)
- [Hypothesis Documentation](https://hypothesis.readthedocs.io/)
