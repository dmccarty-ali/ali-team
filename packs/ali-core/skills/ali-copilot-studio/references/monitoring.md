# ali-copilot-studio - Monitoring Reference

This reference provides detailed monitoring and analytics patterns for Microsoft CoPilot Studio.

---

## Built-in Analytics

### Available Metrics

**Conversation Metrics:**
- **Total conversations**: Count of all conversations started
- **Completed conversations**: Conversations that reached a resolution
- **Abandoned conversations**: Conversations where user stopped responding
- **Average conversation length**: Number of turns per conversation
- **Conversation duration**: Time from start to completion

**User Metrics:**
- **Active users (daily)**: Unique users who started conversations today
- **Active users (weekly)**: Unique users who started conversations this week
- **Active users (monthly)**: Unique users who started conversations this month
- **New users**: First-time users this period
- **Returning users**: Users with multiple conversations

**Engagement Metrics:**
- **Topic engagement**: Which topics are most frequently triggered
- **Topic completion rate**: % of topics that reach completion
- **Escalation rate**: % of conversations escalated to human
- **Knowledge source hit rate**: % of queries answered from knowledge
- **Fallback rate**: % of queries that triggered fallback handler

**Action Metrics:**
- **Action invocations**: Count of each action called
- **Action success rate**: % of actions completed successfully
- **Action failure rate**: % of actions that failed
- **Average action duration**: Time to execute action

**User Satisfaction:**
- **CSAT score**: Customer satisfaction rating (if feedback enabled)
- **Thumbs up/down**: Positive vs. negative feedback count
- **Feedback comments**: Qualitative user feedback

---

## Custom Telemetry

### Implementation Pattern

```
Agent conversation
↓
At each key event:
  - Topic triggered
  - Entity extracted
  - Action invoked
  - Escalation occurred
↓
Log to Application Insights
↓
Custom properties:
  - user_department: "Engineering"
  - user_location: "Seattle"
  - topic_triggered: "Password Reset"
  - escalation_reason: "VPN configuration"
  - resolution_time: 120 (seconds)
  - user_satisfaction: 5 (out of 5)
↓
Power BI dashboard for executive reporting
```

### Azure Application Insights Integration

**Setup:**
1. Create Application Insights resource
2. Copy instrumentation key
3. Configure in CoPilot Studio settings
4. Enable custom telemetry

**Custom event logging (Power Automate):**
```yaml
action: log_custom_event
parameters:
  eventName: "Expense_Submitted"
  properties:
    user_id: "{user.email}"
    amount: "{expense_amount}"
    category: "{expense_category}"
    approval_required: true
  measurements:
    amount_usd: 150.00
    processing_time_ms: 342
```

**Query in Application Insights:**
```kql
customEvents
| where name == "Expense_Submitted"
| where timestamp > ago(7d)
| summarize
    count(),
    avg(todouble(customMeasurements.amount_usd)),
    avg(todouble(customMeasurements.processing_time_ms))
  by tostring(customDimensions.category)
| order by count_ desc
```

---

## Error Handling and Logging

### Log Categories

**1. User input (sanitized)**
- Original message (PII redacted)
- Intent detected
- Entities extracted

**2. Topic triggers**
- Which topic matched
- Confidence score
- Fallback triggered (if any)

**3. Action executions**
- Action name
- Input parameters (sanitized)
- Start time, end time, duration
- Success/failure status

**4. API call results**
- Endpoint called
- HTTP status code
- Response time
- Error message (if failed)

**5. Error messages**
- Error type (validation, network, timeout, etc.)
- Error message (safe, no sensitive data)
- Stack trace (if dev environment)
- Remediation suggestion

**6. Escalations**
- Reason for escalation
- User input leading to escalation
- Support ticket created (if applicable)

### Log Structure (JSON)

```json
{
  "timestamp": "2026-02-16T14:35:22Z",
  "conversation_id": "conv-abc-123",
  "user_id": "user@company.com",
  "event_type": "action_execution",
  "action_name": "create_ticket",
  "status": "success",
  "duration_ms": 234,
  "properties": {
    "ticket_id": "TICKET-12345",
    "priority": "high",
    "category": "IT Support"
  },
  "metadata": {
    "environment": "production",
    "agent_version": "1.2.0",
    "channel": "teams"
  }
}
```

---

## Log Retention

### Retention Policies

| Log Type | Standard | Extended | Long-Term Archive |
|----------|----------|----------|-------------------|
| **Conversation logs** | 30 days | 90 days | N/A |
| **Audit logs** | 90 days | 1 year | 7 years (compliance) |
| **Error logs** | 90 days | 1 year | N/A |
| **Custom telemetry** | 90 days | 1 year | N/A |

**Standard**: Included with CoPilot Studio license

**Extended**: Premium license or Azure Log Analytics integration

**Long-Term Archive**: Export to Azure Storage (cool/archive tier)

### Export to Azure Storage

**Power Automate flow (daily):**
```
Trigger: Scheduled (daily 2 AM)
↓
Action: Get conversation logs (CoPilot API)
  - Filter: timestamp > yesterday
↓
Action: Transform to JSONL
↓
Action: Upload to Azure Blob Storage
  - Container: copilot-logs
  - Path: /logs/{year}/{month}/{day}/conversations.jsonl
↓
Action: Log export completion
```

---

## Dashboards and Reporting

### Executive Dashboard (Power BI)

**Metrics included:**
- **Total conversations** (trend over time)
- **Active users** (DAU, WAU, MAU)
- **Top topics** (by volume)
- **Escalation rate** (trend)
- **User satisfaction** (CSAT score, feedback)
- **Action success rate** (by action)

**Visual examples:**
- Line chart: Daily active users (last 30 days)
- Bar chart: Top 10 topics by volume
- Pie chart: Escalation reasons
- Gauge: Overall CSAT score
- Table: Action performance (name, invocations, success rate)

### Operational Dashboard (Azure Monitor)

**Metrics included:**
- **Error rate** (errors per hour)
- **Response time** (p50, p95, p99)
- **API failures** (count by endpoint)
- **Concurrent conversations** (real-time)
- **Content filter violations** (count by type)

**Alerts configured:**
- Error rate >5% for 10 minutes → Notify engineering
- Response time p95 >10 seconds → Notify infrastructure
- API failure rate >10% → Notify on-call
- Content violations >50/hour → Notify security

### Compliance Audit Report (Monthly)

**Sections:**
- **Conversation volume**: Total, by department, by channel
- **Security events**: Authentication failures, content violations, PII detections
- **Access reviews**: Who accessed what, when
- **Data retention**: Logs purged per policy
- **Incidents**: Security incidents, resolution status
- **Compliance posture**: GDPR/HIPAA/SOC2 requirements met

---

## Real-Time Monitoring

### Conversation Monitoring

**Live conversation view:**
- Active conversations (real-time)
- Current topic for each conversation
- User waiting time
- Actions in progress

**Use case**: Support team monitoring for intervention opportunities

**Implementation**: Power BI streaming dataset + real-time dashboard

### Performance Monitoring

**Metrics:**
- **Response time**: Time from user message to agent response
- **Action execution time**: Time for each action to complete
- **Queue depth**: Conversations waiting for processing

**Alerts:**
- Response time >5 seconds → Check infrastructure
- Action timeout → Investigate external API
- Queue depth >100 → Scale up compute resources

### Error Spike Detection

**Pattern:**
```
Monitor error rate (rolling 5-minute window)
↓
If error rate >10% AND previous 5 min was <5%:
  → Spike detected
  → Send alert to on-call engineer
  → Include: error types, affected users, sample error messages
```

---

## User Feedback Collection

### In-Conversation Feedback

**Pattern:**
```
Agent provides answer
↓
Ask: "Was this helpful?"
Options: ["Yes", "No", "Partially"]
↓
If "No" or "Partially":
  Ask: "What could be improved?" (free text)
↓
Log feedback:
  - conversation_id
  - helpful: false
  - comment: "Answer was too technical"
↓
Review weekly for improvement opportunities
```

### Post-Conversation Survey

**Triggered:** After conversation ends (topic completed or escalated)

**Questions:**
1. Overall satisfaction (1-5 stars)
2. Agent was helpful (Yes/No)
3. Issue resolved (Yes/No)
4. What could be better? (free text)

**Power Automate flow:**
```
Conversation ends
↓
Wait 5 minutes (allow user to finish)
↓
Send survey (Teams adaptive card or email)
↓
Collect responses
↓
Log to Application Insights
```

---

## Alerting Strategy

### Alert Severity Levels

| Severity | Response Time | Notification Method | Example |
|----------|---------------|---------------------|---------|
| **Critical** | Immediate | Phone, SMS, email | Agent completely down, auth failure |
| **High** | 15 minutes | Email, Teams | Error rate >20%, API dependency down |
| **Medium** | 1 hour | Email | Error rate 10-20%, slow response time |
| **Low** | 24 hours | Email | Escalation rate increased, topic underperforming |

### Alert Configuration

**Azure Monitor Alert Rules:**

**1. High error rate**
```yaml
condition:
  metric: ErrorRate
  threshold: 10%
  window: 5 minutes
  frequency: 1 minute
action:
  severity: High
  notify: engineering-team@company.com
  message: "CoPilot error rate exceeded 10% in last 5 minutes"
```

**2. Response time degradation**
```yaml
condition:
  metric: ResponseTimeP95
  threshold: 10 seconds
  window: 10 minutes
  frequency: 5 minutes
action:
  severity: Medium
  notify: infrastructure-team@company.com
  message: "CoPilot p95 response time >10s for 10 minutes"
```

**3. External API failure**
```yaml
condition:
  metric: APIFailureRate
  threshold: 50%
  window: 5 minutes
  frequency: 1 minute
action:
  severity: High
  notify: on-call-engineer
  message: "CoPilot external API failure rate >50%"
```

---

## Performance Optimization

### Response Time Analysis

**Breakdown:**
- **Intent recognition**: Time to match user input to topic
- **Knowledge retrieval**: Time to search knowledge sources
- **Action execution**: Time for external API calls
- **Response generation**: Time to formulate agent response

**Optimization targets:**
- Intent recognition: <500ms
- Knowledge retrieval: <1s
- Action execution: <3s (depends on external API)
- Response generation: <500ms

**Total target: <5 seconds for 95th percentile**

### Bottleneck Identification

**Query Application Insights:**
```kql
customEvents
| where name == "ActionExecution"
| summarize
    count(),
    avg(todouble(customMeasurements.duration_ms)),
    percentile(todouble(customMeasurements.duration_ms), 95)
  by tostring(customDimensions.action_name)
| where percentile_customMeasurements_duration_ms_95 > 3000
| order by percentile_customMeasurements_duration_ms_95 desc
```

**Actions with p95 >3 seconds need optimization:**
- Cache API responses if data doesn't change frequently
- Use async processing for long-running operations
- Optimize external API query (indices, filters)
- Consider pre-computing results

---

## Continuous Improvement

### Weekly Review Process

**1. Review metrics**
- Conversation volume trend
- Top topics (identify patterns)
- Escalation reasons (opportunities for automation)
- Low satisfaction scores (identify failures)

**2. Analyze feedback**
- Common complaints or suggestions
- Features requested
- Confusing interactions

**3. Identify improvements**
- Add new topics for common escalations
- Improve knowledge sources (gaps in documentation)
- Optimize slow actions
- Fix confusing conversation flows

**4. Prioritize and implement**
- High impact, low effort first
- Iterate in 2-week sprints

### A/B Testing

**Pattern:**
```
Split traffic: 50% variant A, 50% variant B
↓
Variant A: Current welcome message
Variant B: New welcome message with suggested actions
↓
Run for 2 weeks
↓
Compare metrics:
  - Conversation completion rate
  - Average conversation length
  - User satisfaction
↓
Choose winner based on data
↓
Roll out winning variant to 100%
```

**Tools**: CoPilot Studio built-in A/B testing (if available) or custom implementation with Power Automate

---

## Troubleshooting Common Issues

### High Escalation Rate

**Symptom**: >30% of conversations escalate to human

**Investigate:**
- Which topics escalate most?
- What triggers escalations? (complexity, missing information)
- Are knowledge sources incomplete?

**Solutions:**
- Improve knowledge sources for top escalation topics
- Add topics for common escalation reasons
- Clarify agent capabilities (set user expectations)

### Low Completion Rate

**Symptom**: <50% of conversations reach completion

**Investigate:**
- Where in the flow do users abandon?
- Are questions too complex?
- Is agent asking for too much information?

**Solutions:**
- Simplify flows (fewer steps)
- Provide default values where possible
- Allow users to skip optional questions
- Improve error messages (guide users)

### Slow Response Times

**Symptom**: p95 response time >10 seconds

**Investigate:**
- Bottleneck analysis (which step is slow?)
- External API performance
- Knowledge source search performance

**Solutions:**
- Cache frequent API responses
- Optimize knowledge source indices
- Use async processing for long operations
- Scale infrastructure (if compute-bound)

---

## Best Practices Summary

### Monitoring
1. Enable built-in analytics
2. Configure custom telemetry for business metrics
3. Set up alerting for critical issues
4. Review metrics weekly

### Logging
1. Log all critical events (actions, errors, escalations)
2. Sanitize logs (no PII)
3. Retain logs per compliance requirements
4. Export to long-term storage for audit

### Feedback
1. Collect in-conversation feedback
2. Send post-conversation surveys
3. Analyze feedback regularly
4. Act on improvement opportunities

### Optimization
1. Monitor performance metrics
2. Identify bottlenecks
3. Optimize slow components
4. A/B test improvements
