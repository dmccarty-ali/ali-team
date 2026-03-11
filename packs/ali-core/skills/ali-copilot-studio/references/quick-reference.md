# ali-copilot-studio - Quick Reference

This reference provides quick lookup tables and commands for common CoPilot Studio operations.

---

## Common Actions (Code Snippets)

### File Upload to SharePoint

```yaml
action: upload_to_sharepoint
connector: SharePoint
operation: Create file
parameters:
  site_url: https://company.sharepoint.com/sites/docs
  library: Shared Documents
  folder: /Expenses/2026
  file_name: {user_upload.name}
  file_content: {user_upload.content}
outputs:
  file_id: {output.Id}
  file_url: {output.LinkingUrl}
```

### Send Teams Notification

```yaml
action: send_teams_message
connector: Microsoft Teams
operation: Post message in chat or channel
parameters:
  recipient_type: Channel
  team: IT Support
  channel: Urgent
  message: |
    New ticket #{ticket_id}: {issue_summary}

    Priority: {priority}
    Created by: {user_email}
    Link: {ticket_url}
outputs:
  message_id: {output.messageId}
```

### Query Dataverse

```yaml
action: query_dataverse
connector: Microsoft Dataverse
operation: List rows
parameters:
  table: account
  filter: "accountnumber eq '{account_num}'"
  select: ["name", "address1_city", "telephone1"]
  top: 10
outputs:
  records: {output.value}
  count: {output['@odata.count']}
```

### Send Email (Outlook)

```yaml
action: send_email
connector: Office 365 Outlook
operation: Send an email (V2)
parameters:
  to: {user_email}
  subject: "Expense #{expense_id} Approved"
  body: |
    <p>Your expense report has been approved!</p>
    <ul>
      <li>Expense ID: {expense_id}</li>
      <li>Amount: ${amount}</li>
      <li>Reimbursement: Next payroll</li>
    </ul>
  importance: Normal
```

### Create Calendar Event

```yaml
action: create_event
connector: Office 365 Outlook
operation: Create event (V4)
parameters:
  calendar: Calendar
  subject: "Meeting: {meeting_topic}"
  start_time: {meeting_start}
  end_time: {meeting_end}
  required_attendees: {attendee_emails}  # semicolon-separated
  body: |
    {meeting_description}

    Join: {meeting_link}
  is_online_meeting: true
```

### Call REST API

```yaml
action: call_api
connector: HTTP
operation: Invoke an HTTP request
parameters:
  method: POST
  uri: https://api.company.com/v1/tickets
  headers:
    Content-Type: application/json
    Authorization: Bearer {api_token}
  body: |
    {
      "title": "{ticket_title}",
      "priority": "{priority}",
      "assignee": "{assignee}"
    }
outputs:
  status_code: {output.statusCode}
  response_body: {output.body}
```

---

## Entity Types

### Pre-Built Entities

| Entity | Example Input | Extracted Value | Use Case |
|--------|---------------|-----------------|----------|
| **Email** | "john@company.com" | john@company.com | Contact information |
| **Phone** | "(555) 123-4567" | +15551234567 | Contact information |
| **Date** | "next Friday" | 2026-02-20 | Scheduling, deadlines |
| **Time** | "3:30 PM" | 15:30:00 | Scheduling |
| **DateTime** | "tomorrow at 2pm" | 2026-02-17T14:00:00 | Meeting scheduling |
| **Currency** | "$1,234.56" | 1234.56 | Financial amounts |
| **Number** | "forty-two" | 42 | Quantities, counts |
| **Percentage** | "25%" | 0.25 | Rates, discounts |
| **URL** | "https://example.com" | https://example.com | Links, references |
| **Age** | "25 years old" | 25 | Demographics |
| **Dimension** | "5 feet 10 inches" | 177.8 (cm) | Measurements |
| **Temperature** | "72 degrees" | 22.2 (celsius) | Environmental data |

### Custom Entities

**List entity (closed set):**
```yaml
entity_name: Department
type: List
values:
  - Engineering
  - Sales
  - Marketing
  - Finance
  - HR
  - IT
synonyms:
  Engineering: ["Eng", "Dev", "Development", "Tech"]
  Sales: ["Revenue", "Business Development"]
```

**Pattern entity (regex):**
```yaml
entity_name: TicketNumber
type: Pattern
pattern: "TICKET-\\d{5,8}"
examples:
  - TICKET-12345
  - TICKET-9876543
```

**ML entity (trained):**
```yaml
entity_name: ProductName
type: Machine-Learned
training_phrases:
  - "I need help with Product A"
  - "How do I use the Enterprise Edition?"
  - "Tell me about the Premium plan"
```

---

## Debugging Commands

### PowerShell (Teams Admin)

```powershell
# Test topic in Test Canvas
Test-CopilotTopic -Topic "Password Reset" -Input "I forgot my password"

# View conversation transcript
Get-ConversationTranscript -SessionId {guid} -Format JSON

# Check action execution logs
Get-ActionLogs -ActionName "CreateTicket" -TimeRange "Last24Hours" -IncludeErrors

# View agent analytics
Get-CopilotAnalytics -AgentName "IT Support Bot" -Period "Last7Days"

# Validate OpenAPI spec
Test-OpenAPISpec -Path ./ticket-api.yaml -Verbose

# Export agent to solution
Export-CopilotAgent -Name "IT Support Bot" -SolutionName "IT_Automation_v1" -OutputPath ./exports/

# Import agent from solution
Import-CopilotAgent -SolutionPath ./exports/IT_Automation_v1.zip -Environment Production
```

### Azure CLI (Infrastructure)

```bash
# View Application Insights logs
az monitor app-insights query \
  --app copilot-insights \
  --analytics-query "customEvents | where name == 'ActionExecution' | take 100"

# Check Key Vault secrets
az keyvault secret list --vault-name company-secrets-prod

# Get secret value (use carefully, don't log)
az keyvault secret show --vault-name company-secrets-prod --name jira-client-secret

# Check managed identity assignments
az role assignment list --assignee {managed-identity-object-id}

# View resource group resources
az resource list --resource-group copilot-prod --output table
```

### KQL Queries (Application Insights)

**Error analysis:**
```kql
traces
| where timestamp > ago(24h)
| where severityLevel >= 3  // Warning and above
| summarize count() by cloud_RoleName, operation_Name, severityLevel
| order by count_ desc
```

**Action performance:**
```kql
customEvents
| where name == "ActionExecution"
| where timestamp > ago(7d)
| summarize
    count(),
    avg(todouble(customMeasurements.duration_ms)),
    percentile(todouble(customMeasurements.duration_ms), 50, 95, 99)
  by tostring(customDimensions.action_name)
| order by percentile_customMeasurements_duration_ms_99 desc
```

**User activity:**
```kql
customEvents
| where name == "ConversationStarted"
| where timestamp > ago(30d)
| summarize daily_users = dcount(tostring(customDimensions.user_id)) by bin(timestamp, 1d)
| render timechart
```

**Content violations:**
```kql
customEvents
| where name == "ContentViolation"
| where timestamp > ago(7d)
| summarize count() by tostring(customDimensions.violation_type)
| render piechart
```

---

## Topic Trigger Patterns

### Best Practices

| Pattern | Example | When to Use |
|---------|---------|-------------|
| **Action phrases** | "reset password", "create ticket", "submit expense" | User intent is clear action |
| **Question starters** | "how do I", "what is", "where can I" | Knowledge retrieval |
| **Problem statements** | "can't access", "not working", "error" | Troubleshooting |
| **Entities** | "VPN", "laptop", "benefits" | Topic-specific nouns |
| **Synonyms** | "reset password" = "forgot password" = "password reset" | Handle variations |

### Anti-Patterns

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Single trigger phrase | Won't match variations | Add 10-20 trigger variations |
| Too generic ("help") | Matches everything | Use specific triggers per topic |
| Technical jargon only | Users may not know terms | Include common language |
| No entity triggers | Misses noun-based queries | Add key entities as triggers |
| Overlapping triggers | Topics conflict | Make triggers mutually exclusive |

---

## HTTP Status Codes (API Actions)

| Code | Meaning | Action |
|------|---------|--------|
| **200** | OK | Success, process response |
| **201** | Created | Resource created successfully |
| **204** | No Content | Success, no response body |
| **400** | Bad Request | Invalid input, check parameters |
| **401** | Unauthorized | Authentication failed, check credentials |
| **403** | Forbidden | No permission, check authorization |
| **404** | Not Found | Resource doesn't exist |
| **429** | Too Many Requests | Rate limited, implement retry with backoff |
| **500** | Internal Server Error | Server error, retry or escalate |
| **502** | Bad Gateway | Proxy/gateway error, check network |
| **503** | Service Unavailable | Temporary outage, retry later |
| **504** | Gateway Timeout | Request timed out, optimize or increase timeout |

### Retry Strategy

```yaml
retry_policy:
  max_attempts: 3
  initial_interval: 1s
  max_interval: 30s
  backoff_multiplier: 2
  retryable_codes: [408, 429, 500, 502, 503, 504]
  non_retryable_codes: [400, 401, 403, 404]
```

---

## Power Automate Expression Cheat Sheet

### String Manipulation

```javascript
// Concatenation
concat('Hello ', variables('userName'))

// Substring
substring('Hello World', 0, 5)  // "Hello"

// Replace
replace('hello@company.com', '@company.com', '')  // "hello"

// To upper/lower
toUpper('hello')  // "HELLO"
toLower('HELLO')  // "hello"

// Trim whitespace
trim('  hello  ')  // "hello"

// Split string
split('a,b,c', ',')  // ["a", "b", "c"]

// Join array
join(['a', 'b', 'c'], ',')  // "a,b,c"
```

### Date/Time

```javascript
// Current timestamp
utcNow()  // "2026-02-16T14:30:00Z"

// Add time
addDays(utcNow(), 7)  // 7 days from now
addHours(utcNow(), 2)  // 2 hours from now

// Format date
formatDateTime(utcNow(), 'yyyy-MM-dd')  // "2026-02-16"
formatDateTime(utcNow(), 'MMMM dd, yyyy')  // "February 16, 2026"

// Convert timezone
convertFromUtc(utcNow(), 'Pacific Standard Time')

// Day of week
dayOfWeek(utcNow())  // 0 (Sunday) to 6 (Saturday)
```

### Conditional Logic

```javascript
// If-then-else
if(equals(variables('status'), 'approved'), 'Proceed', 'Reject')

// Greater/less than
greater(variables('amount'), 1000)
less(variables('amount'), 500)

// And/Or
and(equals(variables('status'), 'active'), greater(variables('age'), 18))
or(equals(variables('type'), 'A'), equals(variables('type'), 'B'))

// Not
not(equals(variables('flag'), true))

// Contains
contains('hello world', 'world')  // true

// Empty check
empty(variables('myVariable'))  // true if null or empty
```

### Array Operations

```javascript
// Length
length(variables('myArray'))

// First/Last
first(variables('myArray'))
last(variables('myArray'))

// Take/Skip
take(variables('myArray'), 3)  // First 3 items
skip(variables('myArray'), 2)  // Skip first 2 items

// Join
join(variables('myArray'), ', ')

// Contains
contains(variables('myArray'), 'item')
```

### JSON

```javascript
// Parse JSON
json('{"name": "John", "age": 30}')

// Access property
json(variables('jsonString'))['name']

// Convert to JSON string
string(variables('jsonObject'))
```

---

## Environment Variables

### Common Variables

| Variable | Purpose | Example Value |
|----------|---------|---------------|
| `ENVIRONMENT_NAME` | Current environment | Development, Production |
| `TENANT_ID` | Azure AD tenant ID | 00000000-0000-0000-0000-000000000000 |
| `SHAREPOINT_SITE_URL` | SharePoint site | https://company.sharepoint.com/sites/docs |
| `API_BASE_URL` | External API endpoint | https://api.company.com/v1 |
| `JIRA_PROJECT_KEY` | Default Jira project | SUPPORT |
| `SUPPORT_EMAIL` | Escalation email | support@company.com |
| `MAX_FILE_SIZE_MB` | Upload limit | 10 |
| `SESSION_TIMEOUT_MIN` | Conversation timeout | 30 |

### Usage in Power Automate

```javascript
// Reference environment variable
variables('env')['API_BASE_URL']

// Set dynamically based on environment
if(
  equals(variables('env')['ENVIRONMENT_NAME'], 'Production'),
  'https://api.company.com/v1',
  'https://api-dev.company.com/v1'
)
```

---

## Adaptive Card Templates (Teams)

### Basic Card

```json
{
  "type": "AdaptiveCard",
  "version": "1.3",
  "body": [
    {
      "type": "TextBlock",
      "text": "Ticket Created",
      "weight": "bolder",
      "size": "large"
    },
    {
      "type": "TextBlock",
      "text": "Ticket #12345 has been created and assigned to IT Support.",
      "wrap": true
    },
    {
      "type": "FactSet",
      "facts": [
        {"title": "Priority", "value": "High"},
        {"title": "Created", "value": "2026-02-16 2:30 PM"},
        {"title": "Assigned to", "value": "John Doe"}
      ]
    }
  ],
  "actions": [
    {
      "type": "Action.OpenUrl",
      "title": "View Ticket",
      "url": "https://tickets.company.com/12345"
    }
  ]
}
```

### Input Form Card

```json
{
  "type": "AdaptiveCard",
  "version": "1.3",
  "body": [
    {
      "type": "TextBlock",
      "text": "Expense Approval Request",
      "weight": "bolder",
      "size": "large"
    },
    {
      "type": "Input.Text",
      "id": "comments",
      "placeholder": "Add comments (optional)",
      "isMultiline": true
    }
  ],
  "actions": [
    {
      "type": "Action.Submit",
      "title": "Approve",
      "data": {
        "action": "approve",
        "expense_id": "EXP-123"
      }
    },
    {
      "type": "Action.Submit",
      "title": "Reject",
      "data": {
        "action": "reject",
        "expense_id": "EXP-123"
      }
    }
  ]
}
```

---

## Best Practices Checklist

### Before Publishing Agent

- [ ] All topics have 10+ trigger variations
- [ ] Entities have validation rules
- [ ] Error handling implemented for all actions
- [ ] Fallback topic configured
- [ ] Escalation path to human defined
- [ ] Content moderation level set (High for production)
- [ ] Authentication configured securely (no hardcoded secrets)
- [ ] Test cases passed (happy path + error cases)
- [ ] Performance benchmarks met (response time <5s)
- [ ] Cross-channel testing complete (Teams, web, mobile)
- [ ] Documentation written (user guide, admin guide)
- [ ] Training materials prepared (if needed)
- [ ] Support plan in place
- [ ] Monitoring configured (Application Insights, alerts)
- [ ] Approval obtained (security, compliance, business owner)

### Monthly Maintenance

- [ ] Review analytics (usage, errors, satisfaction)
- [ ] Update knowledge sources (new/changed docs)
- [ ] Optimize slow actions (>3s execution time)
- [ ] Address low-satisfaction conversations
- [ ] Review and update trigger phrases (if topics not firing)
- [ ] Test agent with recent changes
- [ ] Check for security updates (connectors, APIs)
- [ ] Rotate secrets (API keys, client secrets)
- [ ] Review access controls (remove departed users)
- [ ] Update documentation (if agent changed)
