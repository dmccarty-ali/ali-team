---
name: ali-google-workspace
description: |
  Google Workspace administration - user lifecycle, group management, OrgUnit
  structure, domain policies, and Admin SDK patterns. Use when:

  PLANNING: Designing Workspace org structure, planning user provisioning workflows,
  architecting group-based access control, evaluating admin delegation strategies

  IMPLEMENTATION: Writing Admin SDK calls, configuring GAM commands, setting up
  provisioning automation, implementing group-based access policies

  GUIDANCE: Asking about Workspace admin best practices, how to manage user
  lifecycle, group nesting strategies, domain policy configuration

  REVIEW: Auditing Workspace admin configuration, reviewing user access patterns,
  validating group structures, checking domain security policies
---

# Google Workspace Administration

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing Workspace OrgUnit hierarchy for policy inheritance
- Planning user provisioning and deprovisioning workflows
- Architecting group-based access control strategies
- Evaluating admin role delegation across teams
- Designing automation for user lifecycle events

**Implementation:**
- Writing GAM commands for user, group, or OrgUnit operations
- Calling Admin SDK Directory, Reports, or Groups Settings APIs
- Setting up provisioning automation (SCIM or API-based)
- Implementing group-based service access policies
- Configuring domain sharing settings or security policies

**Guidance/Best Practices:**
- Asking about Workspace admin best practices
- Asking how to manage user offboarding safely
- Asking about group nesting and membership strategies
- Asking about domain-level security policy configuration
- Asking about admin role delegation patterns

**Review/Validation:**
- Auditing Workspace admin configuration for security gaps
- Reviewing user access patterns and group memberships
- Validating OrgUnit structure and policy inheritance
- Checking domain security policies (2SV, session length, sharing)
- Reviewing admin audit logs for unusual activity

## When NOT to Use This Skill

- Executing Workspace admin operations (delegate to ali-google-workspace-admin)
- Google Workspace developer APIs — Apps Script, Add-ins, Chat bots (different skill domain)
- GCP IAM or Cloud Identity (use ali-gcp skill)
- End-user Workspace help (Gmail tips, Docs formatting, etc.)

---

## Key Principles

- **Group-based access over individual**: Never grant service access directly to a user; always use a group as the access vehicle
- **OrgUnit inheritance is top-down**: Policies apply to an OU and all child OUs; plan hierarchy to minimize exceptions
- **Admin delegation scoped tightly**: Use delegated admin roles instead of super admin for routine tasks
- **Data preservation before deletion**: Always export or transfer user data before deleting any account
- **2SV enforcement is non-negotiable**: Never disable domain-wide two-step verification
- **Audit logs are the source of truth**: Admin activity logs capture all admin changes; review them regularly
- **Least-privilege admin roles**: Super admin is for emergencies; use delegated roles for day-to-day admin work

---

## User Lifecycle

### Provision (New User)

Standard onboarding sequence:

1. Create user account in correct OrgUnit
2. Assign license appropriate to role
3. Add to base groups (all-staff, department group, role-specific groups)
4. Set temporary password with forced change on first login
5. Send welcome email with login instructions

```bash
# Create user
ALIUNDE_GOOGLE_WORKSPACE_ADMIN=1 gam create user jane.smith@domain.com \
  firstname Jane lastname Smith \
  password [temp] changepassword true \
  org /Engineering

# Assign license
ALIUNDE_GOOGLE_WORKSPACE_ADMIN=1 gam user jane.smith@domain.com add license Google-Apps

# Add to groups
ALIUNDE_GOOGLE_WORKSPACE_ADMIN=1 gam update group all-staff@domain.com add members user jane.smith@domain.com
ALIUNDE_GOOGLE_WORKSPACE_ADMIN=1 gam update group engineering@domain.com add members user jane.smith@domain.com
```

### Suspend (Temporary Leave / Investigation)

Use suspend rather than delete for temporary situations. Suspended users cannot log in but data is preserved:

```bash
# Suspend user
ALIUNDE_GOOGLE_WORKSPACE_ADMIN=1 gam suspend user jane.smith@domain.com

# Revoke active sessions
ALIUNDE_GOOGLE_WORKSPACE_ADMIN=1 gam user jane.smith@domain.com signout

# Verify suspension
gam info user jane.smith@domain.com
```

### Delete (Permanent Offboarding)

**NEVER delete without completing data preservation steps first.**

Data preservation sequence before deletion:

1. Transfer Drive files to manager or designated owner
2. Export Gmail data if required for compliance
3. Forward email aliases to appropriate address or group
4. Document group memberships before removal
5. Wait for data transfer to complete (can take hours for large accounts)
6. Then delete the account

```bash
# Step 1: Transfer Drive ownership
ALIUNDE_GOOGLE_WORKSPACE_ADMIN=1 gam create datatransfer jane.smith@domain.com \
  gdrive manager@domain.com

# Step 2: Check data transfer status
gam print transfers status new

# Step 3: Add forwarding alias if needed
ALIUNDE_GOOGLE_WORKSPACE_ADMIN=1 gam create alias jane.smith@domain.com group team@domain.com

# Step 4: Document group memberships
gam show user jane.smith@domain.com groups

# Step 5: Delete account (only after data transfer complete)
ALIUNDE_GOOGLE_WORKSPACE_ADMIN=1 gam delete user jane.smith@domain.com
```

### Alias Management

```bash
# Add email alias to user
ALIUNDE_GOOGLE_WORKSPACE_ADMIN=1 gam create alias j.smith@domain.com user jane.smith@domain.com

# List aliases for user
gam info user jane.smith@domain.com

# Remove alias
ALIUNDE_GOOGLE_WORKSPACE_ADMIN=1 gam delete alias j.smith@domain.com
```

---

## Group Management

### Group Types

| Type | Purpose | Membership | External? |
|------|---------|-----------|-----------|
| **Security group** | Access control for services | Users, groups, SAs | No |
| **Email list** | Distribution list | Anyone invited | Optionally |
| **Discussion group** | Forum-style discussion | Managed | Optionally |

For access control, always use security groups with no external members.

### Group Nesting Strategy

Flat is simpler; nested enables role-based inheritance:

```
all-staff@domain.com           — everyone
  └── engineering@domain.com   — all engineers
       ├── platform@domain.com — platform team
       └── frontend@domain.com — frontend team
  └── operations@domain.com    — ops team
```

**Nesting rules:**
- A group can be a member of another group
- Avoid circular nesting (A member of B, B member of A)
- Max nesting depth of 5 is a practical limit for performance
- Dynamic groups (rules-based membership) require Google Workspace Enterprise

### Dynamic Groups

Dynamic groups update membership automatically based on user attributes (requires Enterprise edition):

```
# Example dynamic group query (set in Admin Console or API)
# All users in the Engineering OrgUnit:
orgUnitPath="/Engineering"

# All users with a specific custom attribute:
custom_attribute_name="department" custom_attribute_value="Engineering"
```

### Group Settings Reference

Key group settings to configure via Groups Settings API or Admin Console:

| Setting | Recommended Value | Notes |
|---------|-------------------|-------|
| Who can join | Invited users only | Never open membership for security groups |
| Who can post | Members of group | Restrict to prevent spam |
| External members allowed | Off (for security groups) | Only on for external collaboration groups |
| Message moderation | None (for internal groups) | Moderate for external-facing groups |
| Reply-to | Reply to group | Standard for distribution lists |

---

## OrgUnit Structure

### Flat vs. Deep Hierarchy

| Approach | When to Use |
|----------|-------------|
| Flat (few OUs) | Simple policy inheritance, small org |
| Deep (many OUs) | Complex policies per team/location/function |

**Practical guideline:** Keep OrgUnit depth to 4 levels or fewer. Every additional level adds policy management complexity.

### Policy Inheritance

- Policies flow from parent OU to child OU
- Child OU can override a parent policy for its subtree
- Exceptions (per-user overrides) should be rare; use OrgUnit moves instead

### Device Policies

OrgUnit determines which device management policies apply. Common OU-based device policy splits:

- `/Managed` — Full device management (MDM enrolled)
- `/BYOD` — Bring your own device policies
- `/Contractors` — Limited access, no device management

---

## Domain Policies

### 2-Step Verification (2SV)

2SV enforcement should always be on for the full domain. Key settings:

| Setting | Recommended |
|---------|-------------|
| 2SV enforcement | On for all users |
| New user enrollment period | 7 days max |
| Allowed 2SV methods | Security key preferred; TOTP allowed; SMS discouraged |
| Admin accounts | Security key required (hardware key) |

### Sharing Settings

Control external sharing at domain level:

| Setting | Recommended for Internal Orgs |
|---------|-------------------------------|
| Drive sharing outside domain | Off or limited to approved domains |
| Google Sites publishing | Off unless needed |
| Calendar sharing | Free/busy only for external users |
| Hangouts Meet external | Allow (needed for client meetings) |

### Session Length

Configure in Admin Console > Security > Google session control:

- **Super admin sessions:** 1 hour (re-auth required)
- **Standard users:** 8-24 hours depending on security posture
- **Contractors/external users:** 4 hours

### Google Workspace Marketplace Access

Control which third-party apps can access Workspace data:

- Default: Allow all apps (high risk)
- Recommended: Allow only approved apps (allowlist)
- For high-security orgs: Block all marketplace apps

---

## Admin SDK Patterns

### Directory API (Users and Groups)

Base URL: `https://admin.googleapis.com/admin/directory/v1`

Key endpoints:

```bash
# List users
GET /users?domain={domain}&maxResults=500

# Get user
GET /users/{userKey}

# Create user
POST /users
Body: {"primaryEmail":"...", "name":{"givenName":"...", "familyName":"..."}, "password":"..."}

# Update user
PUT /users/{userKey}

# Delete user
DELETE /users/{userKey}

# List groups
GET /groups?domain={domain}

# Get group members
GET /groups/{groupKey}/members
```

### Reports API

Base URL: `https://admin.googleapis.com/admin/reports/v1`

```bash
# Admin activity
GET /activity/users/all/applications/admin?startTime={iso}&maxResults=1000

# Login activity
GET /activity/users/all/applications/login?startTime={iso}

# Drive activity
GET /activity/users/{userKey}/applications/drive
```

### Groups Settings API

Base URL: `https://www.googleapis.com/groups/v1/groups/{groupEmail}`

```bash
# Get group settings
GET /{groupEmail}

# Update group settings
PUT /{groupEmail}
Body: {"allowExternalMembers":"false", "whoCanJoin":"INVITED_CAN_JOIN"}
```

---

## GAM Quick Reference

### User Operations

```bash
# List all users
gam print users allfields

# User info
gam info user [email]

# Create user
gam create user [email] firstname [F] lastname [L] password [P] org /[OU]

# Update user
gam update user [email] [field] [value]

# Suspend / unsuspend
gam suspend user [email]
gam unsuspend user [email]

# Reset password
gam update user [email] password [new-password] changepassword true

# Sign out all sessions
gam user [email] signout

# Delete user
gam delete user [email]
```

### Group Operations

```bash
# List groups
gam print groups

# Group info + members
gam info group [email]
gam print group-members group [email]

# Create group
gam create group [email] name "[Name]"

# Add / remove members
gam update group [email] add members user [user-email]
gam update group [email] remove members user [user-email]

# Delete group
gam delete group [email]
```

### OrgUnit Operations

```bash
# Show OU tree
gam show orgtree

# OU info
gam info org /[path]

# Create OU
gam create org /[parent]/[name] description "[desc]"

# Move user to OU
gam update user [email] org /[path]

# Delete OU (must be empty)
gam delete org /[path]
```

### Reports

```bash
# Login report
gam report login [optional: user [email]] [optional: start 2026-01-01]

# Admin activity report
gam report admin [optional: start 2026-01-01]

# User usage stats
gam report usage user [email] parameters accounts:used_quota_in_mb

# Token grants (3rd party app access)
gam print tokens user [email]
```

---

## Security Best Practices

### Super Admin Account Hygiene

- Maintain 2-5 super admin accounts; not more
- Super admin accounts should be break-glass accounts with hardware security keys
- Never use super admin for routine daily tasks; use delegated admin roles
- Review super admin list quarterly

### Admin Role Delegation

Create delegated admin roles scoped to the minimum needed:

| Role | Scope | Who Gets It |
|------|-------|-------------|
| User Management | User create/update/suspend | HR system, helpdesk |
| Group Admin | Group membership management | Team leads |
| Help Desk Admin | Password reset only | Tier 1 support |
| Storage Admin | Drive storage management | IT ops |

### Audit Log Monitoring

Monitor these Admin audit events for unusual activity:

- Super admin role assignment/removal
- 2SV policy changes
- Sharing policy changes
- User deletion
- OAuth token grants to third-party apps
- Failed login attempts (available in Login audit)

### Third-Party App Access

Review token grants regularly:

```bash
# List all token grants for a user
gam print tokens user john.doe@domain.com

# Revoke specific token
ALIUNDE_GOOGLE_WORKSPACE_ADMIN=1 gam delete token user john.doe@domain.com clientid [client-id]

# Revoke all tokens for a user (on offboarding)
ALIUNDE_GOOGLE_WORKSPACE_ADMIN=1 gam user john.doe@domain.com delete tokens
```

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Granting service access to individual users | Unscalable; breaks on user offboarding | Create a group for each service; grant to the group |
| Using super admin for routine admin tasks | Blast radius too large; audit noise | Create delegated admin roles scoped to the task |
| Deleting users without data transfer | Permanent Drive/Gmail data loss | Transfer data first; verify transfer complete before delete |
| No documentation before group deletion | Loss of membership record | Print and archive group members before deletion |
| Flat OrgUnit (everyone in root OU) | Cannot differentiate policies per team | Create meaningful OU hierarchy |
| External members in security groups | Unintended data access for outsiders | Separate distribution lists from security groups |
| Not reviewing token grants periodically | Third-party apps retain access after offboarding | Quarterly token grant review |
| Ignoring admin audit logs | Security incidents go undetected | Alert on super admin changes and policy changes |

---

## References

- [Google Workspace Admin Help](https://support.google.com/a/)
- [Admin SDK Directory API](https://developers.google.com/admin-sdk/directory/reference/rest)
- [Admin SDK Reports API](https://developers.google.com/admin-sdk/reports/reference/rest)
- [Groups Settings API](https://developers.google.com/admin-sdk/groups-settings/v1/reference/groups)
- [GAM Documentation](https://github.com/GAM-team/GAM/wiki)
- [Google Workspace Security Best Practices](https://workspace.google.com/intl/en/security/)
