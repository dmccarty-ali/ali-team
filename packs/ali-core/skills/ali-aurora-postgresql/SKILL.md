---
name: ali-aurora-postgresql
description: |
  Aurora PostgreSQL and PostgreSQL patterns for Aliunde projects. Use when:

  PLANNING: Designing database schemas, planning indexes, evaluating partitioning
  strategies, considering connection pooling, architecting for read replicas

  IMPLEMENTATION: Writing SQL queries, creating tables/indexes, implementing
  migrations, configuring connection pools, using JSON/JSONB, full-text search

  GUIDANCE: Asking about PostgreSQL best practices, query optimization, index
  types, when to use JSONB, Aurora-specific features, connection management

  REVIEW: Reviewing schema design, checking query performance, validating index
  usage, auditing transaction patterns, checking for N+1 queries
---

# Aurora PostgreSQL

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing database schemas
- Planning indexing strategy
- Evaluating partitioning approaches
- Architecting for high availability

**Implementation:**
- Writing SQL queries (SELECT, INSERT, UPDATE, DELETE)
- Creating tables, indexes, constraints
- Implementing migrations (Alembic, raw SQL)
- Using JSONB or full-text search
- Connection pooling setup

**Guidance/Best Practices:**
- Asking about query optimization
- Asking which index type to use
- Asking about transaction isolation
- Asking about Aurora-specific features

**Review/Validation:**
- Reviewing EXPLAIN ANALYZE output
- Checking for missing indexes
- Validating schema design
- Auditing query performance

---

## Key Principles

- **Normalize first, denormalize later**: Start normalized; denormalize for proven performance needs
- **Index for your queries**: Design indexes based on actual query patterns
- **Measure, don't guess**: Use EXPLAIN ANALYZE before optimizing
- **Connection pooling required**: Never open connections per request
- **Transactions are cheap**: Use them for data integrity
- **Constraints in database**: Enforce at DB level, not just application

---

## Schema Design

### Normalization Levels

```sql
-- 3NF (Third Normal Form) - Good default
CREATE TABLE clients (
    client_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) UNIQUE NOT NULL,
    name            VARCHAR(200) NOT NULL,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE addresses (
    address_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID REFERENCES clients(client_id),
    address_type    VARCHAR(20) NOT NULL,  -- 'billing', 'mailing'
    street          VARCHAR(200),
    city            VARCHAR(100),
    state           CHAR(2),
    zip             VARCHAR(10)
);

-- Denormalized for read performance (when justified)
CREATE TABLE client_summary (
    client_id       UUID PRIMARY KEY REFERENCES clients(client_id),
    name            VARCHAR(200),
    email           VARCHAR(255),
    primary_address JSONB,          -- Embedded for fast reads
    total_returns   INTEGER,        -- Pre-aggregated
    last_return_date DATE
);
```

### Standard Columns

```sql
-- Every table should have
CREATE TABLE example (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- business columns here
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW(),
    created_by      UUID REFERENCES users(user_id),
    updated_by      UUID REFERENCES users(user_id)
);

-- Auto-update updated_at
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER set_updated_at
    BEFORE UPDATE ON example
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at();
```

---

## Index Patterns

### B-tree (Default)

```sql
-- Equality and range queries
CREATE INDEX idx_clients_email ON clients(email);
CREATE INDEX idx_returns_date ON returns(filing_date);

-- Multi-column (order matters!)
-- Supports: WHERE status = 'x' AND created_at > 'y'
-- Also supports: WHERE status = 'x' (leftmost prefix)
-- Does NOT support: WHERE created_at > 'y' alone
CREATE INDEX idx_returns_status_date ON returns(status, created_at);
```

### Partial Index

```sql
-- Index only active records (smaller, faster)
CREATE INDEX idx_active_clients ON clients(email)
    WHERE status = 'active';

-- Index only recent data
CREATE INDEX idx_recent_returns ON returns(client_id, filing_date)
    WHERE filing_date > '2023-01-01';
```

### Covering Index (INCLUDE)

```sql
-- Include columns to avoid table lookup
CREATE INDEX idx_clients_email_covering ON clients(email)
    INCLUDE (name, phone);

-- Query can be satisfied entirely from index:
-- SELECT name, phone FROM clients WHERE email = 'x@y.com'
```

### GIN Index (JSONB, Arrays, Full-text)

```sql
-- JSONB containment queries
CREATE INDEX idx_metadata_gin ON documents USING GIN (metadata);

-- Supports: WHERE metadata @> '{"type": "tax_return"}'
-- Supports: WHERE metadata ? 'ssn'

-- Full-text search
CREATE INDEX idx_content_search ON documents
    USING GIN (to_tsvector('english', content));
```

### Expression Index

```sql
-- Index on computed value
CREATE INDEX idx_clients_email_lower ON clients(LOWER(email));

-- Query must match expression exactly:
-- SELECT * FROM clients WHERE LOWER(email) = 'user@example.com'
```

---

## Query Optimization

### EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT c.name, COUNT(r.return_id)
FROM clients c
JOIN returns r ON c.client_id = r.client_id
WHERE r.filing_date > '2024-01-01'
GROUP BY c.name;

-- Key things to look for:
-- - Seq Scan on large tables (missing index?)
-- - Nested Loop with high row counts (wrong join strategy?)
-- - actual time vs planned rows mismatch (stale statistics?)
-- - Buffers: shared hit vs read (cache effectiveness)
```

### Common Optimizations

```sql
-- BAD: Function on indexed column prevents index use
SELECT * FROM clients WHERE LOWER(email) = 'user@example.com';
-- FIX: Create expression index, or store normalized

-- BAD: SELECT * when you need few columns
SELECT * FROM large_table WHERE id = 1;
-- FIX: Select only needed columns
SELECT name, email FROM large_table WHERE id = 1;

-- BAD: OR can prevent index use
SELECT * FROM clients WHERE email = 'a' OR phone = 'b';
-- FIX: Use UNION if separate indexes exist
SELECT * FROM clients WHERE email = 'a'
UNION
SELECT * FROM clients WHERE phone = 'b';

-- BAD: LIKE with leading wildcard
SELECT * FROM clients WHERE name LIKE '%smith';
-- FIX: Use full-text search or trigram index
```

### Statistics

```sql
-- Update statistics after large data changes
ANALYZE clients;
ANALYZE returns;

-- Check table statistics
SELECT relname, n_live_tup, n_dead_tup, last_analyze
FROM pg_stat_user_tables
WHERE schemaname = 'public';
```

---

## Connection Pooling

### psycopg2 Pool

```python
from psycopg2 import pool
from contextlib import contextmanager

# Initialize pool once at startup
connection_pool = pool.ThreadedConnectionPool(
    minconn=5,
    maxconn=20,
    host=os.environ['DB_HOST'],
    port=5432,
    database=os.environ['DB_NAME'],
    user=os.environ['DB_USER'],
    password=os.environ['DB_PASSWORD'],
)

@contextmanager
def get_connection():
    conn = connection_pool.getconn()
    try:
        yield conn
    finally:
        connection_pool.putconn(conn)

# Usage
with get_connection() as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT * FROM clients WHERE id = %s", (client_id,))
        result = cur.fetchone()
```

### SQLAlchemy with Pool

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

engine = create_engine(
    f"postgresql://{user}:{password}@{host}:{port}/{database}",
    pool_size=10,           # Connections to keep open
    max_overflow=20,        # Extra connections when pool exhausted
    pool_recycle=3600,      # Recycle connections after 1 hour
    pool_pre_ping=True,     # Verify connection is alive
)

Session = sessionmaker(bind=engine)
```

---

## Transaction Patterns

### Isolation Levels

| Level | Dirty Read | Non-repeatable Read | Phantom Read | Use Case |
|-------|------------|---------------------|--------------|----------|
| READ COMMITTED | No | Yes | Yes | Default, most cases |
| REPEATABLE READ | No | No | No* | Reports, consistent reads |
| SERIALIZABLE | No | No | No | Financial, strict consistency |

*PostgreSQL's REPEATABLE READ also prevents phantoms

```python
from sqlalchemy import text

with Session() as session:
    # Set isolation level for this transaction
    session.execute(text("SET TRANSACTION ISOLATION LEVEL REPEATABLE READ"))

    # All reads in this transaction see same snapshot
    balance = session.execute(text("SELECT balance FROM accounts WHERE id = 1"))
    # ... other operations see consistent state
    session.commit()
```

### Advisory Locks

```sql
-- Application-level locking (doesn't lock rows)
-- Useful for: batch processing, singleton tasks

-- Try to acquire lock (non-blocking)
SELECT pg_try_advisory_lock(12345);  -- Returns true/false

-- Acquire lock (blocking)
SELECT pg_advisory_lock(12345);

-- Release lock
SELECT pg_advisory_unlock(12345);
```

```python
def process_batch_with_lock(batch_id: int):
    """Ensure only one process handles a batch."""
    with get_connection() as conn:
        with conn.cursor() as cur:
            # Try to get lock
            cur.execute("SELECT pg_try_advisory_lock(%s)", (batch_id,))
            acquired = cur.fetchone()[0]

            if not acquired:
                return  # Another process has it

            try:
                process_batch(batch_id)
            finally:
                cur.execute("SELECT pg_advisory_unlock(%s)", (batch_id,))
```

---

## Partitioning

### Range Partitioning (Time-based)

```sql
CREATE TABLE events (
    event_id        UUID,
    event_time      TIMESTAMPTZ NOT NULL,
    event_type      VARCHAR(50),
    payload         JSONB
) PARTITION BY RANGE (event_time);

-- Create partitions
CREATE TABLE events_2024_01 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE events_2024_02 PARTITION OF events
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Automate partition creation with pg_partman or cron
```

### List Partitioning

```sql
CREATE TABLE returns (
    return_id       UUID,
    return_type     VARCHAR(10) NOT NULL,
    client_id       UUID,
    data            JSONB
) PARTITION BY LIST (return_type);

CREATE TABLE returns_1040 PARTITION OF returns
    FOR VALUES IN ('1040', '1040-SR');

CREATE TABLE returns_business PARTITION OF returns
    FOR VALUES IN ('1120', '1120S', '1065');
```

---

## Aurora-Specific Features

### Read Replicas

```python
import os

# Use environment variables to switch endpoints
WRITER_ENDPOINT = os.environ['DB_WRITER_ENDPOINT']  # For writes
READER_ENDPOINT = os.environ['DB_READER_ENDPOINT']  # For reads

writer_engine = create_engine(f"postgresql://...@{WRITER_ENDPOINT}/db")
reader_engine = create_engine(f"postgresql://...@{READER_ENDPOINT}/db")

# Route queries appropriately
def get_client(client_id):
    """Read operation - use reader."""
    with reader_engine.connect() as conn:
        return conn.execute(...)

def update_client(client_id, data):
    """Write operation - use writer."""
    with writer_engine.connect() as conn:
        conn.execute(...)
```

### Failover

```python
# Aurora handles failover automatically
# Application should:
# 1. Use connection pooling with pre_ping
# 2. Retry on connection errors
# 3. Use cluster endpoint (not instance endpoint)

from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=1, max=10),
    reraise=True
)
def execute_with_retry(query, params):
    with get_connection() as conn:
        with conn.cursor() as cur:
            cur.execute(query, params)
            return cur.fetchall()
```

---

## JSONB Patterns

### Storage

```sql
CREATE TABLE documents (
    doc_id      UUID PRIMARY KEY,
    doc_type    VARCHAR(50),
    metadata    JSONB,              -- Flexible attributes
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

INSERT INTO documents (doc_id, doc_type, metadata)
VALUES (
    gen_random_uuid(),
    'tax_return',
    '{"year": 2024, "form": "1040", "client": {"name": "John", "ssn_last4": "1234"}}'
);
```

### Querying

```sql
-- Access nested value
SELECT metadata->>'year' AS year FROM documents;
SELECT metadata->'client'->>'name' AS client_name FROM documents;

-- Containment (uses GIN index)
SELECT * FROM documents
WHERE metadata @> '{"form": "1040"}';

-- Key existence
SELECT * FROM documents
WHERE metadata ? 'ssn';

-- Array element
SELECT * FROM documents
WHERE metadata->'tags' ? 'urgent';
```

### Updating

```sql
-- Set a key
UPDATE documents
SET metadata = jsonb_set(metadata, '{status}', '"approved"')
WHERE doc_id = '...';

-- Remove a key
UPDATE documents
SET metadata = metadata - 'temp_field'
WHERE doc_id = '...';

-- Merge objects
UPDATE documents
SET metadata = metadata || '{"reviewed": true, "reviewer": "admin"}'
WHERE doc_id = '...';
```

---

## Full-Text Search

```sql
-- Add search vector column
ALTER TABLE documents ADD COLUMN search_vector tsvector;

-- Populate and index
UPDATE documents SET search_vector =
    to_tsvector('english', COALESCE(title, '') || ' ' || COALESCE(content, ''));

CREATE INDEX idx_documents_search ON documents USING GIN(search_vector);

-- Search
SELECT doc_id, title,
       ts_rank(search_vector, query) AS rank
FROM documents,
     to_tsquery('english', 'tax & return') AS query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 10;

-- Keep search vector updated
CREATE TRIGGER documents_search_update
    BEFORE INSERT OR UPDATE ON documents
    FOR EACH ROW EXECUTE FUNCTION
    tsvector_update_trigger(search_vector, 'pg_catalog.english', title, content);
```

---

## Migrations (Alembic)

```python
# alembic/versions/001_create_clients.py
from alembic import op
import sqlalchemy as sa

def upgrade():
    op.create_table(
        'clients',
        sa.Column('client_id', sa.UUID(), primary_key=True),
        sa.Column('email', sa.String(255), nullable=False, unique=True),
        sa.Column('name', sa.String(200), nullable=False),
        sa.Column('created_at', sa.DateTime(timezone=True), server_default=sa.func.now()),
    )
    op.create_index('idx_clients_email', 'clients', ['email'])

def downgrade():
    op.drop_table('clients')
```

```bash
# Generate migration from model changes
alembic revision --autogenerate -m "Add clients table"

# Apply migrations
alembic upgrade head

# Rollback one step
alembic downgrade -1
```

---

## Materialized Views

Cache expensive query results:

```sql
-- Create materialized view for dashboard metrics
CREATE MATERIALIZED VIEW client_summary_mv AS
SELECT
    c.client_id,
    c.name,
    COUNT(d.doc_id) AS document_count,
    MAX(d.uploaded_at) AS last_upload,
    SUM(CASE WHEN d.status = 'processed' THEN 1 ELSE 0 END) AS processed_count
FROM clients c
LEFT JOIN documents d ON c.client_id = d.client_id
GROUP BY c.client_id, c.name;

-- Create unique index for concurrent refresh
CREATE UNIQUE INDEX idx_client_summary_mv_id ON client_summary_mv(client_id);

-- Refresh data (blocks reads)
REFRESH MATERIALIZED VIEW client_summary_mv;

-- Refresh without blocking reads (requires unique index)
REFRESH MATERIALIZED VIEW CONCURRENTLY client_summary_mv;

-- Query like a regular table
SELECT * FROM client_summary_mv WHERE document_count > 10;

-- Schedule refresh (via cron/scheduler)
-- Typically refresh during low-traffic periods
```

---

## Row-Level Security (RLS)

Enforce data access at the database level:

```sql
-- Enable RLS on table
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

-- Policy: Users see only their own documents
CREATE POLICY user_documents ON documents
    FOR ALL
    USING (owner_id = current_setting('app.current_user_id')::uuid);

-- Policy: Admins see everything
CREATE POLICY admin_all ON documents
    FOR ALL
    USING (current_setting('app.user_role') = 'admin');

-- Set context in application
SET app.current_user_id = 'user-uuid-here';
SET app.user_role = 'preparer';

-- Multi-tenant isolation
CREATE POLICY tenant_isolation ON clients
    FOR ALL
    USING (tenant_id = current_setting('app.tenant_id')::uuid);
```

```python
# Set RLS context in SQLAlchemy
async def set_rls_context(session: AsyncSession, user: User):
    await session.execute(
        text("SET app.current_user_id = :user_id"),
        {"user_id": str(user.id)}
    )
    await session.execute(
        text("SET app.user_role = :role"),
        {"role": user.role}
    )
```

---

## Recursive CTEs

Query hierarchical data:

```sql
-- Find all ancestors (org chart: employee → managers)
WITH RECURSIVE org_hierarchy AS (
    -- Base case: start from specific employee
    SELECT employee_id, name, manager_id, 1 AS level
    FROM employees
    WHERE employee_id = 'emp-123'

    UNION ALL

    -- Recursive: find manager of current row
    SELECT e.employee_id, e.name, e.manager_id, h.level + 1
    FROM employees e
    JOIN org_hierarchy h ON e.employee_id = h.manager_id
)
SELECT * FROM org_hierarchy;

-- Find all descendants (manager → all reports)
WITH RECURSIVE reports AS (
    SELECT employee_id, name, manager_id, 1 AS depth
    FROM employees
    WHERE manager_id = 'manager-123'

    UNION ALL

    SELECT e.employee_id, e.name, e.manager_id, r.depth + 1
    FROM employees e
    JOIN reports r ON e.manager_id = r.employee_id
    WHERE r.depth < 10  -- Prevent infinite loops
)
SELECT * FROM reports ORDER BY depth;

-- Category tree (breadcrumb path)
WITH RECURSIVE category_path AS (
    SELECT category_id, name, parent_id, name AS path
    FROM categories
    WHERE category_id = 'cat-123'

    UNION ALL

    SELECT c.category_id, c.name, c.parent_id,
           c.name || ' > ' || cp.path
    FROM categories c
    JOIN category_path cp ON c.category_id = cp.parent_id
)
SELECT path FROM category_path WHERE parent_id IS NULL;
```

---

## Database Maintenance

### VACUUM and ANALYZE

```sql
-- Manual vacuum (reclaim space from dead tuples)
VACUUM documents;

-- Vacuum with analyze (also update statistics)
VACUUM ANALYZE documents;

-- Full vacuum (reclaims more space, but locks table)
VACUUM FULL documents;  -- Use during maintenance windows only

-- Analyze only (update query planner statistics)
ANALYZE documents;

-- Check table bloat
SELECT
    schemaname || '.' || relname AS table_name,
    pg_size_pretty(pg_relation_size(schemaname || '.' || relname)) AS size,
    n_dead_tup AS dead_tuples,
    n_live_tup AS live_tuples,
    ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;
```

### Index Maintenance

```sql
-- Check index usage
SELECT
    schemaname || '.' || relname AS table_name,
    indexrelname AS index_name,
    idx_scan AS index_scans,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;  -- Low usage = candidates for removal

-- Find missing indexes (sequential scans on large tables)
SELECT
    schemaname || '.' || relname AS table_name,
    seq_scan AS sequential_scans,
    seq_tup_read AS rows_scanned,
    idx_scan AS index_scans
FROM pg_stat_user_tables
WHERE seq_scan > 100 AND seq_tup_read > 10000
ORDER BY seq_tup_read DESC;

-- Rebuild bloated indexes
REINDEX INDEX CONCURRENTLY idx_documents_client_id;
```

### Monitoring Queries

```sql
-- Active queries
SELECT
    pid,
    now() - pg_stat_activity.query_start AS duration,
    query,
    state
FROM pg_stat_activity
WHERE state != 'idle'
  AND query NOT LIKE '%pg_stat_activity%'
ORDER BY duration DESC;

-- Lock contention
SELECT
    blocked.pid AS blocked_pid,
    blocked.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_locks blocked_locks ON blocked.pid = blocked_locks.pid
JOIN pg_locks blocking_locks ON blocked_locks.locktype = blocking_locks.locktype
    AND blocked_locks.relation = blocking_locks.relation
    AND blocked_locks.pid != blocking_locks.pid
JOIN pg_stat_activity blocking ON blocking_locks.pid = blocking.pid
WHERE NOT blocked_locks.granted;
```

---

## Anti-Patterns

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| No connection pooling | Connection overhead per request | Use pool (psycopg2, SQLAlchemy) |
| SELECT * everywhere | Transfers unnecessary data | Select only needed columns |
| Missing indexes | Slow queries, full table scans | Index based on query patterns |
| N+1 queries | 100 items = 101 queries | Use JOINs or eager loading |
| Large transactions | Lock contention, memory | Keep transactions short |
| UUID as primary key only | Poor for range queries | Add created_at index |
| Storing large blobs | Database bloat | Store in S3, reference by URL |

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| SQLAlchemy models | `ali-ai-acctg/app/models/` |
| Database config | `ali-ai-acctg/app/database.py` |
| Alembic migrations | `ali-ai-acctg/alembic/` |
| Connection settings | `ali-ai-acctg/.env` |

---

## References

- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [Aurora PostgreSQL User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.AuroraPostgreSQL.html)
- [Use The Index, Luke](https://use-the-index-luke.com/)
- [PostgreSQL EXPLAIN Visualizer](https://explain.depesz.com/)
