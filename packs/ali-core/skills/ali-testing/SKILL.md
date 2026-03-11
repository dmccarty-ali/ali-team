---
name: ali-testing
description: |
  Software testing practices, frameworks, and patterns. Use when:

  PLANNING: Designing test strategies, choosing testing frameworks, planning test
  coverage, architecting E2E or integration test suites, considering BDD approaches

  IMPLEMENTATION: Writing unit tests, integration tests, E2E tests, BDD scenarios,
  setting up test fixtures, mocking dependencies, implementing test utilities

  GUIDANCE: Asking about testing best practices, test patterns, framework selection,
  coverage strategies, AI-assisted testing, what to test vs what not to test

  REVIEW: Reviewing test quality, checking coverage gaps, validating test patterns,
  assessing test maintainability, evaluating test pyramid balance

  Do NOT use for Tax Practice AI project-specific test implementation, fixtures,
  or data-testid patterns (use ali-test-developer instead)
---

# Testing

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing test strategies or test architecture
- Choosing testing frameworks (pytest, JUnit, Playwright, etc.)
- Planning test coverage and test pyramid balance
- Architecting E2E, integration, or BDD test suites
- Deciding what level of testing to apply

**Implementation:**
- Writing unit, integration, or E2E tests
- Creating BDD scenarios (Gherkin/Cucumber)
- Setting up test fixtures and factories
- Mocking, stubbing, or faking dependencies
- Implementing test utilities and helpers

**Guidance/Best Practices:**
- Asking about testing patterns and anti-patterns
- Framework selection and comparison
- AI-assisted test generation
- What to test vs what to skip
- Flaky test prevention

**Review/Validation:**
- Reviewing test quality and coverage
- Assessing test maintainability
- Validating test isolation
- Checking test pyramid balance

---

## Key Principles

- **Test behavior, not implementation** - Tests should verify outcomes, not internal details
- **Fast feedback** - Unit tests run in milliseconds, integration in seconds, E2E in minutes
- **Isolated and independent** - Tests don't depend on each other or shared state
- **Deterministic** - Same input always produces same result (no flakiness)
- **Readable as documentation** - Test names describe expected behavior
- **Maintainable** - Easy to update when requirements change

---

## Test Pyramid

```
        /\
       /  \      E2E Tests (few)
      /----\     - Full system, slow, expensive
     /      \    - Critical user journeys only
    /--------\
   /          \  Integration Tests (some)
  /------------\ - Component boundaries
 /              \- API contracts, database
/----------------\
                  Unit Tests (many)
                  - Fast, isolated, focused
                  - Business logic, utilities
```

**Balance:**
- ~70% unit tests (fast, cheap, many)
- ~20% integration tests (medium speed, some)
- ~10% E2E tests (slow, expensive, few)

---

## Frameworks by Language

### Python

| Framework | Use Case |
|-----------|----------|
| **pytest** | Unit/integration (preferred) |
| **pytest-bdd** | BDD/Gherkin with pytest integration |
| **pytest-asyncio** | Async code testing |
| **behave** | BDD/Gherkin (standalone) |
| **hypothesis** | Property-based testing |
| **unittest** | Standard library, legacy |
| **responses/httpretty** | HTTP mocking |
| **freezegun** | Time mocking |
| **factory_boy** | Test data factories |
| **faker** | Fake data generation |
| **localstack** | AWS service mocking |

**pytest patterns:**
```python
# Fixtures for setup/teardown
@pytest.fixture
def database_connection():
    conn = create_connection()
    yield conn
    conn.close()

# Parametrized tests
@pytest.mark.parametrize("input,expected", [
    ("valid", True),
    ("invalid", False),
])
def test_validation(input, expected):
    assert validate(input) == expected

# Markers for categorization
@pytest.mark.slow
@pytest.mark.integration
def test_full_pipeline():
    ...
```

### Java

| Framework | Use Case |
|-----------|----------|
| **JUnit 5** | Unit/integration (preferred) |
| **Mockito** | Mocking framework |
| **Cucumber** | BDD/Gherkin |
| **AssertJ** | Fluent assertions |
| **Testcontainers** | Integration with containers |
| **WireMock** | HTTP mocking |
| **ArchUnit** | Architecture testing |
| **Maven Surefire** | Test runner plugin |
| **LocalStack** | AWS service mocking |

**JUnit 5 patterns:**
```java
// Nested test classes for organization
@Nested
class WhenUserIsValid {
    @Test
    void shouldReturnSuccess() { ... }
}

// Parameterized tests
@ParameterizedTest
@CsvSource({"valid,true", "invalid,false"})
void testValidation(String input, boolean expected) { ... }

// Lifecycle hooks
@BeforeEach
void setUp() { ... }

@AfterEach
void tearDown() { ... }
```

**JUnit 5 advanced features:**
```java
// Conditional test execution - skip tests based on conditions
@EnabledIf("isLocalStackRunning")
@Test
void testS3Integration() { ... }

boolean isLocalStackRunning() {
    // Check if LocalStack container is available
    return System.getenv("LOCALSTACK_HOSTNAME") != null;
}

// Temporary directory for file-based tests
@Test
void testFileProcessing(@TempDir Path tempDir) {
    Path testFile = tempDir.resolve("test.txt");
    Files.writeString(testFile, "test content");
    // tempDir is automatically cleaned up after test
}

// Shared test instance for expensive setup
@TestInstance(Lifecycle.PER_CLASS)
class IntegrationTests {
    private Connection connection;

    @BeforeAll
    void setUp() {
        // Runs once for all tests in this class
        connection = createExpensiveConnection();
    }

    @AfterAll
    void tearDown() {
        connection.close();
    }
}
```

**Maven Surefire configuration (pom.xml):**
```xml
<!-- Maven Surefire Plugin for running tests -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.2.2</version>
    <configuration>
        <!-- Include test patterns -->
        <includes>
            <include>**/*Test.java</include>
        </includes>
        <!-- Exclude integration tests from unit test phase -->
        <excludes>
            <exclude>**/*IntegrationTest.java</exclude>
        </excludes>
        <!-- Parallel execution -->
        <parallel>methods</parallel>
        <threadCount>4</threadCount>
    </configuration>
</plugin>
```

### JavaScript/TypeScript

| Framework | Use Case |
|-----------|----------|
| **Jest** | Unit/integration (React, Node) |
| **Vitest** | Fast alternative to Jest |
| **Playwright** | E2E browser testing |
| **playwright-bdd** | BDD/Gherkin with Playwright |
| **@axe-core/playwright** | Accessibility testing |
| **Cypress** | E2E browser testing |
| **Testing Library** | Component testing |
| **MSW** | API mocking |
| **@faker-js/faker** | Test data generation |

---

## Unit Testing Patterns

### Arrange-Act-Assert (AAA)

```python
def test_calculate_total():
    # Arrange - set up test data
    cart = ShoppingCart()
    cart.add_item(Product("Widget", 10.00), quantity=2)

    # Act - execute the behavior
    total = cart.calculate_total()

    # Assert - verify the outcome
    assert total == 20.00
```

### Given-When-Then (BDD style)

```python
def test_user_registration():
    # Given a new user with valid credentials
    user_data = {"email": "test@example.com", "password": "secure123"}

    # When they register
    result = register_user(user_data)

    # Then registration succeeds
    assert result.success is True
    assert result.user.email == "test@example.com"
```

### Test Doubles

| Type | Purpose | Example |
|------|---------|---------|
| **Stub** | Return canned responses | `stub.get_user.return_value = User(...)` |
| **Mock** | Verify interactions | `mock.send_email.assert_called_once()` |
| **Fake** | Working implementation | In-memory database |
| **Spy** | Record calls, delegate to real | Wrap real object, track calls |

```python
# Mocking with pytest-mock
def test_sends_welcome_email(mocker):
    mock_email = mocker.patch('app.email.send_email')

    register_user({"email": "test@example.com"})

    mock_email.assert_called_once_with(
        to="test@example.com",
        template="welcome"
    )
```

---

## Integration Testing Patterns

### Database Testing

```python
# Use transactions for isolation
@pytest.fixture
def db_session():
    session = Session()
    session.begin_nested()  # Savepoint
    yield session
    session.rollback()      # Rollback after test

# Or use test containers
@pytest.fixture(scope="session")
def postgres_container():
    with PostgresContainer("postgres:15") as postgres:
        yield postgres.get_connection_url()
```

### API Testing

```python
# Test FastAPI with TestClient
from fastapi.testclient import TestClient

def test_create_user(client: TestClient):
    response = client.post("/users", json={"email": "test@example.com"})
    assert response.status_code == 201
    assert response.json()["email"] == "test@example.com"
```

### External Service Mocking

```python
# Mock HTTP calls with responses
import responses

@responses.activate
def test_external_api_call():
    responses.add(
        responses.GET,
        "https://api.example.com/data",
        json={"result": "success"},
        status=200
    )

    result = fetch_external_data()
    assert result == {"result": "success"}
```

### Async Testing (pytest-asyncio)

Testing async code requires special handling:

```python
import pytest
from unittest.mock import AsyncMock, MagicMock, patch

# Mark entire module as async
pytestmark = pytest.mark.asyncio

# Basic async test
async def test_async_function():
    result = await fetch_data_async()
    assert result is not None

# AsyncMock for async dependencies
async def test_service_with_async_mock():
    # AsyncMock automatically returns coroutines
    mock_client = AsyncMock()
    mock_client.get_user.return_value = {"id": 1, "name": "Test"}

    service = UserService(client=mock_client)
    user = await service.get_user(1)

    assert user["name"] == "Test"
    mock_client.get_user.assert_called_once_with(1)

# Patching async methods
async def test_patched_async_call():
    with patch('myapp.client.fetch', new_callable=AsyncMock) as mock_fetch:
        mock_fetch.return_value = {"status": "ok"}

        result = await process_request()

        assert result["status"] == "ok"

# Async fixtures
@pytest.fixture
async def async_db_connection():
    conn = await asyncpg.connect(DATABASE_URL)
    yield conn
    await conn.close()

# Using async fixtures
async def test_with_async_fixture(async_db_connection):
    result = await async_db_connection.fetch("SELECT 1")
    assert result[0][0] == 1
```

**asyncpg database fixtures:**
```python
import asyncpg
import pytest

@pytest.fixture(scope="session")
def event_loop():
    # Required for session-scoped async fixtures
    import asyncio
    loop = asyncio.get_event_loop_policy().new_event_loop()
    yield loop
    loop.close()

@pytest.fixture(scope="session")
async def db_pool():
    pool = await asyncpg.create_pool(
        host="localhost",
        database="test_db",
        user="test_user",
        password="test_pass",
        min_size=2,
        max_size=10
    )
    yield pool
    await pool.close()

@pytest.fixture
async def db_connection(db_pool):
    async with db_pool.acquire() as conn:
        # Start transaction for test isolation
        tr = conn.transaction()
        await tr.start()
        yield conn
        # Rollback after test
        await tr.rollback()
```

### LocalStack (AWS Service Mocking)

LocalStack provides local AWS service emulation for integration tests:

**Java with LocalStack:**
```java
// Test with LocalStack S3
@TestInstance(Lifecycle.PER_CLASS)
@EnabledIf("isLocalStackRunning")
class S3IntegrationTest {

    private S3Client s3Client;
    private static final String BUCKET_NAME = "test-bucket";

    @BeforeAll
    void setUp() {
        // Connect to LocalStack S3
        s3Client = S3Client.builder()
            .endpointOverride(URI.create("http://localhost:4566"))
            .region(Region.US_EAST_1)
            .credentialsProvider(StaticCredentialsProvider.create(
                AwsBasicCredentials.create("test", "test")))
            .build();

        // Create test bucket
        s3Client.createBucket(b -> b.bucket(BUCKET_NAME));
    }

    @Test
    void testUploadAndDownload() {
        // Upload
        s3Client.putObject(
            PutObjectRequest.builder()
                .bucket(BUCKET_NAME)
                .key("test.txt")
                .build(),
            RequestBody.fromString("test content")
        );

        // Download and verify
        ResponseInputStream<GetObjectResponse> response = s3Client.getObject(
            GetObjectRequest.builder()
                .bucket(BUCKET_NAME)
                .key("test.txt")
                .build()
        );
        String content = new String(response.readAllBytes());
        assertEquals("test content", content);
    }

    boolean isLocalStackRunning() {
        // Check if LocalStack is available
        try {
            new URL("http://localhost:4566/_localstack/health")
                .openConnection().connect();
            return true;
        } catch (Exception e) {
            return false;
        }
    }
}
```

**Python with LocalStack:**
```python
import boto3
import pytest
from moto import mock_s3  # Alternative: moto for in-process mocking

# Using LocalStack directly
@pytest.fixture
def s3_client():
    return boto3.client(
        's3',
        endpoint_url='http://localhost:4566',
        aws_access_key_id='test',
        aws_secret_access_key='test',
        region_name='us-east-1'
    )

@pytest.fixture
def test_bucket(s3_client):
    bucket_name = 'test-bucket'
    s3_client.create_bucket(Bucket=bucket_name)
    yield bucket_name
    # Cleanup
    s3_client.delete_bucket(Bucket=bucket_name)

def test_s3_upload(s3_client, test_bucket):
    s3_client.put_object(
        Bucket=test_bucket,
        Key='test.txt',
        Body=b'test content'
    )

    response = s3_client.get_object(Bucket=test_bucket, Key='test.txt')
    assert response['Body'].read() == b'test content'
```

**docker-compose.yml for LocalStack:**
```yaml
# docker-compose.yml for local development and testing
version: '3.8'
services:
  localstack:
    image: localstack/localstack:latest
    ports:
      - "4566:4566"           # LocalStack Gateway
    environment:
      - SERVICES=s3,sqs,sns   # Services to enable
      - DEBUG=1
      - DATA_DIR=/tmp/localstack/data
    volumes:
      - "./localstack:/tmp/localstack"
      - "/var/run/docker.sock:/var/run/docker.sock"
```

### Airflow DAG Testing

Test Airflow DAGs for integrity and configuration:

```python
import pytest
from airflow.models import DagBag

# Load all DAGs once
@pytest.fixture(scope="session")
def dagbag():
    return DagBag(dag_folder="dags/", include_examples=False)

def test_no_import_errors(dagbag):
    """Verify DAGs have no import errors."""
    assert len(dagbag.import_errors) == 0, \
        f"DAG import errors: {dagbag.import_errors}"

def test_dag_loaded(dagbag):
    """Verify expected DAGs are loaded."""
    expected_dags = ['etl_pipeline', 'data_sync', 'cleanup_job']
    for dag_id in expected_dags:
        assert dag_id in dagbag.dags, f"DAG {dag_id} not found"

def test_dag_has_tags(dagbag):
    """Verify all DAGs have required tags."""
    for dag_id, dag in dagbag.dags.items():
        assert dag.tags, f"DAG {dag_id} has no tags"
        assert any(tag in dag.tags for tag in ['production', 'development']), \
            f"DAG {dag_id} missing environment tag"

def test_dag_has_retries(dagbag):
    """Verify tasks have appropriate retry configuration."""
    for dag_id, dag in dagbag.dags.items():
        for task in dag.tasks:
            assert task.retries >= 1, \
                f"Task {task.task_id} in {dag_id} has no retries configured"

def test_dag_schedule_interval(dagbag):
    """Verify DAGs have valid schedule intervals."""
    for dag_id, dag in dagbag.dags.items():
        # Skip manually triggered DAGs
        if dag.schedule_interval is None:
            continue
        # Verify schedule is not too frequent for production
        assert dag.schedule_interval != '* * * * *', \
            f"DAG {dag_id} runs every minute - too frequent"

def test_dag_email_on_failure(dagbag):
    """Verify production DAGs have alerting configured."""
    for dag_id, dag in dagbag.dags.items():
        if 'production' in dag.tags:
            assert dag.default_args.get('email_on_failure', False), \
                f"Production DAG {dag_id} missing email alerts"
```

**Testing individual operators:**
```python
from airflow.models import TaskInstance
from airflow.utils.state import State
import pendulum

def test_python_operator_execution(dagbag):
    """Test that a PythonOperator executes correctly."""
    dag = dagbag.get_dag('etl_pipeline')
    task = dag.get_task('process_data')

    # Create a task instance
    execution_date = pendulum.now()
    ti = TaskInstance(task=task, execution_date=execution_date)

    # Run the task
    ti.run(ignore_ti_state=True)

    assert ti.state == State.SUCCESS
```

---

## E2E Testing Patterns

### Playwright (Python)

```python
from playwright.sync_api import Page, expect

def test_user_login_flow(page: Page):
    # Navigate
    page.goto("/login")

    # Fill form
    page.fill('[data-testid="email"]', "user@example.com")
    page.fill('[data-testid="password"]', "password123")
    page.click('[data-testid="submit"]')

    # Verify redirect and content
    expect(page).to_have_url("/dashboard")
    expect(page.locator("h1")).to_contain_text("Welcome")
```

### E2E Best Practices

- **Use data-testid attributes** - Don't rely on CSS classes or text
- **Wait for elements properly** - Use explicit waits, not sleep
- **Isolate test data** - Each test creates its own data
- **Run in CI with retries** - E2E tests can be flaky
- **Record videos on failure** - Helps debug issues
- **Keep E2E tests focused** - Test critical paths only

### E2E Test Modes

Configure different execution modes for different contexts (CI, UAT, demos):

```typescript
// playwright.config.ts with test modes
const TEST_MODE = process.env.TEST_MODE || 'ci';

const modeConfigs = {
    // CI mode: fast, parallel, headless
    ci: {
        workers: 4,
        retries: 2,
        use: {
            headless: true,
            video: 'retain-on-failure',
            screenshot: 'only-on-failure',
        },
    },
    // UAT mode: serial, clean output for stakeholders
    uat: {
        workers: 1,
        retries: 0,
        use: {
            headless: true,
            video: 'off',
            screenshot: 'only-on-failure',
        },
    },
    // Record mode: video capture for demos/documentation
    record: {
        workers: 1,
        retries: 0,
        use: {
            headless: false,
            video: 'on',
            screenshot: 'on',
            launchOptions: { slowMo: 500 },  // Slow for visibility
        },
    },
};

export default defineConfig({
    ...modeConfigs[TEST_MODE],
    testDir: './tests',
});
```

**package.json scripts:**
```json
{
    "scripts": {
        "test": "TEST_MODE=ci playwright test",
        "test:uat": "TEST_MODE=uat playwright test --project=chromium",
        "test:record": "TEST_MODE=record playwright test --project=chromium --headed",
        "test:smoke": "TEST_MODE=ci playwright test --grep @smoke",
        "test:a11y": "TEST_MODE=ci playwright test --grep @a11y"
    }
}
```

### Accessibility Testing (@axe-core/playwright)

Automated accessibility testing integrated with Playwright:

```typescript
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test.describe('Accessibility', () => {
    test('login page should have no accessibility violations @a11y', async ({ page }) => {
        await page.goto('/login');

        const accessibilityScanResults = await new AxeBuilder({ page })
            .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])
            .analyze();

        expect(accessibilityScanResults.violations).toEqual([]);
    });

    test('dashboard should be accessible @a11y', async ({ page }) => {
        await page.goto('/dashboard');

        // Exclude known problematic third-party components
        const results = await new AxeBuilder({ page })
            .exclude('.third-party-widget')
            .analyze();

        // Log violations for debugging
        if (results.violations.length > 0) {
            console.log('Accessibility violations:', JSON.stringify(results.violations, null, 2));
        }

        expect(results.violations).toEqual([]);
    });
});
```

**Accessibility testing best practices:**
- Run on all critical pages
- Use @a11y tag for filtering
- Test with different viewport sizes
- Check color contrast, keyboard navigation, screen reader compatibility
- Exclude third-party components you can't control

### Test Tagging Strategy

Use tags to organize and filter tests:

```typescript
// Playwright test tagging
test('critical user flow @smoke @critical', async ({ page }) => { ... });
test('accessibility check @a11y', async ({ page }) => { ... });
test('performance test @perf @slow', async ({ page }) => { ... });
test('regression test @regression', async ({ page }) => { ... });

// Run specific tags
// npx playwright test --grep @smoke
// npx playwright test --grep "@smoke|@critical"
// npx playwright test --grep-invert @slow
```

```python
# pytest markers
@pytest.mark.smoke
@pytest.mark.critical
def test_user_login():
    ...

@pytest.mark.slow
@pytest.mark.integration
def test_full_data_pipeline():
    ...

# Run specific markers
# pytest -m smoke
# pytest -m "smoke or critical"
# pytest -m "not slow"
```

---

## BDD Testing (Behavior-Driven Development)

### Gherkin Syntax

```gherkin
Feature: User Authentication
  As a registered user
  I want to log into my account
  So that I can access my dashboard

  Background:
    Given a registered user exists with email "user@example.com"

  Scenario: Successful login
    Given I am on the login page
    When I enter valid credentials
    And I click the login button
    Then I should be redirected to the dashboard
    And I should see a welcome message

  Scenario Outline: Invalid login attempts
    Given I am on the login page
    When I enter "<email>" and "<password>"
    Then I should see error "<error_message>"

    Examples:
      | email              | password | error_message        |
      | wrong@example.com  | password | Invalid credentials  |
      | user@example.com   | wrong    | Invalid credentials  |
      | invalid-email      | password | Invalid email format |
```

### Behave (Python BDD)

```python
# features/steps/auth_steps.py
from behave import given, when, then

@given('I am on the login page')
def step_on_login_page(context):
    context.page.goto("/login")

@when('I enter valid credentials')
def step_enter_credentials(context):
    context.page.fill('[data-testid="email"]', "user@example.com")
    context.page.fill('[data-testid="password"]', "password123")

@then('I should be redirected to the dashboard')
def step_redirected_to_dashboard(context):
    assert context.page.url.endswith("/dashboard")
```

### Cucumber (Java BDD)

```java
public class AuthSteps {
    @Given("I am on the login page")
    public void navigateToLogin() {
        driver.get(baseUrl + "/login");
    }

    @When("I enter valid credentials")
    public void enterCredentials() {
        driver.findElement(By.cssSelector("[data-testid='email']"))
              .sendKeys("user@example.com");
    }
}
```

### pytest-bdd (Python BDD with pytest integration)

pytest-bdd integrates BDD directly into pytest, allowing Gherkin features with pytest fixtures and markers.

```python
# tests/bdd/features/auth.feature
Feature: User Authentication
    Scenario: Successful login
        Given a registered user exists
        When the user logs in with valid credentials
        Then the user should see the dashboard

# tests/bdd/step_defs/test_auth_steps.py
import pytest
from pytest_bdd import scenarios, given, when, then, parsers

# Load all scenarios from feature file
scenarios('../features/auth.feature')

@given('a registered user exists')
def registered_user(db_session):
    # Use pytest fixtures directly
    return create_test_user(db_session)

@when('the user logs in with valid credentials')
def login_user(client, registered_user):
    response = client.post('/login', json={
        'email': registered_user.email,
        'password': 'test_password'
    })
    return response

@then('the user should see the dashboard')
def verify_dashboard(login_user):
    assert login_user.status_code == 200
    assert 'dashboard' in login_user.json()['redirect']

# Parameterized scenarios with parsers
@given(parsers.parse('a user with role "{role}"'))
def user_with_role(role):
    return create_user(role=role)
```

**pytest-bdd vs behave:**
| Aspect | pytest-bdd | behave |
|--------|------------|--------|
| Integration | Native pytest | Standalone |
| Fixtures | pytest fixtures | behave context |
| Markers | pytest markers | behave tags |
| Parallel | pytest-xdist | Limited |
| Reporting | pytest plugins | behave formatters |

### playwright-bdd (TypeScript/JavaScript BDD with Playwright)

Combines Playwright's browser automation with BDD Gherkin syntax.

```typescript
// features/login.feature
Feature: Login Page
    @smoke
    Scenario: User can log in
        Given I am on the login page
        When I enter valid credentials
        And I click the login button
        Then I should see the dashboard

// steps/login.steps.ts
import { createBdd } from 'playwright-bdd';
import { expect } from '@playwright/test';

const { Given, When, Then } = createBdd();

Given('I am on the login page', async ({ page }) => {
    await page.goto('/login');
});

When('I enter valid credentials', async ({ page }) => {
    await page.fill('[data-testid="email"]', 'user@example.com');
    await page.fill('[data-testid="password"]', 'password123');
});

When('I click the login button', async ({ page }) => {
    await page.click('[data-testid="submit"]');
});

Then('I should see the dashboard', async ({ page }) => {
    await expect(page).toHaveURL(/dashboard/);
    await expect(page.locator('h1')).toContainText('Welcome');
});
```

**playwright.config.ts with BDD:**
```typescript
import { defineConfig } from '@playwright/test';
import { defineBddConfig } from 'playwright-bdd';

const testDir = defineBddConfig({
    features: 'features/**/*.feature',
    steps: 'steps/**/*.ts',
});

export default defineConfig({
    testDir,
    use: {
        baseURL: 'http://localhost:3000',
    },
});
```

---

## AI-Assisted Testing

### Test Generation with AI

**Effective prompts:**
- "Generate unit tests for this function covering edge cases"
- "Write integration tests for this API endpoint"
- "Create BDD scenarios for this user story"
- "Suggest test cases I might have missed"

**AI testing workflow:**
1. Write implementation
2. Ask AI to generate test cases
3. Review and refine generated tests
4. Add edge cases AI missed
5. Ensure tests are maintainable

### AI-Powered Test Analysis

- **Coverage gap analysis** - "What scenarios aren't covered?"
- **Mutation testing suggestions** - "What mutations would survive?"
- **Flaky test detection** - "Why might this test be flaky?"
- **Test refactoring** - "How can I make this test more maintainable?"

---

## Property-Based Testing

Instead of specific examples, define properties that should always hold:

```python
from hypothesis import given, strategies as st

@given(st.lists(st.integers()))
def test_sort_maintains_length(lst):
    sorted_lst = sorted(lst)
    assert len(sorted_lst) == len(lst)

@given(st.lists(st.integers()))
def test_sort_is_idempotent(lst):
    once = sorted(lst)
    twice = sorted(once)
    assert once == twice

@given(st.integers(), st.integers())
def test_addition_commutative(a, b):
    assert a + b == b + a
```

---

## Test Data Management

### Factories (factory_boy)

```python
import factory
from faker import Faker

fake = Faker()

class UserFactory(factory.Factory):
    class Meta:
        model = User

    email = factory.LazyAttribute(lambda _: fake.email())
    name = factory.LazyAttribute(lambda _: fake.name())
    created_at = factory.LazyFunction(datetime.utcnow)

# Usage
user = UserFactory()                    # Random data
user = UserFactory(email="specific@example.com")  # Override
users = UserFactory.create_batch(10)    # Multiple
```

### Fixtures Organization

```
tests/
├── conftest.py          # Shared fixtures
├── factories/
│   ├── __init__.py
│   ├── user_factory.py
│   └── order_factory.py
├── fixtures/
│   ├── sample_data.json
│   └── test_files/
├── unit/
├── integration/
└── e2e/
```

---

## Anti-Patterns to Avoid

| Anti-Pattern | Problem | Better Approach |
|--------------|---------|-----------------|
| **Testing implementation** | Brittle, breaks on refactor | Test behavior/outcomes |
| **Shared mutable state** | Tests affect each other | Isolate test data |
| **Sleep in tests** | Slow, still flaky | Use explicit waits |
| **Too much mocking** | Tests don't reflect reality | Integration tests for boundaries |
| **Giant test methods** | Hard to debug failures | One assertion per concept |
| **No test organization** | Can't run subsets | Use markers/categories |
| **Ignoring flaky tests** | Erodes trust in suite | Fix or delete immediately |
| **100% coverage goal** | Diminishing returns | Cover critical paths well |

---

## Quick Reference

### pytest commands
```bash
pytest                          # Run all tests
pytest tests/unit/              # Run subset
pytest -k "test_user"           # Run matching name
pytest -m slow                  # Run by marker
pytest --cov=app                # With coverage
pytest -x                       # Stop on first failure
pytest --pdb                    # Debug on failure
pytest -v                       # Verbose output
```

### Test naming conventions
```python
# Function-based (pytest)
def test_user_registration_with_valid_email_succeeds():
def test_user_registration_with_duplicate_email_fails():

# Class-based organization
class TestUserRegistration:
    def test_with_valid_email_succeeds(self):
    def test_with_duplicate_email_fails(self):
```

### Coverage targets
- **Critical business logic**: 90%+
- **Utilities and helpers**: 80%+
- **Infrastructure/glue code**: 60%+
- **Generated code**: Skip

---

## References

- [pytest Documentation](https://docs.pytest.org/)
- [JUnit 5 User Guide](https://junit.org/junit5/docs/current/user-guide/)
- [Playwright Documentation](https://playwright.dev/python/)
- [Behave Documentation](https://behave.readthedocs.io/)
- [Hypothesis (Property-Based Testing)](https://hypothesis.readthedocs.io/)
- [Testing Trophy (Kent C. Dodds)](https://kentcdodds.com/blog/the-testing-trophy-and-testing-classifications)
