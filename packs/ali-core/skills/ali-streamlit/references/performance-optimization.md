# ali-streamlit - Performance Optimization Reference

Performance tuning and optimization techniques for Streamlit applications. Load when debugging performance or optimizing reruns.

---

## Profiling Slow Operations

### Identify Performance Bottlenecks

**Cache decorator reveals slow operations:**

```python
@st.cache_data(ttl=300, show_spinner=True)
def expensive_operation():
    """Cache decorator shows spinner - reveals slow operations."""
    import time
    time.sleep(5)  # This will be visible to user
    return compute_results()
```

**Manual profiling:**

```python
import time
from contextlib import contextmanager

@contextmanager
def timer(label: str):
    """Profile code block execution time."""
    start = time.perf_counter()
    try:
        yield
    finally:
        elapsed = time.perf_counter() - start
        st.write(f"⏱️ {label}: {elapsed:.3f}s")

# Usage
with timer("Loading clients"):
    clients = load_all_clients()  # Reveals if this is slow

with timer("Rendering table"):
    st.dataframe(large_dataframe)  # Check rendering performance
```

**Profiling with cProfile:**

```python
import cProfile
import pstats
from io import StringIO

def profile_function(func):
    """Decorator to profile function calls."""
    def wrapper(*args, **kwargs):
        profiler = cProfile.Profile()
        profiler.enable()
        result = func(*args, **kwargs)
        profiler.disable()

        # Print stats
        stream = StringIO()
        stats = pstats.Stats(profiler, stream=stream)
        stats.sort_stats('cumulative')
        stats.print_stats(20)  # Top 20 slowest
        st.code(stream.getvalue())

        return result
    return wrapper

@profile_function
def slow_computation():
    # Complex logic
    pass
```

---

## Rerun Optimization

### Move Expensive Operations Outside Main Script

**Problem: Database connection created on every rerun**

```python
# ❌ BAD: Connection created on every rerun
conn = psycopg2.connect(...)
data = load_data(conn)
```

**Solution: Cache connection as resource**

```python
# ✅ GOOD: Connection cached and reused
@st.cache_resource
def get_connection():
    return psycopg2.connect(...)

conn = get_connection()
data = load_data(conn)
```

### Avoid Recomputing Unchanged Data

**Problem: Large dataset recomputed on every interaction**

```python
# ❌ BAD: Filtering happens on every rerun
all_clients = load_all_clients()  # 10,000 records
filtered = [c for c in all_clients if c["status"] == "active"]
st.dataframe(filtered)
```

**Solution: Cache data transformations**

```python
# ✅ GOOD: Filtering cached with TTL
@st.cache_data(ttl=300)
def get_active_clients():
    all_clients = load_all_clients()
    return [c for c in all_clients if c["status"] == "active"]

filtered = get_active_clients()
st.dataframe(filtered)
```

### Lazy Loading

**Load data only when needed:**

```python
# ❌ BAD: All tabs' data loaded upfront
tab1_data = load_dashboard_data()
tab2_data = load_reports_data()
tab3_data = load_settings_data()

tab1, tab2, tab3 = st.tabs(["Dashboard", "Reports", "Settings"])
with tab1:
    st.write(tab1_data)
with tab2:
    st.write(tab2_data)
with tab3:
    st.write(tab3_data)
```

```python
# ✅ GOOD: Only load active tab's data
tab1, tab2, tab3 = st.tabs(["Dashboard", "Reports", "Settings"])

with tab1:
    if "dashboard_data" not in st.session_state:
        st.session_state.dashboard_data = load_dashboard_data()
    st.write(st.session_state.dashboard_data)

with tab2:
    if "reports_data" not in st.session_state:
        st.session_state.reports_data = load_reports_data()
    st.write(st.session_state.reports_data)

with tab3:
    if "settings_data" not in st.session_state:
        st.session_state.settings_data = load_settings_data()
    st.write(st.session_state.settings_data)
```

---

## Widget Keys for State Preservation

### Prevent Unnecessary Reruns

**Problem: Widget state lost on rerun**

```python
# ❌ BAD: Widget loses value on rerun
name = st.text_input("Name")  # Loses value on rerun
```

**Solution: Use keys to preserve state**

```python
# ✅ GOOD: Widget state persists
name = st.text_input("Name", key="client_name")  # Preserves value
```

### Form Batching

**Problem: Every input triggers rerun**

```python
# ❌ BAD: Each input causes rerun (3 reruns total)
name = st.text_input("Name")
email = st.text_input("Email")
phone = st.text_input("Phone")
if st.button("Submit"):
    create_client(name, email, phone)
```

**Solution: Use forms to batch inputs**

```python
# ✅ GOOD: Only submit button triggers rerun (1 rerun)
with st.form("client_form"):
    name = st.text_input("Name")
    email = st.text_input("Email")
    phone = st.text_input("Phone")
    submitted = st.form_submit_button("Submit")
    if submitted:
        create_client(name, email, phone)
```

---

## Caching Best Practices

### Cache Granularity

**Too coarse: Large monolithic cache**

```python
# ❌ BAD: Entire dashboard cached together
@st.cache_data(ttl=300)
def get_dashboard_data():
    return {
        "clients": load_clients(),       # Changes frequently
        "metrics": compute_metrics(),    # Changes frequently
        "settings": load_settings(),     # Changes rarely
    }
```

**Better: Separate caches by update frequency**

```python
# ✅ GOOD: Different TTLs for different data
@st.cache_data(ttl=60)  # 1 minute
def load_clients():
    return db.query("SELECT * FROM clients")

@st.cache_data(ttl=300)  # 5 minutes
def compute_metrics():
    return expensive_calculation()

@st.cache_data(ttl=3600)  # 1 hour
def load_settings():
    return db.query("SELECT * FROM settings")
```

### Cache Invalidation

**Manual cache clearing:**

```python
# Clear all caches
if st.button("Refresh Data"):
    st.cache_data.clear()
    st.rerun()

# Clear specific function's cache
if st.button("Refresh Clients"):
    load_clients.clear()
    st.rerun()
```

**Event-driven invalidation:**

```python
def create_client(name: str, email: str):
    # Create client in database
    db.execute("INSERT INTO clients ...")

    # Invalidate client cache
    load_clients.clear()
    get_active_clients.clear()

    st.success("Client created!")
    st.rerun()
```

---

## Memory Management

### Avoid Storing Large Objects in Session State

**Problem: Memory bloat from large objects**

```python
# ❌ BAD: 1GB DataFrame in session state
st.session_state.large_dataframe = load_huge_dataset()  # 1GB
```

**Solution: Cache data, store IDs only**

```python
# ✅ GOOD: Store ID, cache full data
@st.cache_data
def load_dataset(dataset_id: str):
    return load_huge_dataset(dataset_id)

# Only store ID in session state
st.session_state.current_dataset_id = "dataset_123"

# Load from cache when needed
df = load_dataset(st.session_state.current_dataset_id)
```

### Pagination for Large Datasets

**Problem: Rendering 10,000 rows at once**

```python
# ❌ BAD: Display all rows (slow rendering)
st.dataframe(all_clients)  # 10,000 rows
```

**Solution: Paginate results**

```python
# ✅ GOOD: Paginate data
PAGE_SIZE = 100

if "page" not in st.session_state:
    st.session_state.page = 0

start = st.session_state.page * PAGE_SIZE
end = start + PAGE_SIZE

st.dataframe(all_clients[start:end])

col1, col2, col3 = st.columns(3)
with col1:
    if st.button("Previous") and st.session_state.page > 0:
        st.session_state.page -= 1
        st.rerun()
with col3:
    if st.button("Next") and end < len(all_clients):
        st.session_state.page += 1
        st.rerun()
```

---

## Async Optimization

### Concurrent Operations

**Problem: Sequential async calls (slow)**

```python
# ❌ BAD: Sequential async calls (3 seconds total)
result1 = await service1.fetch()  # 1 second
result2 = await service2.fetch()  # 1 second
result3 = await service3.fetch()  # 1 second
```

**Solution: Concurrent execution with asyncio.gather**

```python
# ✅ GOOD: Concurrent async calls (1 second total)
import asyncio

results = await asyncio.gather(
    service1.fetch(),
    service2.fetch(),
    service3.fetch(),
)
result1, result2, result3 = results
```

**In Streamlit with async bridge:**

```python
async def fetch_all_data():
    return await asyncio.gather(
        load_clients(),
        load_metrics(),
        load_settings(),
    )

# Run concurrently via bridge
results = st.session_state.async_bridge.run_flow(fetch_all_data, {})
clients, metrics, settings = results
```

---

## Network Optimization

### Batch Database Queries

**Problem: N+1 query problem**

```python
# ❌ BAD: N+1 queries (101 queries total)
clients = db.query("SELECT * FROM clients")  # 1 query
for client in clients:
    returns = db.query(
        "SELECT * FROM tax_returns WHERE client_id = %s",
        (client["id"],)
    )  # 100 queries
    client["returns"] = returns
```

**Solution: Single query with JOIN**

```python
# ✅ GOOD: Single query (1 query total)
results = db.query("""
    SELECT c.*, r.*
    FROM clients c
    LEFT JOIN tax_returns r ON r.client_id = c.id
""")

# Group by client
clients = {}
for row in results:
    client_id = row["id"]
    if client_id not in clients:
        clients[client_id] = {
            "id": row["id"],
            "name": row["name"],
            "returns": []
        }
    if row["return_id"]:
        clients[client_id]["returns"].append({
            "id": row["return_id"],
            "year": row["year"]
        })
```

---

## UI Rendering Optimization

### Virtual Scrolling for Large Lists

**Use st.dataframe for large datasets (native virtualization):**

```python
# ✅ GOOD: Dataframe virtualizes rendering
st.dataframe(
    large_dataframe,
    use_container_width=True,
    height=600  # Virtual scroll within 600px
)
```

### Avoid Excessive st.write Calls

**Problem: Thousands of individual st.write calls**

```python
# ❌ BAD: 1000 separate write calls
for item in items:
    st.write(f"- {item}")
```

**Solution: Build string, single write**

```python
# ✅ GOOD: Single write with markdown
markdown = "\n".join([f"- {item}" for item in items])
st.markdown(markdown)
```

---

## Performance Checklist

Use this checklist when optimizing performance:

**Profiling:**
- [ ] Identified slow operations (cProfile, timer)
- [ ] Measured baseline performance
- [ ] Set performance targets

**Caching:**
- [ ] Expensive operations cached (@st.cache_data)
- [ ] Connections/resources cached (@st.cache_resource)
- [ ] Appropriate TTLs set (based on data change frequency)
- [ ] Cache invalidation on data changes

**Reruns:**
- [ ] Forms used to batch inputs
- [ ] Widget keys preserve state
- [ ] Lazy loading for tabs/expanders
- [ ] Expensive ops moved outside main script

**Memory:**
- [ ] Large objects not in session state
- [ ] Pagination for large datasets
- [ ] IDs stored, full data cached

**Database:**
- [ ] N+1 queries eliminated (use JOINs)
- [ ] Batch operations used
- [ ] Connection pooling enabled
- [ ] Query result caching

**Async:**
- [ ] Concurrent operations (asyncio.gather)
- [ ] Event loop reused (async bridge)
- [ ] Appropriate timeouts set

**UI:**
- [ ] Virtual scrolling for large lists (st.dataframe)
- [ ] Minimize st.write calls
- [ ] Avoid deep nesting

---

## Performance Targets

**Recommended targets for Streamlit apps:**

| Metric | Target | Notes |
|--------|--------|-------|
| Initial load | <3 seconds | Time to interactive |
| Rerun (cached) | <500ms | Common interactions |
| Rerun (uncached) | <2 seconds | Data fetch + render |
| Database query | <100ms | Single query |
| API call | <1 second | External service |
| Large dataset render | <1 second | 10k rows |

---

## Debugging Performance

### Enable Debug Mode

```python
# Show execution time for each component
import streamlit as st

st.set_option('client.showErrorDetails', True)
```

### Profile Streamlit App

```bash
# Profile with py-spy
py-spy top --pid $(pgrep -f streamlit)

# Generate flamegraph
py-spy record -o profile.svg -- streamlit run app.py
```

---

## References

- [Streamlit Performance Guide](https://docs.streamlit.io/library/advanced-features/performance)
- [Caching Documentation](https://docs.streamlit.io/library/advanced-features/caching)
- [Python Profiling](https://docs.python.org/3/library/profile.html)
- [asyncio Documentation](https://docs.python.org/3/library/asyncio.html)
