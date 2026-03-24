# VERIFY Flag Protocol

**Full spec for:** skills/plan-builder/references/verify-flag-protocol.md
**Referenced from:** ali-plan-agent.md Step 4, SKILL.md Implement Mode Clean Criteria criterion 7

---

## Flag Format

When writing any implementation-critical value that cannot be verified directly from a source file during planning, plan-agent must flag it with:

```
VERIFY: [assumed value or claim] — confirm against [filename] before implementing
```

---

## Examples

```
VERIFY: primary key column is named "user_id" — confirm against DATABASE_SCHEMA.sql before implementing
```

```
VERIFY: current head migration revision is "a1b2c3d4e5f6" — confirm against migrations/versions/ before implementing
```

```
VERIFY: health check endpoint is "/api/v1/health" — confirm against src/routes/health.py before implementing
```

---

## Implementing Agent Contract

The implementing agent must resolve every VERIFY flag before the write step of the relevant phase:

1. Locate the named source file
2. Confirm the assumed value against the source
3. If the confirmed value matches: proceed with the implementation step
4. If the confirmed value differs: stop; report via ISPR (what the plan assumed, what the source actually says, impact on this phase); wait for approval before revising the plan or implementing with the corrected value

---

## Plan-Agent Step 4 Addition

Before writing any implementation-critical value to the plan, check:
- Can this value be verified by reading a source file accessible during planning?
- If YES: read the file, confirm the value, cite the file in the plan
- If NO: write the value with a VERIFY: flag and the source file to check

---

## Self-Check Item

```
- [ ] All implementation-critical values that could be verified from accessible source files
      during planning have been verified and cite the source file; values that could not
      be verified carry a VERIFY: flag with the source file to check
```
