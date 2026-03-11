# ali-copilot-studio - Actions and Plugins Reference

This reference provides detailed implementation patterns for API plugins and Power Automate actions in Microsoft CoPilot Studio.

---

## API Plugins

### Purpose
Expose APIs as callable actions within agent conversations

### OpenAPI Specification Pattern

```yaml
# OpenAPI specification
openapi: 3.0.0
info:
  title: Ticket System API
  version: 1.0.0
paths:
  /tickets:
    post:
      summary: Create support ticket
      operationId: createTicket
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                title:
                  type: string
                  description: Ticket title
                priority:
                  type: string
                  enum: [low, medium, high]
                  description: Ticket priority
      responses:
        '200':
          description: Ticket created
          content:
            application/json:
              schema:
                type: object
                properties:
                  ticket_id:
                    type: string
```

### Security Requirements

- **Authentication header**: API key or OAuth token
- **Input validation**: Validate all parameters before processing
- **Rate limiting**: Protect against abuse
- **Error handling**: Return safe error messages (no stack traces or internal details)

### Complete Example: Jira Ticket API

```yaml
openapi: 3.0.0
info:
  title: Jira Ticket API
  version: 1.0.0
servers:
  - url: https://company.atlassian.net/rest/api/3
components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
paths:
  /issue:
    post:
      summary: Create Jira issue
      operationId: createIssue
      security:
        - BearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                fields:
                  type: object
                  properties:
                    project:
                      type: object
                      properties:
                        key:
                          type: string
                          default: SUPPORT
                    summary:
                      type: string
                    description:
                      type: string
                    issuetype:
                      type: object
                      properties:
                        name:
                          type: string
                          enum: [Bug, Task, Story]
      responses:
        '201':
          description: Issue created
```

**Connector configuration:**
- Auth type: OAuth 2.0
- Token endpoint: https://auth.atlassian.com/oauth/token
- Scopes: write:jira-work
- Secret storage: Azure Key Vault

---

## Power Automate Actions

### Purpose
Integrate with Microsoft 365, Dynamics, Azure services

### Common Patterns

| Use Case | Power Automate Flow |
|----------|---------------------|
| Send notifications | Email, Teams message, SMS |
| Create records | SharePoint list item, Dynamics record |
| Approval workflows | Manager approval, multi-step reviews |
| File operations | Upload to OneDrive, SharePoint document library |
| Data retrieval | Fetch from Dataverse, SQL database |

### Example Flow: Ticket Creation with Conditional Notification

```
Trigger: CoPilot action called
↓
Input: ticket_title, priority, user_email
↓
Action: Create SharePoint list item
↓
Condition: If priority = "high"
  ↓ Yes: Send Teams notification to support channel
  ↓ No: Email notification to support queue
↓
Return: ticket_id, created_timestamp
```

### Example Action: File Upload to SharePoint

```yaml
action: upload_to_sharepoint
parameters:
  site_url: https://company.sharepoint.com/sites/docs
  library: Shared Documents
  file: {user_upload}
  folder: /Expenses/2026
```

### Example Action: Send Teams Notification

```yaml
action: send_teams_message
parameters:
  team: IT Support
  channel: Urgent
  message: "New ticket #{ticket_id}: {issue_summary}"
```

### Example Action: Query Dataverse

```yaml
action: query_dataverse
parameters:
  table: account
  filter: "accountnumber eq '{account_num}'"
  select: ["name", "address1_city", "telephone1"]
```

---

## Best Practices

### API Plugin Design
1. **Use descriptive operation IDs**: Makes actions discoverable
2. **Provide default values**: Reduces user input burden
3. **Use enums for constrained choices**: Prevents invalid inputs
4. **Include examples in schema**: Helps AI understand expected format
5. **Version your APIs**: Enables backward compatibility

### Power Automate Design
1. **Handle failures gracefully**: Add error handling to all actions
2. **Return structured data**: JSON format preferred over plain text
3. **Log action executions**: For debugging and audit
4. **Test with edge cases**: Null inputs, special characters, large files
5. **Implement timeouts**: Prevent long-running flows from blocking conversation

### Security
1. **Never hardcode credentials**: Use connectors and Key Vault
2. **Validate all inputs**: Prevent injection attacks
3. **Use least-privilege scopes**: Only request necessary permissions
4. **Rotate secrets regularly**: API keys, client secrets
5. **Monitor API usage**: Detect anomalies and abuse
