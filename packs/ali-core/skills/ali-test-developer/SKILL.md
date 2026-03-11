---
name: ali-test-developer
description: |
  Test Developer for Tax Practice AI. Use when:

  PLANNING: Designing test strategy, planning test coverage, architecting
  fixture patterns, choosing test frameworks

  IMPLEMENTATION: Creating test files, adding test cases, implementing step
  definitions, writing fixtures, debugging test failures, fixing flaky tests

  GUIDANCE: Asking about test patterns, fixture management best practices,
  selector strategies, isolation techniques

  REVIEW: Checking test coverage, validating test isolation, reviewing
  data-testid usage, auditing fixture management, analyzing flaky tests

  Do NOT use for general testing strategy, framework selection, or test patterns
  across projects (use ali-testing instead). This skill is specific to Tax Practice AI.
---

# Test Developer - Tax Practice AI

## STOP - PRE-TEST VERIFICATION REQUIRED

**BEFORE writing ANY test that interacts with UI elements:**

### Step 1: Verify data-testid exists in component

Use Grep to check the component file:
```
Grep "data-testid" path/to/Component.tsx
```

### Step 2: If data-testid is MISSING

**DO NOT write the test yet.** First:
1. Open the component file
2. Add the required `data-testid` attribute
3. THEN write the test

### Step 3: Use ONLY data-testid selectors

```typescript
// CORRECT - the ONLY acceptable approach
await page.getByTestId('btn-save-client').click();

// WRONG - these will cause test failures and PR rejection
await page.getByText('Save').click();         // NEVER
await page.locator('.btn-primary').click();   // NEVER
await page.locator('button').click();         // NEVER
```

**Tests using text/class/tag selectors will be rejected. No exceptions.**

---

## When I Activate

This skill activates when working on:

- `tests/**/*` - Backend Python tests
- `frontend/tests/**/*` - Frontend Playwright tests
- `frontend/features/**/*` - BDD feature files
- `frontend/steps/**/*` - BDD step definitions
- `*.spec.ts` - Playwright spec files
- `*.feature` - Gherkin feature files
- `conftest.py` - Pytest configuration
- `playwright.config.ts` - Playwright configuration

---

## FIRST ACTION (MANDATORY)

Before ANY other action, check for project-specific testing documentation:

1. **Try reading `docs/TESTING.md`** (project-specific testing philosophy)
2. If not found, try `docs/TEST_PROTOCOL.md`
3. If neither exists, proceed with general best practices from this skill

**DO NOT proceed with writing tests until you've checked for project documentation.**

---

## CRITICAL: No Service Mocking (Integration Tests)

**This rule applies when project has a `docs/TESTING.md` with no-mocking policy.**

### ❌ FORBIDDEN in Integration Tests

```python
# tests/integration/**/*.py
from unittest.mock import patch, Mock, MagicMock  # ❌ REJECTED
from pytest_mock import mocker                     # ❌ REJECTED

# Example of WRONG approach - DO NOT DO THIS:
def test_workflow(self):
    with patch("src.services.SyncClientService") as mock:  # ❌ NO
        mock.return_value.create_client.return_value = {"id": "fake"}
        # This test gives FALSE CONFIDENCE - it only tests the mock!
```

**Why forbidden:**
- Mocked tests hide bugs - tests pass while real implementation is broken
- False positives waste time and erode trust
- You're testing the mock, not the actual code

### ✅ REQUIRED in Integration Tests

```python
# tests/integration/**/*.py
def test_workflow(client_service, test_id):  # ✅ CORRECT
    # Use REAL service with REAL PostgreSQL database
    client = client_service.create_client(
        email=f"test_{test_id}@test.local"  # Unique per test
    )
    assert client["id"]  # Real UUID from real database

# Real database provides real confidence
```

### Exception: AI Services Only

```python
# ✅ ONLY acceptable mocking: AI services (cost/non-determinism)
from unittest.mock import patch

@patch("src.services.ai_service.BedrockService")
def test_ai_workflow(mock_ai):
    """AI services can be mocked to avoid API costs."""
    mock_ai.return_value.generate.return_value = "mocked response"
    # Rest of test uses REAL services
```

**Why this exception exists:**
1. AI responses are non-deterministic
2. API costs would be prohibitive in CI
3. Testing for API contract compliance, not AI quality

### Pre-Integration-Test Checklist

Before writing integration tests:

- [ ] Read `docs/TESTING.md` for project-specific testing philosophy
- [ ] Find existing integration test as template (grep for similar service)
- [ ] Verify test will use REAL database (Docker PostgreSQL, not mocked)
- [ ] Verify test uses REAL business services (SyncClientService, etc.)
- [ ] Only mock AI services (Anthropic/Bedrock) if present
- [ ] Use `test_id` fixture for unique test data isolation

**If you import `unittest.mock` in an integration test for non-AI services, STOP and reconsider.**

---

## Golden Rules

### 1. Test Isolation is Non-Negotiable

Every test MUST be completely isolated. No test should depend on or affect another test.

**Backend (pytest):**
```python
# ALWAYS use test_id fixture for unique data
def test_create_client(test_id, client_factory):
    client_data = client_factory(test_id)  # Unique per test
    # test_id is a full UUID, embedded in email, names, etc.
```

**Frontend (Playwright):**
```typescript
// ALWAYS use TEST_ prefix for API fixture data
const response = await fetch('/v1/test/fixtures/pending-batch', {
  body: JSON.stringify({ prefix: 'TEST_' })
});

// ALWAYS filter assertions to TEST_ data when old data may exist
await expect(page.locator('[data-testid^="TEST_"]')).toHaveCount(3);
```

### 2. data-testid is MANDATORY (Zero Tolerance)

**THIS IS THE MOST VIOLATED RULE. PAY ATTENTION.**

Every single element interaction in tests MUST use `getByTestId()`. No exceptions.

```typescript
// THE ONLY ACCEPTABLE PATTERN
await page.getByTestId('btn-submit-form').click();
await page.getByTestId('client-row-123').click();
await page.getByTestId('input-email').fill('test@example.com');

// AUTOMATIC REJECTION - DO NOT USE THESE
await page.getByText('Submit').click();           // REJECTED
await page.getByRole('button').click();           // REJECTED
await page.locator('.btn-primary').click();       // REJECTED
await page.locator('div > button').click();       // REJECTED
await page.locator('button:has-text("Save")');    // REJECTED
```

**If component lacks data-testid: STOP. Add it to component FIRST.**

**Naming convention:** `{element-type}-{context}-{identifier}`
- `btn-save-client`
- `input-email-field`
- `row-client-{id}`
- `panel-ai-analysis`
- `card-document-{id}`

### 3. Fixtures are Single Source of Truth

Test data comes from YAML fixtures, not hardcoded in tests.

| Fixture File | Purpose |
|--------------|---------|
| `tests/fixtures/demo_users.yaml` | Staff users (stable UUIDs for FK constraints) |
| `tests/fixtures/demo_clients.yaml` | Demo clients |
| `tests/fixtures/demo_transmitters.yaml` | E-filing transmitters |

**Loading fixtures:**
```python
# Backend - use conftest.py helpers
@pytest.fixture
def sample_user_id():
    user = get_demo_staff_by_email("demo@taxpractice.ai")
    return UUID(user["id"])
```

### 4. Cleanup Happens AFTER Test, Not Before Polling

API fixtures create data that background polling may fetch. This causes cross-test contamination in parallel execution.

**Pattern for tests with background polling:**
```gherkin
# These tests MUST run individually or with --workers=1
@isolation-required
Scenario: Assign batch to existing client
```

```bash
# Run individually (recommended)
npm run test:bdd -- --grep "Assign batch to existing client"

# Or use serial execution
npm run test:bdd -- --workers=1
```

### 5. Scope Assertions to Current Context

When a page shows data from multiple sources, scope your assertions.

```typescript
// WRONG - may find elements from other tests or old data
await expect(page.locator('[data-testid="document-row"]')).toHaveCount(3);

// CORRECT - scoped to specific panel/section
const panel = page.getByTestId('panel-current-upload');
await expect(panel.locator('[data-testid="document-row"]')).toHaveCount(3);

// CORRECT - filtered to test data only
await expect(page.locator('[data-testid^="doc-TEST_"]')).toHaveCount(3);
```

---

## Backend Testing Patterns

### Test Structure
```
tests/
├── conftest.py              # Global fixtures, service init
├── fixtures/
│   ├── demo_users.yaml      # Staff users (stable IDs)
│   ├── demo_clients.yaml    # Demo clients
│   └── demo_transmitters.yaml
├── unit/                    # Fast, no I/O, mocked deps
├── integration/             # Real DB (Docker postgres)
└── e2e/                     # Full HTTP stack
```

### Required Fixtures

```python
# test_id - PRIMARY isolation mechanism
def test_something(test_id):
    email = f"test_{test_id}@unittest.it"  # Unique per test

# client_factory - Creates isolated client data
def test_client_creation(test_id, client_factory):
    data = client_factory(test_id, client_type="business")

# sample_user_id - Stable UUID for FK constraints
def test_with_user_fk(sample_user_id):
    record = {"created_by": sample_user_id}
```

### Test Tiers

| Tier | Location | DB | Speed | Purpose |
|------|----------|-----|-------|---------|
| Unit | tests/unit/ | Mock | <1s | Logic, validation, edge cases |
| Integration | tests/integration/ | Docker | 1-5s | Repository CRUD, service interactions |
| E2E | tests/e2e/ | Docker | 5-30s | Full API flows, auth, middleware |

---

## Frontend Testing Patterns

### Test Structure
```
frontend/
├── features/                # BDD feature files (Gherkin)
│   └── chapter-*/
│       └── *.feature
├── steps/                   # Step definitions
│   └── *.steps.ts
├── tests/
│   ├── lib/
│   │   ├── fixtures.ts      # Custom Playwright fixtures
│   │   ├── file-utils.ts    # Test file creation
│   │   ├── mode.ts          # ci/uat/record modes
│   │   └── test-config.ts   # Environment config
│   └── *.spec.ts            # Legacy spec files
└── playwright.config.ts
```

### Test Modes

| Mode | Command | Use Case |
|------|---------|----------|
| ci | `TEST_MODE=ci npx playwright test` | CI pipeline (parallel, fast) |
| uat | `TEST_MODE=uat npx playwright test` | UAT verification (serial, clean) |
| record | `TEST_MODE=record npx playwright test --headed` | Demo recording (slow, video) |

### File Upload Testing

**Workflow tests (recommended):** Use API fixtures
```typescript
// Fast, reliable, isolated
Given('I have uploaded {int} documents', async function(count) {
  await fetch('/v1/test/fixtures/pending-batch', {
    method: 'POST',
    body: JSON.stringify({ document_count: count })
  });
});
```

**UI tests:** Use file chooser pattern
```typescript
// Wait for file chooser BEFORE clicking
const fileChooserPromise = page.waitForEvent('filechooser');
await page.getByTestId('btn-browse-files').click();
const fileChooser = await fileChooserPromise;
await fileChooser.setFiles(testFiles);
```

---

## Anti-Patterns

| Anti-Pattern | Why Bad | Do Instead |
|--------------|---------|------------|
| **getByText()** | **REJECTED - fragile** | **Use getByTestId() ONLY** |
| **getByRole()** | **REJECTED - ambiguous** | **Use getByTestId() ONLY** |
| **locator('.class')** | **REJECTED - changes** | **Use getByTestId() ONLY** |
| **locator('tag')** | **REJECTED - fragile** | **Use getByTestId() ONLY** |
| **Mocking services in integration tests** | **FALSE CONFIDENCE - hides real bugs** | **Use real database/services** |
| Hardcoded test data | Breaks isolation | Use test_id fixture |
| Shared mutable state | Cross-test contamination | Fresh fixtures per test |
| Static file names | Hash collisions, duplicates | Unique content per file |
| Skipping cleanup | Pollutes subsequent tests | Always clean in After hooks |
| Parallel tests with polling | Documents leak between tests | Run individually or serial |
| Unscoped assertions | Finds wrong elements | Scope to panel/section |
| Missing FK user IDs | Constraint violations | Use sample_user_id fixture |

---

## Adding data-testid to Components

When tests need new selectors, add them to components:

```tsx
// Component (React)
<button data-testid="btn-save-client" onClick={handleSave}>
  Save
</button>

<div data-testid="panel-ai-analysis">
  {analysis.map(item => (
    <div key={item.id} data-testid={`row-analysis-${item.id}`}>
      {item.content}
    </div>
  ))}
</div>
```

**Naming patterns:**
- Buttons: `btn-{action}-{context}` → `btn-save-client`, `btn-delete-document`
- Inputs: `input-{field}` → `input-email`, `input-ssn`
- Rows: `row-{type}-{id}` → `row-client-123`, `row-document-456`
- Panels: `panel-{name}` → `panel-ai-analysis`, `panel-documents`
- Cards: `card-{type}-{id}` → `card-client-123`

---

## Quick Reference

### Running Tests

```bash
# Backend
pytest tests/unit -v                    # Unit tests only
pytest tests/integration -v             # Integration (needs Docker)
pytest tests/e2e -v                     # E2E (needs Docker)
pytest -k "test_client" -v              # Pattern match

# Frontend
npm run test:bdd                        # All BDD tests
npm run test:bdd -- --grep "@workflow"  # Tagged tests
npm run test:bdd -- --grep "scenario"   # Specific scenario
npm run test:bdd -- --workers=1         # Serial execution
npx playwright test --ui                # Interactive mode
```

### Debugging Failures

1. **Check test isolation** - Is test_id/TEST_ prefix used?
2. **Check selectors** - Are data-testids present and correct?
3. **Check scoping** - Is assertion scoped to correct panel?
4. **Check parallel issues** - Try running individually
5. **Check fixtures** - Are FK constraints satisfied?

### Related Documentation

- `docs/TEST_PROTOCOL.md` - Full testing guidelines
- `docs/TESTING_STRATEGY.md` - Frontend test strategy
- `docs/PENDING_UPLOADS_TESTS.md` - Parallel execution guide
- `tests/conftest.py` - Backend fixture definitions
- `frontend/tests/lib/fixtures.ts` - Frontend fixture extensions

---

## Recent Lessons Learned

From git history (2026-01):

1. **MOST COMMON FAILURE: Missing data-testid** - Tests repeatedly written using getByText(), getByRole(), or CSS selectors instead of getByTestId(). This causes flaky tests and test churn. **ALWAYS verify data-testid exists in component BEFORE writing the test. If missing, add it first.**

2. **Panel scoping** - AI Analysis tests failed because assertions found elements in wrong panel. Fix: scope to `panel.locator()`.

3. **Parallel contamination** - Pending uploads tests failed when run together. Fix: run individually or use --workers=1.

4. **Client row clicks** - Tests tried clicking non-existent buttons. Fix: click the row itself with `row-client-{id}`.

5. **FK constraints** - Chat sessions failed without user. Fix: use `sample_user_id` fixture for stable UUIDs.

6. **Forgetting to add data-testid to new components** - Components created without data-testid, then tests written with fragile selectors. **Check the verification checklist at the top of this skill before every edit.**
