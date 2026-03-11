---
name: ali-streamlit
description: |
  Streamlit app development patterns and best practices. Use when:

  PLANNING: Designing Streamlit apps, planning state management, architecting
  multi-page apps, evaluating component choices, considering deployment strategies,
  planning async integration with LangGraph/asyncpg

  IMPLEMENTATION: Building Streamlit apps, implementing caching, creating layouts,
  handling user input, managing session state, adding authentication, optimizing
  performance, integrating with async services, implementing file uploads

  GUIDANCE: Asking about Streamlit best practices, state management patterns,
  caching strategies, deployment options, component usage, performance optimization,
  accessibility considerations, security patterns, testing approaches

  REVIEW: Reviewing Streamlit code for anti-patterns, checking state management,
  validating caching strategy, auditing performance, checking security patterns,
  validating accessibility compliance, reviewing code organization

  DO NOT USE FOR:
  - General security principles (use ali-secure-coding for OWASP, compliance frameworks)
  - Non-Streamlit frontend frameworks (use ali-frontend-developer for React/Next.js)
  - Backend API development (use ali-backend-developer for FastAPI/Flask)
  - Generic testing patterns (use ali-testing for test strategies)
  - LangGraph workflow design (use ali-langgraph for graph architecture)
  - Async Python patterns outside Streamlit context (use ali-backend-developer)
---

# Streamlit Development

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing Streamlit applications and user interfaces
- Planning state management strategies
- Architecting multi-page apps with navigation
- Evaluating layout choices (sidebar, columns, tabs, containers)
- Planning async integration (LangGraph, asyncpg, external APIs)
- Considering deployment strategies (Streamlit Cloud, Docker, AWS)

**Implementation:**
- Building Streamlit apps with forms and widgets
- Implementing session state management
- Creating custom layouts and themes
- Adding authentication and authorization
- Handling file uploads with validation
- Integrating async operations (database, AI workflows)
- Implementing caching strategies
- Adding accessibility features

**Guidance/Best Practices:**
- Asking about Streamlit best practices
- Asking about state management patterns
- Asking about caching strategies (@st.cache_data vs @st.cache_resource)
- Asking about performance optimization
- Asking about security patterns (authentication, file uploads, audit logging)
- Asking about testing strategies for Streamlit apps
- Asking about accessibility compliance (WCAG 2.1 AA)

**Review/Validation:**
- Reviewing Streamlit code for anti-patterns
- Checking session state management
- Validating caching strategy
- Auditing performance and rerun optimization
- Checking security patterns (auth, file validation, audit logging)
- Validating accessibility compliance
- Reviewing code organization (god files, component extraction)

---

## Key Principles

- **Rerun behavior is fundamental**: Every interaction reruns the entire script from top to bottom - design accordingly
- **Session state is your friend**: Use `st.session_state` for persistence across reruns, not global variables
- **Cache aggressively**: Use `@st.cache_data` for data transformations, `@st.cache_resource` for connections
- **Extract components early**: Keep files under 500 lines - extract reusable components to separate modules
- **Async bridge for I/O**: Use event loop management pattern for asyncpg, LangGraph, and async operations
- **Security is not built-in**: Implement authentication, file validation, audit logging, and session timeouts yourself
- **Accessibility requires effort**: Streamlit has limited ARIA support - use semantic HTML, focus indicators, and WCAG compliance
- **Testing requires discipline**: Extract business logic from UI code for unit testing, use AppTest for component testing
- **Forms prevent chaos**: Use `st.form()` to batch inputs and prevent premature reruns
- **Professional design matters**: Custom CSS with design systems, WCAG contrast ratios, and brand consistency

---

## Execution Model

### Rerun Behavior

Streamlit reruns the **entire script from top to bottom** on every user interaction (button click, text input change, etc.). This is fundamentally different from traditional web frameworks.

**Implications:**
- Expensive operations in main script body run on every interaction
- Widget state persists automatically (Streamlit manages it)
- Session state persists across reruns (manual management required)
- Event loop handling requires special patterns (see Async Bridge)

**Best Practice:**
```python
# ❌ BAD: Expensive operation runs on every interaction
data = expensive_database_query()  # Runs every time!

# ✅ GOOD: Cache expensive operations
@st.cache_data(ttl=300)
def load_data():
    return expensive_database_query()

data = load_data()  # Only runs once per 5 minutes
```

---

## State Management

### Session State Patterns

`st.session_state` is a dictionary-like object that persists data across reruns.

**Initialization Pattern:**
```python
def init_session_state():
    """Initialize session state with defaults."""
    defaults = {
        "messages": [],
        "current_client": None,
        "active_form": None,
        "pending_actions": [],
        "toast_message": None,
        "authenticated": False,
    }
    for key, value in defaults.items():
        if key not in st.session_state:
            st.session_state[key] = value

# Call once at module level
init_session_state()
```

**Naming Conventions:**
- `current_*` - Active state (current_client, current_user, current_return)
- `active_*` - UI state (active_form, active_panel, active_tab)
- `pending_*` - Workflow state (pending_actions, pending_data)
- `last_*` - Temporal state (last_activity, last_login, last_created)

**Callbacks vs Direct Manipulation:**
```python
# ✅ GOOD: Callback updates state before rerun
def on_submit():
    st.session_state.form_submitted = True
    st.session_state.form_data = {"name": st.session_state.name_input}

st.button("Submit", on_click=on_submit)

# ❌ BAD: State update happens after widget already rendered
if st.button("Submit"):
    st.session_state.form_submitted = True  # Too late!
```

**State Cleanup on Logout:**
```python
def logout():
    """Clear authentication and session state."""
    st.session_state.authenticated = False
    st.session_state.current_user = None

    # Clear workflow state
    for key in ["messages", "active_form", "pending_actions", "current_client"]:
        if key in st.session_state:
            del st.session_state[key]
```

### Anti-Patterns

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Global variables | Lost on rerun, not thread-safe | Use `st.session_state` |
| Storing large objects in session state | Memory bloat, slow reruns | Cache data, store IDs only |
| Using cache for state | Cache is for computation, not state | Use `st.session_state` |
| No state initialization | KeyError on first access | Initialize with defaults dict |
| Sprawling state (20+ variables) | Hard to track, debug | Group related state in dicts |

---

## Async Bridge Pattern

**Problem:** Streamlit reruns the entire script on each interaction. If you use `asyncio.run()` for each async call, you get a new event loop each time, causing asyncpg's connection pool to fail with "Event loop is closed" errors.

**Solution:** Maintain a single event loop for the Streamlit session and reuse it across all async operations.

**Implementation (from tax-ai-chat):**
```python
# src/workflows/chat/streamlit_bridge.py
import asyncio
from typing import Any

class StreamlitAsyncBridge:
    """
    Bridge between Streamlit (sync) and async LangGraph flows.
    Maintains a single event loop for the Streamlit session.

    This prevents "Event loop is closed" errors with asyncpg connection pools
    by reusing the same event loop across Streamlit reruns.
    """

    def __init__(self):
        """Initialize the bridge with no event loop (created on first use)."""
        self._loop = None

    def _ensure_loop(self) -> asyncio.AbstractEventLoop:
        """
        Get or create event loop for this session.

        Creates a new loop only if one doesn't exist or if it's been closed.
        Sets the loop as the current event loop for asyncio operations.
        """
        if self._loop is None or self._loop.is_closed():
            self._loop = asyncio.new_event_loop()
            asyncio.set_event_loop(self._loop)
        return self._loop

    def run_flow(self, flow, inputs: dict[str, Any]) -> dict[str, Any]:
        """
        Run async LangGraph flow in managed event loop.

        Args:
            flow: LangGraph flow instance (with ainvoke method)
            inputs: Dictionary of inputs to pass to the flow

        Returns:
            Dictionary result from the flow execution
        """
        loop = self._ensure_loop()
        return loop.run_until_complete(flow.ainvoke(inputs))
```

**Usage:**
```python
# Initialize once in session state
if "async_bridge" not in st.session_state:
    from src.workflows.chat.streamlit_bridge import StreamlitAsyncBridge
    st.session_state.async_bridge = StreamlitAsyncBridge()

# Use for all async flows
result = st.session_state.async_bridge.run_flow(chat_flow, {
    "message": user_input,
    "client_id": client_id,
})
```

**When to Use:**
- Async database operations (asyncpg connection pools)
- LangGraph async workflows
- External async API calls
- Any asyncio-based library requiring persistent event loop

---

## Caching Strategy

### @st.cache_data vs @st.cache_resource

**@st.cache_data** - For data transformations (pure functions):
```python
@st.cache_data(ttl=300)  # Cache for 5 minutes
def load_clients_from_db():
    """Load client list - data snapshot."""
    return pd.DataFrame(client_service.get_all())

# Use for: API responses, database queries, file reads, data transformations
```

**@st.cache_resource** - For connections and stateful objects:
```python
@st.cache_resource
def get_database_connection():
    """Get database connection pool - stateful resource."""
    return psycopg2.pool.SimpleConnectionPool(...)

# Use for: Database connections, ML models, API clients, thread pools
```

**Decision Tree:**
- Returns data snapshot → `@st.cache_data`
- Returns mutable object that changes → `@st.cache_resource`
- Pure function (same input = same output) → `@st.cache_data`
- Creates connection/resource → `@st.cache_resource`

**Cache Invalidation:**
```python
# Clear all caches
st.cache_data.clear()
st.cache_resource.clear()

# Clear specific function's cache
load_clients_from_db.clear()
```

---

## Code Organization

### File Size Limits

**Maximum 500 Lines Per File:**
- Main app file: Layout, navigation, initialization only
- Extract: `components/`, `services/`, `utils/`, `auth.py`

**Bad: God File (1200+ lines):**
```
chat_ui.py (1200 lines)
  - Authentication (200 lines)
  - Custom CSS (550 lines)
  - Forms (150 lines)
  - Business logic (200 lines)
  - Main UI (100 lines)
```

**Good: Organized Structure:**
```
app.py (150 lines)              # Main entry, layout, navigation
auth.py (200 lines)             # Authentication, session management
components/
  context_header.py (50 lines)  # Reusable context header
  client_form.py (100 lines)    # Client creation form
  chat_panel.py (150 lines)     # Chat interface
styles/
  theme.py (100 lines)          # CSS design system
services/
  client_service.py (200 lines) # Business logic
```

### Component Extraction Pattern

**Extract Reusable Components:**
```python
# components/client_form.py
import streamlit as st

def render_client_form(on_submit_callback):
    """Reusable client creation form component."""
    with st.form("client_form"):
        st.subheader("New Client")
        name = st.text_input("Full Name", key="client_name")
        email = st.text_input("Email", key="client_email")
        phone = st.text_input("Phone", key="client_phone")

        col1, col2 = st.columns(2)
        with col1:
            cancel = st.form_submit_button("Cancel", type="secondary")
        with col2:
            submit = st.form_submit_button("Create Client", type="primary")

        if submit and name and email:
            on_submit_callback(name, email, phone)
        elif cancel:
            st.session_state.active_form = None
            st.rerun()

# app.py
from components.client_form import render_client_form

def handle_client_creation(name, email, phone):
    client = client_service.create(name, email, phone)
    st.session_state.current_client = client
    st.session_state.active_form = None
    st.success(f"Client {name} created!")
    st.rerun()

if st.session_state.active_form == "create_client":
    render_client_form(on_submit_callback=handle_client_creation)
```

### Service Initialization Pattern

**Lazy Loading to Avoid Circular Imports:**
```python
# Initialize services once in session state
if "services_initialized" not in st.session_state:
    st.session_state.services_initialized = False
    st.session_state.db_error = None

if not st.session_state.services_initialized:
    try:
        # Lazy import to avoid circular dependencies
        from src.services.sync_client_service import SyncClientService
        from src.workflows.chat.streamlit_bridge import StreamlitAsyncBridge

        SyncClientService.initialize()
        st.session_state.client_service = SyncClientService()
        st.session_state.async_bridge = StreamlitAsyncBridge()
        st.session_state.services_initialized = True
    except RuntimeError as e:
        st.error(f"⚠️ Database connection required: {e}")
        st.info("Please ensure PostgreSQL is running: `docker compose up -d postgres`")
        st.stop()
```

---

## Anti-Patterns to Avoid

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| God file (1000+ lines) | Unmaintainable, untestable | Extract components, services, utils |
| Global variables for state | Lost on rerun, race conditions | Use `st.session_state` |
| No file size limits | Memory exhaustion, slow UI | Maximum 500 lines per file |
| Expensive ops in main script | Runs on every interaction | Use `@st.cache_data` or `@st.cache_resource` |
| Using cache for state | Cache is for computation | Use `st.session_state` for state |
| No authentication | Security risk | Implement database-backed auth + session timeout |
| No file upload validation | Malware, XSS risk | Validate size, type, magic bytes |
| No audit logging | Compliance violation | Append-only audit log for all user actions |
| Hardcoded CSS in Python | Hard to maintain | Extract to styles.py or external .css |
| No component extraction | Code duplication | Extract reusable components |
| Missing accessibility | WCAG violation | Add focus indicators, ARIA, semantic HTML |
| No session timeout | Security risk | Implement absolute (12h) + idle (30m) timeout |
| Direct asyncio.run() calls | Event loop errors with asyncpg | Use StreamlitAsyncBridge pattern |

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| Main Streamlit app | `projects/tax-ai-chat/src/workflows/chat_ui.py` |
| Async bridge pattern | `projects/tax-ai-chat/src/workflows/chat/streamlit_bridge.py` |
| File validation | `projects/tax-ai-chat/src/workflows/chat/file_validators.py` |
| Professional CSS design | `projects/tax-ai-chat/src/workflows/chat_ui.py` (lines 347-900) |
| Authentication patterns | `projects/tax-ai-chat/src/workflows/chat_ui.py` (lines 119-341) |
| Audit logging | `projects/tax-ai-chat/src/workflows/chat_ui.py` (lines 54-96) |

---

## Quick Reference

### Essential Imports

```python
import streamlit as st
from datetime import datetime, timedelta
from pathlib import Path
import logging
import json
```

### Session State Initialization

```python
if "key" not in st.session_state:
    st.session_state.key = default_value
```

### Forms

```python
with st.form("my_form"):
    name = st.text_input("Name")
    submitted = st.form_submit_button("Submit")
    if submitted:
        # Handle submission
        pass
```

### Caching

```python
@st.cache_data(ttl=300)  # Data transformations
def load_data():
    return expensive_query()

@st.cache_resource  # Connections, resources
def get_connection():
    return create_connection()
```

### File Uploads

```python
uploaded_file = st.file_uploader("Upload", type=["pdf", "png"])
if uploaded_file:
    content = uploaded_file.read()
    # Validate, process, upload
```

---

## When to Load Detailed References

For deeper implementations, consult reference files when specific topics arise:

**For security implementations**, consult `references/security-patterns.md` covering:
- Database-backed authentication with bcrypt
- Lockout protection (5 failures, 15-minute lock)
- Session timeout (dual: 12h absolute, 30m idle)
- Audit logging (append-only, IRS Circular 230 / HIPAA compliance)
- File upload validation (size, extension, magic bytes, deduplication)

**For UI design and styling**, consult `references/ui-design-system.md` covering:
- Professional CSS design systems (navy/gold palette for tax/finance)
- WCAG 2.1 AA accessibility (focus indicators, reduced motion, high contrast)
- Dark mode support (CRITICAL: both light and dark mode must be explicitly defined)
- Layout patterns (split layouts, context headers, sticky panels)
- Loading states (spinners, toast notifications, progress bars)

**For testing implementations**, consult `references/testing-strategies.md` covering:
- Unit testing pure functions (file validators, data transformations)
- Component testing with AppTest (login flows, forms, UI interactions)
- Integration testing (database services, async bridge, external APIs)
- Test organization (unit/, component/, integration/, e2e/)
- Mocking patterns and fixtures

**For performance tuning**, consult `references/performance-optimization.md` covering:
- Profiling slow operations (cProfile, manual timers)
- Rerun optimization (cache connections, avoid recomputation)
- Widget keys for state preservation
- Memory management (avoid large objects in session state)
- Network optimization (batch queries, concurrent async)

---

## References

- [Streamlit Documentation](https://docs.streamlit.io/)
- [Streamlit API Reference](https://docs.streamlit.io/library/api-reference)
- [AppTest Documentation](https://docs.streamlit.io/library/api-reference/app-testing)
- [Streamlit Session State Guide](https://docs.streamlit.io/library/advanced-features/session-state)
- [Caching Guide](https://docs.streamlit.io/library/advanced-features/caching)
- [WCAG 2.1 Guidelines](https://www.w3.org/WAI/WCAG21/quickref/)
