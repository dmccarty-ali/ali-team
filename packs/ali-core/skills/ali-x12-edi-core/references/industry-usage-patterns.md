# ali-x12-edi-core - Industry Usage Patterns Reference

## Industry Usage Patterns

### File Size Guidelines

HIPAA does not mandate file size or claims-per-file limits. Limits are determined by trading partner agreements and clearinghouse specifications.

**Typical Industry Ranges:**

| Scenario | Claims/File | Segments/File | Est. Memory | Notes |
|----------|-------------|---------------|-------------|-------|
| Small Practice | 10-50 | 500-2,500 | 1-5 MB | Direct submission to payer |
| Hospital Daily | 500-2,000 | 25K-100K | 50-200 MB | Daily batch submission |
| Clearinghouse Batch | 5,000-50,000 | 250K-2.5M | 500 MB - 5 GB | Aggregated from multiple providers |
| Month-End Dump | 100,000+ | 5M+ | 10+ GB | Large health systems, rare |

**Clearinghouse Recommendations (general guidance):**
- Change Healthcare: Up to 50MB or 10,000+ claims
- Availity: Varies by payer connection, typically 5,000-10,000 claims
- Most clearinghouses: 1,000-5,000 claims per file recommended for easier troubleshooting

### Memory Management for Parsers

**Design Decision:** Use bounded row buffering with 10,000 row limit per flush.

**Rationale:**
- Balances batching efficiency with bounded memory usage
- Caps memory at ~50-100MB regardless of input file size
- Handles clearinghouse batch files gracefully (auto-flushes mid-file)
- Works for 95%+ of real-world scenarios without configuration

**Implementation Pattern:**
```
Parse segments → accumulate in buffer per table
If buffer > 10K rows OR end of file:
    insertRows() per table and clear buffer
```

**Throughput Impact:**
- Single-row inserts: ~20 files/sec (50 parsers)
- Batched inserts (10K buffer): Target 60-80 files/sec (3-4x improvement)

### Batching vs Streaming Tradeoffs

| Approach | Throughput | Latency | Memory | Use Case |
|----------|------------|---------|--------|----------|
| Single-row insert | Lower | Low (~1 sec) | Minimal | Real-time streaming |
| Bounded batch (10K) | Higher | Medium (batch size dependent) | Bounded | Bulk processing |
| Full-file batch | Highest | High (file size dependent) | Unbounded | Small files only |
| CSV + COPY INTO | Highest | High | Disk-based | Largest scale |

**Recommendation:** Bounded batch (10K rows) for production bulk processing. Single-row for real-time use cases where sub-second latency matters.

### Scaling Projections

Based on bounded batching with 50 parsers:

| Files | Est. Time | Notes |
|-------|-----------|-------|
| 5,000 | ~1-2 min | Small daily batch |
| 10,000 | ~2-3 min | Medium daily batch |
| 100,000 | ~20-30 min | Large batch |
| 1,000,000 | ~3-5 hours | Enterprise scale |

**Scaling factors:**
- Network I/O to Snowflake (primary bottleneck on retail connections)
- Parser count (linear scaling up to network saturation)
- Snowflake warehouse size (affects ingestion throughput)
