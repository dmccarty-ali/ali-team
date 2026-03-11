# ali-sap-mdg - Environment Management Reference

This reference contains detailed transport management, client strategy, and deployment procedures for SAP MDG environments.

---

## Transport Management

**MDG follows standard SAP transport patterns using Transport Management System (TMS).**

### Transport Objects

**What gets transported:**

1. **Customizing (Configuration)**
   - BRFplus decision tables (DT_SINGLE_VAL, DT_USER_AGT_GRP, DT_CONDITIONS)
   - Workflow rules (USMD_SSW_RULE)
   - Matching rules (USMD_RULE)
   - Field mappings (data model configuration)
   - UI configurations (Floorplan Manager, Fiori)
   - Validation rules

2. **Data Model Extensions**
   - Custom entities (Z-tables)
   - Custom attributes (additional fields)
   - Relationships between entities
   - Extensibility configuration

3. **UI Configurations**
   - Floorplan Manager configurations
   - Fiori tile definitions
   - Search configurations
   - UI personalizations (if exported)

4. **Access Control**
   - Role definitions (PFCG)
   - Authorization objects
   - User-to-role assignments (usually manual in PROD)

**What does NOT get transported:**
- Transactional data (customer records, material records)
- User credentials
- System-specific configurations (RFC destinations, logical systems)

---

### Transport Route

**Standard three-tier landscape:**

```
Development (DEV)
  Client: 100
  ↓ Transport Request (auto-import to QA)
Quality/Test (QA)
  Client: 200
  ↓ Transport Request (manual import to PROD after approval)
Production (PROD)
  Client: 300
```

**Transport Flow:**
1. Developer creates change in DEV (client 100)
2. Change saved to Transport Request (TR)
3. Developer releases TR
4. TR automatically imports to QA (client 200)
5. QA team tests, approves
6. Basis team manually imports TR to PROD (client 300)

---

### Key Transactions

| Transaction | Purpose | Who Uses |
|-------------|---------|----------|
| **STMS** | Transport Management System configuration | Basis team |
| **SE09** | Display Transport Organizer | All developers |
| **SE10** | Customizing Organizer | All developers |
| **USMD_TRANSPORT** | MDG-specific transport utilities | MDG developers |
| **SCC1** | Client copy (same system) | Basis team |
| **SCC9** | Remote client copy | Basis team |

---

### Creating Transport Requests

**Step 1: Create Transport Request (Transaction: SE09)**

```
1. Open SE09 (Transport Organizer)
2. Click "Create" button
3. Select type:
   - "Workbench Request" (for data model, code)
   - "Customizing Request" (for BRFplus, workflows)
4. Enter description: "MDG - Add credit limit approval workflow"
5. Save (generates TR number, e.g., DEVK900123)
```

**Step 2: Make changes in MDG**

```
1. Open USMD_SSW_RULE (configure BRFplus)
2. Make changes to decision tables
3. Save changes
4. System prompts for Transport Request
5. Enter TR number (DEVK900123)
```

**Step 3: Release Transport Request**

```
1. Open SE09, find your TR (DEVK900123)
2. Select TR, click "Release"
3. System validates changes
4. If validation passes, TR released
5. TR automatically queued for import to QA
```

---

### Transport Dependencies

**Problem:** Transports may depend on each other (e.g., field must exist before workflow can reference it)

**Solution: Document transport sequence**

**Example:**
```
TR001: DEVK900120 - Add custom field CREDIT_RATING to Customer
TR002: DEVK900121 - Add CREDIT_RATING to BRFplus decision table
TR003: DEVK900122 - Update UI to display CREDIT_RATING

Import order: TR001 → TR002 → TR003 (sequential, cannot be parallel)
```

**Best Practice:**
- Create transports in dependency order
- Document dependencies in TR description
- Test import sequence in sandbox before QA
- Use "transport of copies" (STMS) to bundle dependent TRs

---

## Client Strategy

### Standard Three-Client Model

| Client | Purpose | Configuration |
|--------|---------|---------------|
| **000** | TMS configuration, reference | Read-only, no customizing |
| **100** | Development client | Customizing, BRFplus rules, data model |
| **200** | Quality/Test client | Testing, UAT, no development |
| **300** | Production client | Live governance operations, no development |

**Client 000:** SAP-delivered reference client, do not modify

**Client 100:** All development work happens here
- Developers have full access
- Transports created and released from here
- Test data for unit testing

**Client 200:** Quality assurance and user acceptance testing
- Read-only for developers (no transport creation)
- Test data for UAT
- QA team validates transports from DEV

**Client 300:** Live production environment
- No development allowed
- Real master data
- Restricted access (only MDG stewards, admins)

---

### Four-Client Model (Larger Organizations)

| Client | Purpose |
|--------|---------|
| **100** | Development |
| **200** | Integration testing (system-to-system testing) |
| **300** | User acceptance testing (UAT) |
| **400** | Production |

**Use when:**
- Large teams (>5 developers) need isolated dev clients
- Complex integration testing requires dedicated environment
- UAT needs to be separate from integration testing

---

### Client Copy Strategies

**Scenario: Need to refresh QA with PROD data**

**Option 1: Client Copy (Same System)**
```
Transaction: SCC1 (local client copy)

Source: Client 300 (PROD)
Target: Client 200 (QA)
Profile: SAP_ALL (copy everything)

Duration: 2-4 hours (depends on data volume)

Result: QA client has copy of PROD data
```

**Option 2: Remote Client Copy (Different System)**
```
Transaction: SCC9 (remote client copy)

Source System: PROD_SYSTEM, Client 300
Target System: QA_SYSTEM, Client 200
Profile: SAP_ALL

Duration: 4-8 hours (network transfer)

Result: QA system has copy of PROD data
```

**Best Practice:**
- Copy PROD → QA quarterly (keeps QA data realistic)
- Copy PROD → DEV semi-annually (dev needs test data)
- Anonymize sensitive data after copy (GDPR compliance)

---

## Deployment Checklist

**Before promoting changes to production:**

### Technical Validation

- [ ] All BRFplus decision tables tested in QA
  - [ ] All approval paths tested (approve, reject, send back)
  - [ ] All agent determination scenarios tested
  - [ ] Conditional routing tested (boundary values, edge cases)

- [ ] Matching rules validated against sample data
  - [ ] Deterministic rules tested (100% matches)
  - [ ] Probabilistic rules tested (partial matches)
  - [ ] False positive rate <5%
  - [ ] False negative rate <3%

- [ ] Workflows tested end-to-end
  - [ ] Create change request → Approve → Activate
  - [ ] Create change request → Reject → Return to requestor
  - [ ] Mass processing workflows tested with >100 records
  - [ ] Workflow processing time <2 seconds per CR

- [ ] Integration endpoints tested
  - [ ] MDI replication tested (DEV → QA)
  - [ ] IDoc distribution tested (sample customer/material)
  - [ ] REST/OData API tested (GET, POST, PATCH)
  - [ ] Error handling tested (target system down, invalid data)

- [ ] DQM address validation tested
  - [ ] All supported countries tested
  - [ ] Valid address (auto-approve) tested
  - [ ] Invalid address (block) tested
  - [ ] Partial match (manual review) tested

### User Readiness

- [ ] User training completed
  - [ ] Stewards trained on MDG UI
  - [ ] Stewards trained on manual match review
  - [ ] Stewards trained on data quality rules
  - [ ] Approvers trained on workflow approval process

- [ ] Roles and authorizations configured
  - [ ] Data steward roles assigned
  - [ ] Approver roles assigned
  - [ ] Read-only analyst roles assigned
  - [ ] System admin roles assigned

- [ ] User acceptance testing (UAT) completed
  - [ ] All user scenarios tested
  - [ ] User feedback incorporated
  - [ ] UAT sign-off obtained

### Data Readiness

- [ ] Data migration plan approved
  - [ ] Legacy data extraction complete
  - [ ] Data cleansing rules applied
  - [ ] Duplicate resolution complete
  - [ ] Data migration tested in QA

- [ ] Initial load tested
  - [ ] Sample data loaded (1000 records)
  - [ ] Full data load tested (production volume)
  - [ ] Load performance acceptable (<4 hours)
  - [ ] Data validation passed (100% records valid)

### Operational Readiness

- [ ] Cutover plan approved
  - [ ] Cutover date/time scheduled
  - [ ] Communication plan (stakeholders notified)
  - [ ] Freeze period defined (no changes during cutover)
  - [ ] Go-live checklist prepared

- [ ] Rollback plan documented
  - [ ] How to revert transports
  - [ ] How to restore data (if needed)
  - [ ] Contact list (escalation path)
  - [ ] Decision criteria (when to rollback)

- [ ] Monitoring configured
  - [ ] BRFplus workflow monitoring (USMD_SSW_RULE)
  - [ ] Integration monitoring (MDI/IDoc/API)
  - [ ] Performance monitoring (response times)
  - [ ] Error alerting (email/SMS for critical errors)

---

## Go-Live Procedure

### Pre-Go-Live (T-1 week)

```
Monday:
- [ ] Final transport import to QA
- [ ] Smoke tests in QA (all critical scenarios)
- [ ] UAT sign-off obtained

Tuesday-Thursday:
- [ ] Final user training sessions
- [ ] Dress rehearsal (full cutover in QA)
- [ ] Validate rollback procedure

Friday:
- [ ] Change freeze begins (no new transports)
- [ ] Final transport package prepared
- [ ] Go/No-Go meeting
```

### Go-Live Day (Saturday, 2 AM - 10 AM)

```
2:00 AM:
- [ ] Start cutover
- [ ] Backup production client (SCC1 or database backup)

2:30 AM:
- [ ] Import transports to PROD (sequential order)
- [ ] Validate each transport (check STMS import log)

4:00 AM:
- [ ] Smoke tests in PROD
  - [ ] Create test change request (customer, material)
  - [ ] Test workflow approval
  - [ ] Test integration (MDI/IDoc)
  - [ ] Test DQM validation

6:00 AM:
- [ ] Hypercare monitoring begins
- [ ] Help desk on standby
- [ ] Data stewards online

8:00 AM:
- [ ] Business users begin work
- [ ] Monitor for errors (first 2 hours critical)

10:00 AM:
- [ ] Go-live status meeting
- [ ] Decision: Continue or rollback
```

### Post-Go-Live (T+1 week)

```
Day 1-3 (Hypercare):
- [ ] 24/7 monitoring
- [ ] Immediate response to issues
- [ ] Daily status meeting

Day 4-7:
- [ ] Transition to standard support
- [ ] Document lessons learned
- [ ] Plan for next domain/phase
```

---

## Rollback Procedure

**When to rollback:**
- Critical error affecting >50% of users
- Data corruption detected
- Integration failure (cannot distribute master data)
- Performance degradation (>5 second response times)

**Rollback steps:**

1. **Stop new transactions** (announce to users)
2. **Revert transports** (Transaction: STMS)
   ```
   STMS → Import Queue → Select TR → Revert
   ```
3. **Restore data** (if data corruption occurred)
   ```
   Restore from backup taken at 2:00 AM
   ```
4. **Validate rollback** (smoke tests)
5. **Resume operations** (announce to users)
6. **Post-mortem meeting** (analyze what went wrong)

---

## Best Practices

### 1. Transport Discipline

- Always use transports (never manual changes in QA/PROD)
- One logical change per transport (not a grab-bag of unrelated changes)
- Document transport dependencies in descriptions
- Test transport import in sandbox before QA

### 2. Client Management

- Never develop in production client (always DEV → QA → PROD)
- Refresh QA client from PROD quarterly (keeps test data realistic)
- Anonymize sensitive data in DEV (GDPR compliance)

### 3. Deployment Windows

- Schedule go-live during low-traffic periods (weekends, after-hours)
- Allow 4-8 hour window for transports + testing
- Plan rollback window (2-3 hours if needed)

### 4. Communication

- Notify stakeholders 2 weeks before go-live
- Send reminder 1 week before (freeze begins)
- Send final reminder day before (system unavailable during cutover)
- Send post-go-live summary (success metrics, known issues)
