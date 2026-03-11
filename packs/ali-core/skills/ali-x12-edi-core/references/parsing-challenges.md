# ali-x12-edi-core - Parsing Challenges Reference

## Common Parsing Challenges

### 1. Multiple Occurrences of Same Segment

**Challenge:** NM1 can appear dozens of times in different loops with different meanings

**Solution:**
- Master table captures all occurrences
- section_code identifies context
- Split tables separate by use case

### 2. Hierarchical Loop Structure

**Challenge:** Loops nest within loops (2000 → 2010 → 2100 → 2110)

**Solution:**
- Track loop hierarchy in section_code
- Use parent_key_hash to link child to parent
- Maintain loop start/end boundaries during parsing

### 3. Conditional Segments

**Challenge:** Many segments are situational (present only under specific conditions)

**Solution:**
- Design for NULL values in optional columns
- Don't assume segment presence
- Use situational flags in metadata (tbl_defs.yaml)

### 4. Variable Delimiters

**Challenge:** Delimiters can vary between interchanges (ISA16 defines component separator)

**Solution:**
- Read ISA segment first to determine delimiters
- Use delimiter configuration for parsing
- Store delimiter info in file metadata
