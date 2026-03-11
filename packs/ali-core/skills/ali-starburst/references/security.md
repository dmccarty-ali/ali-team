# ali-starburst - Security Reference

Detailed security configurations, policies, and examples.

---

## Authentication Configuration

### LDAP Configuration

```properties
# config.properties
http-server.authentication.type=LDAP
authentication.ldap.url=ldaps://ldap.example.com:636
authentication.ldap.user-bind-pattern=uid=${USER},ou=users,dc=example,dc=com
authentication.ldap.group-authorization-search-pattern=(&(objectClass=groupOfNames)(member=${USER}))
```

### Active Directory

Similar to LDAP configuration with AD-specific settings:

```properties
http-server.authentication.type=LDAP
authentication.ldap.url=ldaps://ad.example.com:636
authentication.ldap.user-bind-pattern=${USER}@ad.example.com
```

### OAuth 2.0

```properties
http-server.authentication.type=OAUTH2
http-server.authentication.oauth2.issuer=https://auth.example.com
http-server.authentication.oauth2.client-id=starburst-client
http-server.authentication.oauth2.client-secret=<secret>
```

### Kerberos

```properties
http-server.authentication.type=KERBEROS
http.server.authentication.krb5.service-name=trino
http.server.authentication.krb5.keytab=/etc/trino/trino.keytab
```

---

## Apache Ranger

### Ranger Configuration

```properties
# access-control.properties
access-control.name=ranger
ranger.service-name=starburst-trino
ranger.policy-rest-url=http://ranger:6080
```

### Ranger Policies

**Policy Types:**
- **Database-level**: Allow/deny access to catalogs
- **Table-level**: Allow/deny access to schemas and tables
- **Column-level**: Mask or deny specific columns
- **Row-level**: Filter rows based on user attributes

### Example Policy

```yaml
Policy: PII Protection
Resource:
  Catalog: iceberg
  Schema: gold
  Table: customers
  Column: ssn, credit_card
Users: analysts_group
Access: Deny
```

---

## Row-Level Security (RLS)

### Function-Based Filtering

```sql
-- Define row filter function
CREATE FUNCTION iceberg.security.region_filter()
RETURNS VARCHAR
RETURN current_user_region();  -- Custom function returning user's region

-- Apply row filter to table
ALTER TABLE iceberg.gold.sales
SET PROPERTIES
security.row.filter = 'region = security.region_filter()';

-- User in US region queries table
SELECT * FROM iceberg.gold.sales;
-- Implicitly filters: WHERE region = 'US'
```

### User Attribute Based

```sql
-- Filter based on user department
ALTER TABLE iceberg.gold.orders
SET PROPERTIES
security.row.filter = 'department = current_user_department()';
```

---

## Column Masking

### Role-Based Masking

```sql
-- Mask SSN column based on role
ALTER TABLE iceberg.gold.customers
ALTER COLUMN ssn SET PROPERTIES
masking.policy = 'CASE WHEN current_user_role() = ''admin''
                       THEN ssn
                       ELSE ''XXX-XX-'' || SUBSTR(ssn, 8, 4)
                  END';

-- Regular user sees: XXX-XX-1234
-- Admin sees: 123-45-1234
```

### Email Masking

```sql
-- Mask email addresses
ALTER TABLE iceberg.gold.users
ALTER COLUMN email SET PROPERTIES
masking.policy = 'CASE WHEN current_user_role() IN (''admin'', ''manager'')
                       THEN email
                       ELSE CONCAT(SUBSTR(email, 1, 2), ''***@'',
                                   SPLIT_PART(email, ''@'', 2))
                  END';

-- Regular user sees: ab***@example.com
-- Admin sees: alice.bob@example.com
```

---

## Open Policy Agent (OPA)

### OPA Configuration

```properties
# access-control.properties
access-control.name=opa
opa.policy.uri=http://opa:8181/v1/data/trino/allow
opa.policy.row-filters-uri=http://opa:8181/v1/data/trino/rowFilters
opa.policy.column-masking-uri=http://opa:8181/v1/data/trino/columnMask
```

### Rego Policy Examples

#### Basic Access Control

```rego
package trino

# Allow access to sales table for analysts
allow {
    input.action.operation = "SelectFromColumns"
    input.action.resource.table.tableName = "sales"
    input.context.identity.user = "analyst"
}

# Deny DELETE operations for non-admins
deny {
    input.action.operation = "DeleteFromTable"
    input.context.identity.groups[_] != "admin"
}
```

#### Row-Level Security Policy

```rego
package trino

# Row filter: show only user's region
rowFilters[{"expression": expression}] {
    input.action.resource.table.tableName = "sales"
    expression := sprintf("region = '%s'", [user_region(input.context.identity.user)])
}

# Helper function to lookup user's region
user_region(user) = region {
    user_regions := {
        "alice": "US",
        "bob": "EU",
        "charlie": "APAC"
    }
    region := user_regions[user]
}
```

#### Column Masking Policy

```rego
package trino

# Mask SSN for non-admin users
columnMask[{"expression": expression}] {
    input.action.resource.table.tableName = "customers"
    input.action.resource.column.columnName = "ssn"
    input.context.identity.groups[_] != "admin"
    expression := "CONCAT('XXX-XX-', SUBSTR(ssn, 8, 4))"
}
```

#### Time-Based Access

```rego
package trino

# Allow access only during business hours
allow {
    input.action.operation = "SelectFromColumns"
    hour := time.clock(time.now_ns())[0]
    hour >= 8
    hour <= 18
}
```

---

**Last Updated:** 2026-02-16
