# Large-File Protocol

**Full spec for:** skills/plan-builder/references/large-file-protocol.md
**Referenced from:** SKILL.md Implement Mode Walkthrough STEP 2/3

---

## Threshold Definition

A source file is "large" for plan purposes if it exceeds 500 lines.

---

## Plan-Agent Obligation

When producing a plan that references a source file over the threshold, plan-agent must determine the file's actual line count empirically — by reading the file with offset/limit or running `wc -l` via Bash. Plan-agent does not estimate line counts. Plan-agent does not omit the reading strategy because the file seems familiar. The reading strategy is a required entry in the plan's Phase 0 section.

---

## Reading Strategy Format

In the plan, Phase 0 section:

```
Large File Protocol:
- [filename] ([actual line count] lines): Read lines [start-end] for [what is there].
  Search for [pattern] to locate [specific value].
```

## Example

```
Large File Protocol:
- DATABASE_SCHEMA.sql (1,247 lines): Read lines 1-100 for table definitions.
  Search for "CREATE TABLE users" to locate column definitions for the users table.
  Search for "CREATE INDEX" to locate all indexed columns.
```

---

## Implementing Agent Obligation

Before the write step of any phase that uses a value from a large file, the implementing agent must:

1. Read the specified sections using the Phase 0 reading strategy
2. Confirm values match the plan's assumptions
3. If values differ: stop and report via ISPR (what the plan assumed, what the source actually says, impact on this phase); wait for approval before revising the plan or implementing with the corrected value

---

## Connection to VERIFY Protocol

If the plan-agent reads a large source file and cannot confirm a specific value due to file size, the value is treated as unverifiable during planning and carries a VERIFY: flag per verify-flag-protocol.md.

The large-file protocol and the VERIFY protocol are complementary: the large-file reading strategy reduces the number of values that need VERIFY flags; the VERIFY flag covers what the reading strategy cannot reach.

See: skills/plan-builder/references/verify-flag-protocol.md
