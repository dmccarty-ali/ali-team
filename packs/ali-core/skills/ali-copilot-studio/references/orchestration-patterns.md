# ali-copilot-studio - Orchestration Patterns Reference

This reference provides detailed orchestration and conversation flow patterns for Microsoft CoPilot Studio.

---

## Built-in Orchestration Features

CoPilot Studio provides automatic orchestration capabilities:

### Intent Recognition
- Matches user input to topics based on trigger phrases
- Uses natural language understanding (NLU) to interpret intent
- Handles variations and synonyms automatically
- Supports multi-language intent detection

### Context Management
- Maintains conversation state across turns
- Remembers user inputs from previous messages
- Tracks entity values collected during conversation
- Persists context within session boundaries

### Entity Extraction
- Pulls structured data from unstructured user input
- Recognizes built-in entities (email, phone, date, currency)
- Supports custom entity types
- Validates extracted values against schema

### Fallback Handling
- Redirects unknown queries to knowledge sources
- Offers suggested topics when intent unclear
- Escalates to human agent when appropriate
- Provides graceful degradation path

### Multi-Turn Conversations
- Handles clarification questions
- Collects required information step-by-step
- Confirms user intent before actions
- Supports branching conversation flows

### Session State
- Remembers context within active session
- Clears state when session ends (timeout or explicit end)
- Supports custom variables for tracking
- Allows state export/import for handoffs

---

## Topic Design Patterns

### Pattern 1: Guided Workflow

**Use case**: Multi-step processes requiring structured data collection

**Example: Expense Report Submission**

```
Topic: Expense Report Submission

1. Trigger: "submit expense", "expense report"

2. System: "Let me help you submit an expense report."

3. Ask: "What's the total amount?"
   Entity: Currency (validates number format)
   Store: expense_amount

4. Ask: "What category is this expense?"
   Options: ["Travel", "Meals", "Supplies", "Other"]
   Entity: ExpenseCategory
   Store: expense_category

5. Ask: "Please upload your receipt."
   Entity: Attachment (validates file type: PDF, PNG, JPG)
   Store: receipt_file

6. Confirm: "Submit expense for ${expense_amount} in category {expense_category}?"
   Options: ["Submit", "Cancel"]

7. If Submit:
     Action: Power Automate flow
       - Upload receipt to SharePoint
       - Create expense record in Dynamics
       - Send approval email to manager
     Return: expense_id, approval_url

8. System: "Expense #{expense_id} submitted. Your manager will review within 2 business days. Track status: {approval_url}"
```

**Key characteristics**:
- Sequential data collection
- Validation at each step
- Confirmation before irreversible action
- Structured output for downstream systems

---

### Pattern 2: Knowledge-Grounded Q&A

**Use case**: Answer questions using knowledge sources

**Example: Product Questions**

```
Topic: Product Questions

1. Trigger: "how do I", "what is", "tell me about"

2. Query: Knowledge source (product documentation)
   - Search based on user query
   - Retrieve top 3 most relevant passages
   - Rank by relevance score

3. Return: Grounded answer with source citation
   "Based on the Product User Guide, here's how to [task]:
    [Answer text]

    Source: Product User Guide, Section 3.2"

4. Offer: "Was this helpful?" (feedback loop)
   Options: ["Yes", "No", "I need more help"]

5. If "No" or "I need more help":
   - Escalate to Pattern 3 (human agent)
```

**Key characteristics**:
- Knowledge source integration
- Source attribution
- Feedback collection
- Escalation path

---

### Pattern 3: Escalation to Human

**Use case**: Handoff to human agent when automation insufficient

**Example: Complex Issue Escalation**

```
Topic: Escalate to Human

1. Trigger: "talk to human", "agent", "help", "speak to someone"

2. System: "I'll connect you with a support agent. To help them assist you faster, can you briefly describe the issue?"

3. Collect: Issue description
   Entity: FreeText (multi-line input)
   Store: issue_description

4. Ask: "What's the best contact method?"
   Options: ["Email", "Phone", "Teams chat"]
   Store: contact_method

5. Action: Power Automate flow
   - Create support ticket in ServiceNow
   - Assign to available agent
   - Send notification to support team via Teams
   Return: ticket_id, estimated_wait_time

6. If contact_method = "Teams chat":
     Initiate: Live agent handoff
     Transfer: Conversation context + issue_description
   Else:
     System: "Ticket #{ticket_id} created. An agent will {contact_method} you within {estimated_wait_time}."
```

**Key characteristics**:
- Graceful handoff with context
- User preference for contact method
- Ticket creation for tracking
- SLA communication

---

## Advanced Orchestration Patterns

### Pattern 4: Conditional Branching

**Use case**: Different flows based on user attributes or responses

```
Topic: IT Support Request

1. Trigger: "need help with", "IT support"

2. Check: User department (from Azure AD)
   If department = "Engineering":
     - Offer self-service options first (password reset, VPN)
   Else:
     - Escalate to IT support immediately

3. Ask: "What do you need help with?"
   Dynamic options based on department:
   - Engineering: ["VPN", "Dev tools", "Build issues"]
   - Sales: ["CRM access", "Email", "Laptop setup"]
   - HR: ["Benefits portal", "Payroll", "Onboarding"]
```

---

### Pattern 5: Multi-Agent Orchestration

**Use case**: Different specialized agents for different domains

```
Main Agent: Company Assistant

User: "I need to submit an expense and check my PTO balance"

Orchestration:
1. Parse: Two intents detected
   - Submit expense (Finance Agent)
   - Check PTO (HR Agent)

2. Sequence:
   Step 1: Hand off to Finance Agent
     - Complete expense submission
   Step 2: Hand off to HR Agent
     - Retrieve PTO balance

3. Synthesize: Combine results
   "Done! Expense #{expense_id} submitted. Your PTO balance is 15 days."
```

---

### Pattern 6: Proactive Notifications

**Use case**: Agent initiates conversation based on event

```
Trigger: System event (Power Automate trigger)
  - New security alert
  - Approval pending
  - Scheduled reminder

Flow:
1. Event detected: Security alert fired in Azure Security Center

2. Action: Send proactive Teams message
   "⚠️ Security Alert: Unusual login detected for your account from IP 203.0.113.42 (Moscow, Russia).

   Was this you?"
   Options: ["Yes, it was me", "No, not me"]

3. If "No, not me":
   - Trigger: Password reset flow
   - Escalate: Security team notified
   - Action: Account temporarily locked
```

---

## State Management Patterns

### Session Variables

```
Variables:
  - user_email: Extracted from Azure AD
  - selected_department: User choice
  - items_in_cart: Array of item IDs
  - approval_pending: Boolean flag

Usage:
  - Store: user_email = {user.email}
  - Retrieve: "Your email is {user_email}"
  - Condition: If approval_pending = true
  - Increment: items_in_cart.add({item_id})
```

### Context Passing Between Topics

```
Topic A: Product Selection
  - User selects: product_id = "LAPTOP-001"
  - Redirect to: Topic B (Checkout)
  - Pass context: {product_id, quantity, user_email}

Topic B: Checkout
  - Receive context: product_id, quantity, user_email
  - Fetch: product_details from API
  - Continue: Payment collection
```

---

## Debugging Conversation Flows

### Test Canvas Commands

```powershell
# Test topic in Test Canvas
Test-CopilotTopic -Topic "Password Reset" -Input "I forgot my password"

# View conversation transcript
Get-ConversationTranscript -SessionId {guid}

# Check action execution logs
Get-ActionLogs -ActionName "CreateTicket" -TimeRange "Last24Hours"
```

### Common Flow Issues

| Issue | Symptom | Solution |
|-------|---------|----------|
| **Topic not triggering** | Agent doesn't respond | Add more trigger variations |
| **Entity not extracted** | Agent asks repeatedly | Improve entity definition, add examples |
| **Action fails silently** | No error shown to user | Add error handling, return failure message |
| **Infinite loop** | Conversation stuck | Add loop counter, exit condition |
| **Context lost** | Agent forgets previous inputs | Store in session variables |

---

## Performance Optimization

### Best Practices

1. **Minimize API calls**: Cache results when possible
2. **Use async actions**: Don't block conversation flow
3. **Limit topic depth**: Max 10 steps per topic
4. **Optimize knowledge search**: Use specific queries, not broad searches
5. **Prune old sessions**: Set appropriate timeout (30 min idle)

### Metrics to Monitor

- **Average conversation length**: Number of turns
- **Topic completion rate**: % of users completing flow
- **Escalation rate**: % of conversations requiring human
- **Response time**: Time from user input to agent response
- **Action success rate**: % of actions completing successfully
