# ali-streamlit - Testing Strategies Reference

Comprehensive testing patterns for Streamlit applications. Load when writing tests with AppTest or pytest.

---

## Unit Testing Pure Functions

**Extract business logic from UI for testability:**

```python
# src/workflows/chat/file_validators.py (pure functions)
def validate_file(
    content: bytes,
    filename: str,
    content_type: str,
) -> tuple[bool, str]:
    """Validate file for upload. Pure function - no side effects."""
    # Validation logic
    pass

def compute_file_hash(content: bytes) -> str:
    """Compute SHA-256 hash. Pure function."""
    import hashlib
    return hashlib.sha256(content).hexdigest()
```

**Test pure functions with pytest:**

```python
# tests/unit/test_file_validators.py
import pytest
from src.workflows.chat.file_validators import validate_file, compute_file_hash

class TestFileValidation:
    def test_size_limit_exceeded(self):
        content = b"x" * (51 * 1024 * 1024)  # 51MB
        is_valid, error = validate_file(content, "test.pdf", "application/pdf")
        assert not is_valid
        assert "exceeds 50MB" in error

    def test_invalid_extension(self):
        content = b"test"
        is_valid, error = validate_file(content, "test.exe", "application/x-executable")
        assert not is_valid
        assert "not allowed" in error

    def test_pdf_magic_bytes(self):
        content = b"not a pdf"
        is_valid, error = validate_file(content, "test.pdf", "application/pdf")
        assert not is_valid
        assert "does not match PDF format" in error

    def test_valid_pdf(self):
        content = b"%PDF-1.4\nTest PDF content"
        is_valid, error = validate_file(content, "test.pdf", "application/pdf")
        assert is_valid
        assert error == ""

class TestFileHashing:
    def test_compute_hash(self):
        content = b"test content"
        hash1 = compute_file_hash(content)
        hash2 = compute_file_hash(content)
        assert hash1 == hash2
        assert len(hash1) == 64  # SHA-256 hex

    def test_different_content_different_hash(self):
        hash1 = compute_file_hash(b"content1")
        hash2 = compute_file_hash(b"content2")
        assert hash1 != hash2
```

**Benefits:**
- Fast (no Streamlit app launching)
- Isolated (no session state dependencies)
- Easy to debug
- High coverage achievable

---

## Component Testing with AppTest

**Streamlit's built-in testing framework for UI components:**

### Login Component Test

```python
# tests/component/test_login_component.py
from streamlit.testing.v1 import AppTest

def test_login_success():
    at = AppTest.from_file("components/auth.py")
    at.run()

    # Input valid credentials
    at.text_input("email").input("staff@taxpractice.ai").run()
    at.text_input("password").input("correct_password").run()
    at.button("Login").click().run()

    # Verify success state
    assert at.session_state.authenticated == True
    assert at.session_state.current_user is not None

def test_login_lockout_after_five_attempts():
    at = AppTest.from_file("components/auth.py")
    at.run()

    # Simulate 5 failed attempts
    for i in range(5):
        at.text_input("email").input("test@example.com").run()
        at.text_input("password").input("wrongpassword").run()
        at.button("Login").click().run()

    # Verify lockout
    assert at.session_state.lockout_until is not None
    assert "Account locked" in at.error[0].value
```

### Form Component Test

```python
def test_client_form_validation():
    at = AppTest.from_file("components/client_form.py")
    at.run()

    # Submit without required fields
    at.button("Create Client").click().run()

    # Should show validation error
    assert at.error[0].value == "Name and email are required"

def test_client_form_success():
    at = AppTest.from_file("components/client_form.py")
    at.run()

    # Fill valid data
    at.text_input("name").input("John Doe").run()
    at.text_input("email").input("john@example.com").run()
    at.text_input("phone").input("555-1234").run()
    at.button("Create Client").click().run()

    # Verify success
    assert at.session_state.toast_message["message"] == "Client created successfully!"
```

---

### AppTest API Reference

**Common operations:**

```python
# Launch app
at = AppTest.from_file("app.py")
at.run()

# Interact with widgets
at.text_input("widget_key").input("value").run()
at.button("button_label").click().run()
at.selectbox("select_key").select("option").run()
at.checkbox("checkbox_key").check().run()
at.checkbox("checkbox_key").uncheck().run()
at.file_uploader("uploader_key").upload(file_bytes).run()

# Access session state
value = at.session_state.my_variable
assert at.session_state.authenticated == True

# Check UI elements
assert at.title[0].value == "My App"
assert at.header[0].value == "Welcome"
assert "success" in at.success[0].value
assert "error" in at.error[0].value
assert "warning" in at.warning[0].value
```

**Assertions:**

```python
# Session state assertions
assert at.session_state.key == expected_value

# UI content assertions
assert at.markdown[0].value == "Expected markdown"
assert at.button[0].label == "Submit"
assert at.text_input[0].label == "Email"

# Error/success message assertions
assert len(at.error) == 1  # One error message
assert "Invalid credentials" in at.error[0].value

# Widget state assertions
assert at.button[0].disabled == False
```

---

## Integration Testing

**Test service layer (database, external APIs):**

### Database Service Test

```python
# tests/integration/test_client_service.py
import pytest
from src.services.sync_client_service import SyncClientService

@pytest.fixture
def test_db():
    """Setup test database connection."""
    # Create test database, run migrations
    yield db_connection
    # Cleanup

def test_create_and_retrieve_client(test_db):
    service = SyncClientService()

    # Create client
    client = service.create(
        name="Test Client",
        email="test@example.com",
        phone="555-1234"
    )

    assert client["id"] is not None
    assert client["account_number"].startswith("CLI-")

    # Retrieve client
    retrieved = service.get_by_id(client["id"])
    assert retrieved["name"] == "Test Client"
```

### Async Bridge Integration Test

```python
# tests/integration/test_async_bridge.py
import pytest
from src.workflows.chat.streamlit_bridge import StreamlitAsyncBridge

@pytest.mark.asyncio
async def test_bridge_maintains_event_loop():
    bridge = StreamlitAsyncBridge()

    # First async operation
    result1 = bridge.run_flow(mock_flow, {"input": "test1"})
    assert result1 is not None

    # Second operation should reuse same loop
    result2 = bridge.run_flow(mock_flow, {"input": "test2"})
    assert result2 is not None

    # Verify no "Event loop is closed" error
```

---

## Test Organization

### Directory Structure

```
tests/
├── unit/                    # Pure function tests (fast)
│   ├── test_validators.py
│   ├── test_formatters.py
│   └── test_utils.py
├── component/               # UI component tests (AppTest)
│   ├── test_auth.py
│   ├── test_client_form.py
│   └── test_chat_panel.py
├── integration/             # Service layer tests (slower)
│   ├── test_client_service.py
│   ├── test_document_service.py
│   └── test_async_bridge.py
└── e2e/                     # End-to-end tests (slowest)
    └── test_full_workflow.py
```

### Fixture Organization

```python
# tests/conftest.py (shared fixtures)
import pytest
from src.services.sync_client_service import SyncClientService

@pytest.fixture
def mock_db():
    """Mock database connection for unit tests."""
    return MockDatabase()

@pytest.fixture
def test_db():
    """Real test database for integration tests."""
    # Setup
    db = setup_test_database()
    yield db
    # Teardown
    cleanup_test_database(db)

@pytest.fixture
def sample_client_data():
    """Reusable test data."""
    return {
        "name": "Test Client",
        "email": "test@example.com",
        "phone": "555-1234"
    }
```

---

## Mocking Patterns

### Mock External Dependencies

```python
# tests/unit/test_client_service.py
from unittest.mock import Mock, patch
import pytest

def test_create_client_success():
    # Mock database cursor
    mock_cursor = Mock()
    mock_cursor.fetchone.return_value = {"id": "123", "name": "Test"}

    with patch("psycopg2.connect") as mock_connect:
        mock_connect.return_value.cursor.return_value = mock_cursor

        service = SyncClientService()
        result = service.create("Test", "test@example.com")

        assert result["id"] == "123"
        mock_cursor.execute.assert_called_once()
```

### Mock Streamlit Session State

```python
# tests/component/test_with_session_state.py
from unittest.mock import Mock
import streamlit as st

def test_component_with_session_state(monkeypatch):
    # Mock session state
    mock_session = {
        "authenticated": True,
        "current_user": {"id": "123", "name": "Test User"}
    }
    monkeypatch.setattr(st, "session_state", mock_session)

    # Test component
    result = my_component()
    assert result is not None
```

---

## Test Data Management

### Factories for Test Data

```python
# tests/factories.py
from dataclasses import dataclass
from typing import Any

@dataclass
class ClientFactory:
    """Generate test client data."""
    name: str = "Test Client"
    email: str = "test@example.com"
    phone: str = "555-1234"

    def build(self, **kwargs) -> dict[str, Any]:
        """Build client dict with overrides."""
        data = {
            "name": self.name,
            "email": self.email,
            "phone": self.phone,
        }
        data.update(kwargs)
        return data

# Usage
def test_create_client():
    client_data = ClientFactory().build(name="John Doe")
    result = service.create(**client_data)
    assert result["name"] == "John Doe"
```

---

## Coverage Goals

**Recommended coverage targets:**

| Test Type | Coverage Target | Speed |
|-----------|-----------------|-------|
| Unit tests | 80%+ | Fast (ms) |
| Component tests | 60%+ | Medium (seconds) |
| Integration tests | 40%+ | Slow (seconds) |
| E2E tests | 20%+ | Very slow (minutes) |

**Run coverage:**

```bash
# All tests with coverage
pytest --cov=src --cov-report=html

# Unit tests only (fast feedback)
pytest tests/unit/ --cov=src --cov-report=term

# View HTML report
open htmlcov/index.html
```

---

## Continuous Integration

### GitHub Actions Configuration

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-python@v2
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-cov

      - name: Run unit tests
        run: pytest tests/unit/ --cov=src --cov-report=xml

      - name: Upload coverage
        uses: codecov/codecov-action@v2
```

---

## Testing Checklist

Use this checklist when writing tests:

**Unit Tests:**
- [ ] Pure functions extracted from UI
- [ ] All edge cases covered
- [ ] Happy path and error cases tested
- [ ] No Streamlit dependencies
- [ ] Fast execution (<100ms per test)

**Component Tests:**
- [ ] Widget interactions tested (input, click, select)
- [ ] Session state changes verified
- [ ] Error messages checked
- [ ] Success flows validated
- [ ] Isolated from external dependencies (mocked)

**Integration Tests:**
- [ ] Database operations tested (CRUD)
- [ ] External API calls tested (with real endpoints or stubs)
- [ ] Async operations tested (event loop handling)
- [ ] Error handling verified
- [ ] Cleanup after each test

**E2E Tests:**
- [ ] Complete user workflows tested
- [ ] Multiple components working together
- [ ] Realistic data used
- [ ] Performance acceptable
- [ ] Run in CI/CD pipeline

---

## Debugging Test Failures

### View AppTest Output

```python
def test_debug_output():
    at = AppTest.from_file("app.py")
    at.run()

    # Print all elements for debugging
    print(at.markdown)
    print(at.session_state)
    print(at.error)

    # Take screenshot (if supported)
    at.screenshot("debug.png")
```

### pytest Debug Flags

```bash
# Verbose output
pytest -v tests/

# Show print statements
pytest -s tests/

# Stop on first failure
pytest -x tests/

# Debug with pdb
pytest --pdb tests/

# Run specific test
pytest tests/unit/test_validators.py::TestFileValidation::test_size_limit
```

---

## References

- [Streamlit AppTest Documentation](https://docs.streamlit.io/library/api-reference/app-testing)
- [pytest Documentation](https://docs.pytest.org/)
- [unittest.mock Guide](https://docs.python.org/3/library/unittest.mock.html)
- [Coverage.py](https://coverage.readthedocs.io/)
- [ali-testing skill](../ali-testing/SKILL.md) - General testing patterns
