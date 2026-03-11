# ali-x12-edi-core - Java Parser Implementation Reference

## Java Parser Architecture

### BaseParser Template Method Pattern

Transaction-specific parsers extend a common base class using the template method pattern:

```java
public abstract class BaseParser {
    protected final ParserConfig config;
    protected final RecordWriter writer;
    protected ParseContext context;

    // Template method - called for each segment
    protected void processSegment(X12Segment segment) {
        String sectionCode = determineSectionCode(segment);
        if (sectionCode == null) return;  // Skip unmapped segments

        handleTransactionSpecificContext(segment, sectionCode);
        String keyPath = handleContext(segment, sectionCode);

        Map<String, Object> record = segmentHandler.handleSegment(segment, context, sectionCode);
        if (record != null) {
            record.put("key_hash", keyPath);
            String tableName = segmentHandler.getOutputTable(sectionCode);
            writer.write(tableName, record);
        }
    }

    // Abstract methods for transaction-specific behavior
    protected abstract Set<String> getParentSegments();
    protected abstract void handleTransactionSpecificContext(X12Segment segment, String sectionCode);
    protected abstract String getTransactionType();
}
```

**Transaction-Specific Implementations:**

```java
// 837i uses HL segments for hierarchy
public class Parser837i extends BaseParser {
    @Override
    protected Set<String> getParentSegments() {
        return Set.of("ISA", "GS", "ST", "HL", "CLM", "LX", "SBR");
    }

    @Override
    protected String determineHlSectionCode(X12Segment segment) {
        String hlLevel = segment.getElement(3);  // HL03 = hierarchy level code
        return switch (hlLevel) {
            case "20" -> "hl_20_billing";      // Billing Provider
            case "22" -> "hl_22_subscriber";   // Subscriber
            case "23" -> "hl_23_patient";      // Patient
            default -> null;
        };
    }
}

// 835 has no HL segments, uses CLP for claims
public class Parser835 extends BaseParser {
    @Override
    protected Set<String> getParentSegments() {
        return Set.of("ISA", "GS", "ST", "LX", "CLP");  // No HL
    }

    @Override
    protected String determineHlSectionCode(X12Segment segment) {
        return null;  // 835 doesn't use HL segments
    }
}
```

---

## X12Segment Class

Immutable representation of a parsed X12 segment:

```java
public class X12Segment {
    private final String segmentId;      // e.g., "ISA", "NM1", "CLP"
    private final List<String> elements; // All elements including segment ID
    private final String rawSegment;     // Original segment string
    private final int lineNumber;        // 1-based line number in file

    /**
     * Get element by 1-based position (X12 convention).
     * Position 0 = segment ID
     * Position 1 = first data element (e.g., CLM01, CLP01)
     */
    public String getElement(int position) {
        if (position < 0 || position >= elements.size()) {
            return "";  // Return empty string for out-of-range
        }
        return elements.get(position) != null ? elements.get(position) : "";
    }

    /**
     * Get composite element component.
     * For CLM05 (C023) or SVC01 (C003), access sub-components.
     */
    public String getCompositeElement(int elementPosition, int componentPosition, char componentSeparator) {
        String element = getElement(elementPosition);
        if (element.isEmpty()) return "";

        String[] components = element.split(String.valueOf(componentSeparator), -1);
        int index = componentPosition - 1;  // Convert to 0-based
        return (index >= 0 && index < components.length) ? components[index] : "";
    }

    // Helper methods
    public boolean isSegment(String type) { return segmentId.equals(type); }
    public boolean isAnySegment(String... types) { /* ... */ }
}
```

**X12 Element Positioning:**
- Position 0: Segment ID itself (ISA, GS, NM1, etc.)
- Position 1: First element (e.g., NM101, CLP01)
- Position n: Nth element (e.g., NM109 = position 9)

---

## ParseContext Push/Pop Pattern

Stack-based hierarchy tracking eliminates manual path construction bugs:

```java
public class ParseContext {
    private final Deque<ContextFrame> stack;
    private final String fileKey;
    private String currentPath;

    /**
     * Push instance-based context: /{section_code}/I={occurrence}
     * Used for repeating loops (HL, CLM, CLP, LX)
     */
    public String push(String sectionCode) {
        int occurrence = stack.peek().getNextChildOccurrence(sectionCode);
        String pathSegment = "/" + sectionCode + "/I=" + occurrence;

        ContextFrame frame = new ContextFrame(sectionCode, occurrence, fullPath);
        stack.push(frame);
        currentPath = currentPath + pathSegment;

        return pathSegment;  // Returns key_hash segment
    }

    /**
     * Push qualifier-based context: /{section_code}/Q={qualifier}
     * Used for multi-purpose segments (NM1*QC, NM1*PR, REF*6R, etc.)
     */
    public String pushWithQualifier(String sectionCode, String qualifierValue) {
        String pathSegment = "/" + sectionCode + "/Q=" + qualifierValue;
        ContextFrame frame = new ContextFrame(sectionCode, qualifierValue, occurrence, fullPath);
        stack.push(frame);
        currentPath = currentPath + pathSegment;

        return pathSegment;
    }

    /**
     * Pop to parent of target section (for sibling transitions).
     */
    public void popToParentOf(String targetSectionCode) {
        HierarchyEntry entry = config.getHierarchy(targetSectionCode);
        String targetParent = entry.getParent();
        popTo(targetParent);  // Pop until we reach target parent
    }

    /**
     * Generate key for leaf segment without pushing onto stack.
     */
    public String generateChildKeyPath(String sectionCode) {
        int occurrence = stack.peek().getNextChildOccurrence(sectionCode);
        return "/" + sectionCode + "/I=" + occurrence;
    }
}
```

**Key Path Structure:**
```
{fileKey}/isa_01_envelope/I=1/gs/I=1/st/I=1/hl_20_billing/I=1/clm/I=1/nm1_2010aa/Q=85
         └── file hash    └── ISA  └── GS  └── ST  └── HL level 20 └── CLM └── NM1*85
```

**When to Push vs Generate:**
- **Push:** Parent segments (ISA, GS, ST, HL, CLM, CLP, LX, SBR) - creates new context level
- **Generate:** Leaf segments (NM1, REF, DTP, CAS, etc.) - creates key without new context

---

## RowAccumulator Flush Pattern

Memory-bounded row batching with configurable thresholds:

```java
public class RowAccumulator {
    private final int maxRowsPerTable;     // e.g., 10000
    private final long bufferSizeBytes;    // e.g., 50MB per table
    private final long maxTotalMemoryBytes; // e.g., 500MB global cap

    private final Map<String, List<Map<String, Object>>> tableBuffers;
    private String pendingFlushTable;
    private FlushReason pendingFlushReason;

    public enum FlushReason {
        ROW_COUNT,      // Per-table row count exceeded
        TABLE_SIZE,     // Per-table size exceeded
        GLOBAL_MEMORY   // Global memory cap exceeded (flush largest table)
    }

    /**
     * Add row to table buffer, check thresholds.
     */
    public void add(String tableName, Map<String, Object> row) {
        tableBuffers.computeIfAbsent(tableName, k -> new ArrayList<>()).add(row);
        updateMetrics(tableName);
        checkThresholds(tableName);
    }

    /**
     * Check if any flush threshold exceeded (OR logic).
     */
    private void checkThresholds(String tableName) {
        if (rowCount >= maxRowsPerTable) {
            pendingFlushTable = tableName;
            pendingFlushReason = FlushReason.ROW_COUNT;
        } else if (tableSize >= bufferSizeBytes) {
            pendingFlushTable = tableName;
            pendingFlushReason = FlushReason.TABLE_SIZE;
        } else if (totalEstimatedBytes >= maxTotalMemoryBytes) {
            pendingFlushTable = findLargestTable();
            pendingFlushReason = FlushReason.GLOBAL_MEMORY;
        }
    }

    public boolean shouldFlush() { return pendingFlushTable != null; }
    public String getTableToFlush() { return pendingFlushTable; }
    public List<Map<String, Object>> drainTable(String tableName) { /* ... */ }
}
```

**Usage Pattern:**
```java
RowAccumulator accumulator = new RowAccumulator(config);

// During parsing
accumulator.add("CLP_MASTER", clpRecord);
if (accumulator.shouldFlush()) {
    String table = accumulator.getTableToFlush();
    List<Map<String, Object>> rows = accumulator.drainTable(table);
    parquetWriter.writeRows(table, rows);
    stageWriter.putToStage(table);
}

// At end of file
for (String table : accumulator.getTableNames()) {
    List<Map<String, Object>> rows = accumulator.drainTable(table);
    // Write remaining rows
}
```

---

## Qualifier-Based Segment Lookup

Multi-purpose segments (NM1, REF, DTP, AMT) require qualifier-aware resolution:

```java
protected HierarchyEntry findHierarchyEntry(X12Segment segment) {
    String segmentId = segment.getSegmentId();
    List<String> contextStack = context.getStackSectionCodes();

    // Qualifier-based segments use element 01 as discriminator
    if (config.isQualifierBasedSegment(segmentId)) {
        String qualifierValue = extractQualifierValue(segment);
        if (qualifierValue != null) {
            // Try specific lookup: NM1*QC in Loop 2100 -> nm1_2100_patient
            HierarchyEntry entry = config.findEntryBySegmentAndQualifier(
                segmentId, qualifierValue, contextStack);
            if (entry != null) return entry;
        }
    }

    // Fall back to context-aware lookup (segment ID + current loop)
    return config.findEntryBySegment(segmentId, contextStack);
}

protected String extractQualifierValue(X12Segment segment) {
    String segmentId = segment.getSegmentId();
    // All qualifier-based segments use element 01
    if ("NM1".equals(segmentId) || "REF".equals(segmentId) ||
        "DTP".equals(segmentId) || "AMT".equals(segmentId)) {
        return segment.getElement(1);  // NM101, REF01, DTP01, AMT01
    }
    return null;
}
```

**Qualifier Examples:**
| Segment | Qualifier Element | Example | Maps To |
|---------|-------------------|---------|---------|
| NM1*QC | NM101=QC | Patient | nm1_2100_patient |
| NM1*PR | NM101=PR | Payer | nm1_2010bb_payer |
| NM1*82 | NM101=82 | Rendering Provider | nm1_4_2100_service_provider |
| REF*6R | REF01=6R | Claim Reference | ref_6r_subscriber_id |
| DTP*472 | DTP01=472 | Service Date | dtp_472_service_date |

---

## File Key Generation

SHA-256 hash of file contents provides unique, reproducible file identification:

```java
protected String generateFileKey(Path filePath) throws IOException {
    MessageDigest md = MessageDigest.getInstance("SHA-256");
    byte[] content = Files.readAllBytes(filePath);
    byte[] digest = md.digest(content);

    StringBuilder sb = new StringBuilder();
    for (byte b : digest) {
        sb.append(String.format("%02x", b));
    }
    return sb.toString();  // 64-character hex string
}
```

**Key Usage in Records:**
```java
// File key becomes base of all key_hash paths
String fileKey = "a1b2c3d4e5f6...";  // SHA-256 hash

// key_hash for segment:
// {fileKey}/isa_01_envelope/I=1/gs/I=1/st/I=1/clp/I=5/nm1_2100/Q=QC
record.put("file_key", fileKey);
record.put("key_hash", keyPath);
record.put("parent_key_hash", context.getLastParentKeyPath());
```

**Parent-Child Linking:**
- `file_key`: Links all records to source file
- `key_hash`: Unique identifier for this record
- `parent_key_hash`: Links to parent record for hierarchy traversal
