# ali-copilot-studio - Example Patterns Reference

This reference provides complete implementation examples for common CoPilot Studio patterns.

---

## Pattern 1: SharePoint Knowledge Hub

### Use Case
Centralized documentation repository for agent grounding

### Structure

```
SharePoint Site: Company Knowledge Hub
├── IT Documentation (library)
│   ├── VPN Setup.docx
│   ├── Password Policy.pdf
│   └── Software Catalog.xlsx
├── HR Policies (library)
│   ├── PTO Policy.docx
│   ├── Benefits Guide.pdf
│   └── Onboarding Checklist.docx
└── Product Docs (library)
    ├── User Manual.pdf
    ├── API Reference.docx
    └── Release Notes.txt
```

### SharePoint Library Configuration

**IT Documentation library:**
- Permissions: IT staff (edit), All employees (read)
- Metadata columns:
  - Category: Networking, Security, Software, Hardware
  - Keywords: VPN, password, laptop, etc.
  - Last Reviewed: Date
  - Owner: IT staff member

**HR Policies library:**
- Permissions: HR staff (edit), All employees (read)
- Metadata columns:
  - Policy Type: Benefits, Time Off, Conduct, Onboarding
  - Effective Date: Date
  - Review Frequency: Annual, Biannual, As Needed

**Product Docs library:**
- Permissions: Product team (edit), All employees (read)
- Metadata columns:
  - Product: Product A, Product B
  - Document Type: Manual, API Docs, Release Notes
  - Version: 1.0, 2.0, etc.

### Agent Configuration

**Knowledge sources:**
1. Connect to SharePoint site "Company Knowledge Hub"
2. Select libraries:
   - IT Documentation
   - HR Policies
   - Product Docs
3. Sync schedule: Real-time
4. Permissions: All employees (read)

**Topics:**

**Topic: IT Help**
- Triggers: "VPN", "password", "laptop", "software"
- Query: IT Documentation library
- Fallback: Escalate to IT support

**Topic: HR Questions**
- Triggers: "PTO", "benefits", "onboarding", "policy"
- Query: HR Policies library
- Fallback: Escalate to HR

**Topic: Product Help**
- Triggers: "how do I use", "product manual", "API"
- Query: Product Docs library
- Fallback: Escalate to product support

### Example Conversation

```
User: "How do I set up VPN?"

Agent: [Queries IT Documentation library]
Agent: "Based on the VPN Setup Guide, here's how to set up VPN:

1. Download the VPN client from https://vpn.company.com
2. Install and launch the application
3. Enter your company email and password
4. Click 'Connect'

For detailed instructions, see the full VPN Setup Guide.

Was this helpful?"

User: "Yes, thanks!"

Agent: "Great! Let me know if you need anything else."
```

---

## Pattern 2: API Action with Authentication

### Use Case
Create Jira ticket from CoPilot conversation

### OpenAPI Specification

```yaml
openapi: 3.0.0
info:
  title: Jira Ticket API
  version: 1.0.0
  description: Create and manage Jira issues from CoPilot Studio
servers:
  - url: https://company.atlassian.net/rest/api/3
    description: Jira Cloud API

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      description: OAuth 2.0 access token

  schemas:
    IssueFields:
      type: object
      required:
        - project
        - summary
        - issuetype
      properties:
        project:
          type: object
          properties:
            key:
              type: string
              description: Project key (e.g., SUPPORT)
              default: SUPPORT
        summary:
          type: string
          description: Brief issue title
          example: "VPN connection fails on MacBook"
        description:
          type: string
          description: Detailed issue description
          example: "User unable to connect to VPN from MacBook Pro. Error message: 'Connection timeout'"
        issuetype:
          type: object
          properties:
            name:
              type: string
              enum: [Bug, Task, Story, Support Request]
              description: Type of issue
              default: Support Request
        priority:
          type: object
          properties:
            name:
              type: string
              enum: [Low, Medium, High, Critical]
              default: Medium
        assignee:
          type: object
          properties:
            accountId:
              type: string
              description: Jira user account ID

paths:
  /issue:
    post:
      summary: Create Jira issue
      description: Create a new issue in Jira
      operationId: createIssue
      security:
        - BearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - fields
              properties:
                fields:
                  $ref: '#/components/schemas/IssueFields'
            examples:
              supportRequest:
                summary: Example support request
                value:
                  fields:
                    project:
                      key: SUPPORT
                    summary: "VPN connection fails"
                    description: "Detailed issue description"
                    issuetype:
                      name: "Support Request"
                    priority:
                      name: "High"
      responses:
        '201':
          description: Issue created successfully
          content:
            application/json:
              schema:
                type: object
                properties:
                  id:
                    type: string
                    example: "10001"
                  key:
                    type: string
                    example: "SUPPORT-123"
                  self:
                    type: string
                    example: "https://company.atlassian.net/rest/api/3/issue/10001"
        '400':
          description: Invalid request
          content:
            application/json:
              schema:
                type: object
                properties:
                  errorMessages:
                    type: array
                    items:
                      type: string
        '401':
          description: Unauthorized
        '403':
          description: Forbidden
```

### Authentication Configuration

**OAuth 2.0 Setup:**

**1. Register app in Atlassian:**
- Navigate to https://developer.atlassian.com/apps
- Create OAuth 2.0 integration
- Set callback URL: `https://global.consent.azure-apim.net/redirect`
- Note Client ID and generate Client Secret

**2. Configure scopes:**
- `write:jira-work` - Create and edit issues
- `read:jira-work` - Read issues

**3. CoPilot Studio connector configuration:**
```yaml
connector_name: jira-ticket-creator
auth_type: OAuth 2.0
authorization_url: https://auth.atlassian.com/authorize
token_url: https://auth.atlassian.com/oauth/token
client_id: {from step 1}
client_secret: {stored in Azure Key Vault}
scopes: write:jira-work read:jira-work
```

**4. Secret storage:**
```bash
# Store in Azure Key Vault
az keyvault secret set \
  --vault-name company-secrets-prod \
  --name jira-client-secret \
  --value {secret_from_step_1}
```

**5. Reference in Power Platform:**
- Environment variables > New
- Name: JiraClientSecret
- Type: Secret
- Value: Reference to Key Vault secret

### CoPilot Topic Integration

**Topic: Create Support Ticket**

```
1. Trigger: "create ticket", "open issue", "report problem"

2. System: "I'll help you create a support ticket. Let's gather some information."

3. Ask: "What's the issue?"
   Entity: FreeText (multi-line)
   Store: issue_summary

4. Ask: "Can you provide more details?"
   Entity: FreeText (multi-line)
   Store: issue_description

5. Ask: "What's the priority?"
   Options: ["Low", "Medium", "High", "Critical"]
   Store: issue_priority

6. Confirm: "Create ticket with:
   - Summary: {issue_summary}
   - Priority: {issue_priority}

   Should I create this ticket?"
   Options: ["Yes", "No", "Edit"]

7. If Yes:
     Action: createIssue (Jira API)
       Input:
         fields:
           project:
             key: "SUPPORT"
           summary: {issue_summary}
           description: {issue_description}
           issuetype:
             name: "Support Request"
           priority:
             name: {issue_priority}
     Store: ticket_response

8. System: "Ticket created! 🎫

   Ticket ID: {ticket_response.key}
   Track your ticket: {ticket_response.self}

   An agent will respond within [SLA time based on priority]."

9. Action: Send notification (Power Automate)
     - Send Teams message to support channel
     - Include ticket ID and priority
```

### Error Handling

```
If createIssue action fails:
  Log error to Application Insights

  If error = 401 (unauthorized):
    System: "Authentication failed. Please contact IT to refresh your Jira access."

  Else if error = 400 (invalid request):
    System: "Unable to create ticket due to invalid information. Please try again or contact support@company.com."

  Else:
    System: "Unable to create ticket. The ticketing system is temporarily unavailable. Please try again in a few minutes or email support@company.com."

  Optionally: Escalate to human agent
```

---

## Pattern 3: Multi-Turn Expense Approval

### Use Case
Guided workflow for expense report submission with approval

### Topic Flow

**Topic: Submit Expense Report**

```
1. Trigger: "submit expense", "expense report", "reimbursement"

2. System: "I'll help you submit an expense report. This will take about 2 minutes."

3. Ask: "What's the total amount?"
   Entity: Currency
   Validation: >0, <10000 (max reimbursable amount)
   Store: expense_amount

4. Ask: "What category is this expense?"
   Options: ["Travel", "Meals", "Lodging", "Supplies", "Other"]
   Store: expense_category

5. Ask: "What's the business purpose?"
   Entity: FreeText (single line, max 200 chars)
   Store: business_purpose

6. If expense_category = "Travel" OR "Lodging":
     Ask: "Where did you travel?"
     Entity: Location (city, state/country)
     Store: travel_destination

7. Ask: "Please upload your receipt."
   Entity: Attachment
   Validation:
     - File types: PDF, PNG, JPG
     - Max size: 5MB
   Store: receipt_file

8. Confirm: "Review your expense:

   Amount: ${expense_amount}
   Category: {expense_category}
   Purpose: {business_purpose}
   Destination: {travel_destination} [if applicable]
   Receipt: {receipt_file.name}

   Submit for approval?"
   Options: ["Submit", "Edit", "Cancel"]

9. If Submit:
     Action: Power Automate flow "Submit Expense"
       Inputs:
         - employee_email: {user.email}
         - amount: {expense_amount}
         - category: {expense_category}
         - purpose: {business_purpose}
         - destination: {travel_destination}
         - receipt: {receipt_file}
       Outputs:
         - expense_id
         - approval_url
         - manager_name

10. System: "Expense submitted! ✅

    Expense ID: {expense_id}
    Amount: ${expense_amount}
    Submitted to: {manager_name}

    Your manager will review within 2 business days.
    Track status: {approval_url}

    You'll receive an email when approved."
```

### Power Automate Flow: Submit Expense

```
1. Trigger: CoPilot action called
   Inputs:
     - employee_email
     - amount
     - category
     - purpose
     - destination
     - receipt (file)

2. Get employee details (Azure AD)
   - Manager email
   - Department
   - Cost center

3. Upload receipt to SharePoint
   - Site: Finance
   - Library: Expense Receipts
   - Folder: /{year}/{month}/{employee_email}/
   - File: {expense_id}_{receipt_filename}
   - Store: receipt_url

4. Create expense record (Dynamics 365 or Dataverse)
   - Table: Expense
   - Fields:
     - Employee: {employee_email}
     - Amount: {amount}
     - Category: {category}
     - Purpose: {purpose}
     - Destination: {destination}
     - Receipt URL: {receipt_url}
     - Status: Pending Approval
     - Submitted Date: {utcNow()}
   - Store: expense_id

5. Check if manager approval required
   Condition: amount > 500 OR category = "Travel"
     ↓ Yes: Continue to step 6
     ↓ No: Auto-approve, skip to step 8

6. Create approval (Approvals connector)
   - Title: "Expense Approval: ${amount}"
   - Assigned to: {manager_email}
   - Details:
     - Employee: {employee_email}
     - Amount: ${amount}
     - Category: {category}
     - Purpose: {purpose}
     - Receipt: {receipt_url}
   - Store: approval_id

7. Send notification to manager (Teams)
   - Channel: Manager's personal chat
   - Message: "New expense approval request from {employee_email}:
               Amount: ${amount}
               Category: {category}
               [Approve] [Reject] [View Details]"
   - Adaptive card with action buttons

8. Return to CoPilot:
   - expense_id: {expense_id}
   - approval_url: {approval_portal_url}/expense/{expense_id}
   - manager_name: {manager_name}

9. Log event (Application Insights)
   - Event: Expense_Submitted
   - Properties:
     - employee: {employee_email}
     - amount: {amount}
     - category: {category}
     - requires_approval: {true/false}
```

### Approval Flow (Separate Power Automate Flow)

```
1. Trigger: Approval response received
   - Approval ID: {from step 6 above}

2. Get approval details
   - Response: Approved / Rejected
   - Comments: {optional manager comments}

3. Update expense record (Dataverse)
   - Expense ID: {expense_id}
   - Status: {Approved / Rejected}
   - Approved By: {manager_email}
   - Approved Date: {utcNow()}
   - Comments: {comments}

4. If Approved:
     - Send to payroll system (API call)
     - Email employee: "Expense approved! Reimbursement in next payroll."

   If Rejected:
     - Email employee: "Expense rejected. Manager comments: {comments}"

5. Send Teams notification to employee
   - Adaptive card with status, details, next steps

6. Log event (Application Insights)
   - Event: Expense_Approved or Expense_Rejected
   - Properties:
     - expense_id
     - manager
     - amount
     - processing_time (days from submission)
```

### Entity Definitions

**Currency entity:**
```yaml
entity_name: Currency
type: prebuilt (currency)
resolution:
  - Extract numeric value
  - Handle formats: $1,234.56 or 1234.56 or "one thousand two hundred thirty four dollars"
validation:
  - Must be > 0
  - Must be <= 10000 (company policy: max $10k per expense)
```

**Attachment entity:**
```yaml
entity_name: Attachment
type: prebuilt (file)
validation:
  - file_types: [".pdf", ".png", ".jpg", ".jpeg"]
  - max_size: 5242880 (5MB in bytes)
  - virus_scan: true (if available)
error_messages:
  invalid_type: "Please upload a PDF, PNG, or JPG file."
  too_large: "File too large. Maximum size is 5MB."
  scan_failed: "File failed security scan. Please contact IT."
```

### Example Conversation

```
User: "I need to submit an expense report"

Agent: "I'll help you submit an expense report. This will take about 2 minutes. What's the total amount?"

User: "$450"

Agent: "What category is this expense?"
[Shows buttons: Travel, Meals, Lodging, Supplies, Other]

User: [Clicks "Meals"]

Agent: "What's the business purpose?"

User: "Client dinner with ABC Corp"

Agent: "Please upload your receipt."

User: [Uploads receipt.pdf]

Agent: "Review your expense:

Amount: $450.00
Category: Meals
Purpose: Client dinner with ABC Corp
Receipt: receipt.pdf

Submit for approval?"
[Shows buttons: Submit, Edit, Cancel]

User: [Clicks "Submit"]

Agent: [Processing...]

Agent: "Expense submitted! ✅

Expense ID: EXP-2026-00123
Amount: $450.00
Submitted to: Jane Smith (your manager)

Your manager will review within 2 business days.
Track status: https://expenses.company.com/EXP-2026-00123

You'll receive an email when approved."

User: "Thanks!"

Agent: "You're welcome! Let me know if you need anything else."
```

---

## Common Patterns Across Examples

### Pattern: Guided Data Collection
1. Greet user, set expectations
2. Ask questions sequentially
3. Validate each input
4. Store in variables
5. Confirm before action
6. Execute action
7. Confirm completion

### Pattern: Error Handling
- Validate inputs at each step
- Provide helpful error messages
- Offer retry or alternative path
- Escalate to human if needed
- Log errors for debugging

### Pattern: User Experience
- Set expectations upfront ("This will take 2 minutes")
- Use clear, concise language
- Provide options with buttons (not free text when possible)
- Confirm before irreversible actions
- Provide next steps after completion
- Offer help if user seems stuck

### Pattern: Integration
- Use Power Automate for complex workflows
- Store data in Dataverse or external systems
- Send notifications (Teams, email)
- Log events for analytics
- Return structured results to CoPilot
