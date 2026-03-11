# ali-copilot-studio - Migration Details Reference

This reference provides detailed migration patterns from Claude Code to Microsoft CoPilot Studio.

---

## Comparison: Claude Code vs CoPilot Studio

### Feature Mapping

| Feature | Claude Code | CoPilot Studio | Migration Notes |
|---------|-------------|----------------|-----------------|
| **Agent definition** | Markdown files (*.md) | Visual designer + JSON manifest | Convert MD frontmatter to JSON |
| **Skills** | SKILL.md files | Knowledge sources + custom topics | Export skills to SharePoint or file upload |
| **Tools** | Read, Write, Bash, Grep, Glob, etc. | API plugins, Power Automate flows | Map tools to actions |
| **Context management** | File-based, manual | Automatic conversation state | Leverage built-in orchestration |
| **Deployment** | CLI, local or cloud | Teams, web, mobile channels | Multi-channel deployment |
| **Authentication** | Environment variables, config files | OAuth, managed identities, API keys | Use Power Platform connectors |
| **Orchestration** | Agent spawning (Task tool) | Built-in topic routing | Convert Task calls to topic redirects |
| **Error handling** | Try-catch in code | Topic-level error handling, fallbacks | Define fallback topics |
| **Permissions** | File system, CLI permissions | Role-based access control (RBAC) | Map to Azure AD groups |
| **Versioning** | Git-based | Solution versioning, environments | Use ALM toolkit |
| **Testing** | Manual, unit tests | Test canvas, UAT in staging | Leverage built-in test tools |

---

## Migration Strategy - Detailed Steps

### Phase 1: Assessment (1-2 weeks)

**Objective**: Understand what needs to be migrated and how

**Tasks:**

**1.1 Inventory Claude Code agents**
- List all agent files (*.md in agents/ directory)
- Document agent purposes, tools used, dependencies
- Identify custom vs. standard agents

**Example inventory:**
```
| Agent | Tools Used | External Dependencies | Priority |
|-------|------------|----------------------|----------|
| backend-developer | Read, Write, Bash | Git, npm, pytest | High |
| security-expert | Read, Grep, Bash | OWASP tools, nmap | High |
| database-developer | Read, Bash | psql, snowsql | Medium |
```

**1.2 Inventory skills**
- List all SKILL.md files
- Document skill domains, patterns, anti-patterns
- Estimate content volume (for knowledge source sizing)

**1.3 Map to CoPilot agent types**

**Decision criteria:**

| Claude Code Agent | CoPilot Type | Rationale |
|-------------------|--------------|-----------|
| FAQ bot | Declarative agent | Simple Q&A, no custom logic |
| Code assistant with tool usage | Custom agent | Requires Read/Write file operations |
| Approver (workflow) | Declarative agent | Guided multi-step flow |
| Complex data analyzer | Custom agent | Custom processing logic needed |

**1.4 Identify required actions**

**Map Claude Code tools to CoPilot actions:**

| Claude Tool | CoPilot Action | Implementation |
|-------------|----------------|----------------|
| Read | API plugin or Power Automate | GET endpoint or SharePoint connector |
| Write | API plugin or Power Automate | POST endpoint or file write flow |
| Bash (git operations) | Power Automate | Git connector or REST API |
| Bash (npm run) | Power Automate | Azure DevOps REST API |
| Grep | Custom API | Build search endpoint |

**1.5 Evaluate knowledge sources**

**Content inventory:**
- Total CLAUDE.md files: X files
- Total skill files: Y files
- Total size: Z MB
- Format: Markdown

**Destination:**
- SharePoint library (preferred for versioning, search)
- File upload (if <100 files, static content)
- Website crawl (if content already published)

**1.6 Assess custom code requirements**

**Identify scenarios requiring custom agents:**
- Complex state management beyond built-in variables
- Custom algorithms or data transformations
- Integrations with legacy systems (no pre-built connectors)
- Advanced error handling or retry logic

**Outputs:**
- Migration assessment document
- Effort estimate (weeks, resources)
- Risk assessment
- Prioritization (which agents to migrate first)

---

### Phase 2: Knowledge Migration (2-3 weeks)

**Objective**: Move documentation and context to CoPilot knowledge sources

**Tasks:**

**2.1 Export CLAUDE.md files to SharePoint**

**Process:**
1. Create SharePoint site: "Agent Knowledge Hub"
2. Create document libraries by domain:
   - Backend Development
   - Frontend Development
   - Security
   - Database
   - Infrastructure

3. Convert markdown to Word/PDF (if needed):
   ```bash
   pandoc CLAUDE.md -o CLAUDE.pdf
   ```

4. Upload to appropriate library
5. Add metadata tags:
   - Domain: backend, frontend, security, etc.
   - Keywords: relevant technology keywords
   - Owner: team responsible

**2.2 Convert skill files**

**Process:**
1. Extract key sections from SKILL.md:
   - Key Principles
   - Patterns We Use
   - Anti-Patterns
   - Quick Reference

2. Format as knowledge documents:
   - Add headers for structure
   - Include examples
   - Add cross-references

3. Upload to SharePoint

**Example: backend-developer skill → knowledge document**

```markdown
# Backend Development Patterns

## Key Principles
- DRY (Don't Repeat Yourself)
- Single source of truth for configuration
- No hardcoded strings
- Comprehensive error handling

## Patterns We Use

### Repository Pattern
[Example code from skill]

### Dependency Injection
[Example code from skill]

## Anti-Patterns to Avoid

### Hardcoded Database Credentials
Problem: Security risk, inflexible
Solution: Use environment variables or secrets management

[More anti-patterns...]
```

**2.3 Configure SharePoint sync**

1. In CoPilot Studio > Knowledge Sources
2. Add knowledge source > SharePoint
3. Select site and libraries
4. Configure sync schedule: Real-time (or hourly if volume high)
5. Set permissions: Who can query knowledge

**2.4 Test knowledge retrieval**

**Test queries:**
```
User: "What's our pattern for database connections?"
Expected: Retrieves repository pattern doc, cites source

User: "How do we handle API authentication?"
Expected: Retrieves authentication pattern, provides example

User: "What are our code review standards?"
Expected: Retrieves code standards doc
```

**Measure:**
- Retrieval accuracy (correct document returned?)
- Response time (<3 seconds?)
- Source attribution (citation included?)

**Iterate:**
- Adjust keywords/metadata if retrieval poor
- Consolidate documents if too fragmented
- Split documents if too large (>5MB)

**Outputs:**
- SharePoint site configured and populated
- Knowledge sources connected to CoPilot Studio
- Test results documented

---

### Phase 3: Action Implementation (3-4 weeks)

**Objective**: Map Claude Code tool usage to CoPilot actions

**Tasks:**

**3.1 Map Read/Write operations**

**Read tool → API endpoint pattern:**

```yaml
# OpenAPI spec for file read
openapi: 3.0.0
paths:
  /files/{path}:
    get:
      summary: Read file contents
      parameters:
        - name: path
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          description: File contents
          content:
            text/plain:
              schema:
                type: string
```

**Write tool → Power Automate flow pattern:**

```
Trigger: CoPilot action called
Input: file_path, content
↓
Action: OneDrive - Create file
  Site: Company Intranet
  Folder: /Documents/{extract folder from file_path}
  File name: {extract filename from file_path}
  File content: {content}
↓
Return: Success message with file URL
```

**3.2 Map Bash operations**

**Git operations → Power Automate + Azure DevOps:**

```
# Claude Code: Bash("git commit -m 'message'")
# CoPilot: Power Automate flow

Trigger: CoPilot action "commit_changes"
Input: message
↓
Action: Azure DevOps - Create commit
  Project: {project_name}
  Repository: {repo_name}
  Branch: {current_branch}
  Message: {message}
↓
Return: Commit SHA, link to commit
```

**npm run operations → Power Automate + Azure DevOps:**

```
# Claude Code: Bash("npm run test")
# CoPilot: Power Automate flow

Trigger: CoPilot action "run_tests"
↓
Action: Azure DevOps - Queue build
  Pipeline: Test Pipeline
  Wait for completion: true
↓
Return: Test results summary, link to build
```

**3.3 Expose external APIs via OpenAPI**

**Example: Jira ticket creation (previously Bash curl)**

**Claude Code equivalent:**
```bash
Bash("curl -X POST https://jira.company.com/rest/api/3/issue ...")
```

**CoPilot implementation:**
```yaml
# OpenAPI spec
openapi: 3.0.0
servers:
  - url: https://jira.company.com/rest/api/3
paths:
  /issue:
    post:
      summary: Create Jira issue
      operationId: createJiraIssue
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
                    summary:
                      type: string
                    description:
                      type: string
                    issuetype:
                      type: object
                      properties:
                        name:
                          type: string
```

**Configure in CoPilot:**
1. Import OpenAPI spec
2. Configure OAuth authentication
3. Test action execution
4. Map to topic

**3.4 Implement custom connectors**

**When needed:**
- No pre-built Power Platform connector available
- Custom business logic required
- Legacy system integration

**Process:**
1. Define OpenAPI specification
2. Implement API if none exists
3. Configure authentication (API key, OAuth, certificate)
4. Test operations in Postman/Insomnia
5. Import to Power Platform
6. Share with team

**3.5 Test action execution and error handling**

**Test matrix:**

| Action | Happy Path | Error Cases | Performance |
|--------|------------|-------------|-------------|
| read_file | File exists | File not found, no permission | <500ms |
| write_file | Write succeeds | Disk full, permission denied | <1s |
| create_ticket | Ticket created | API down, invalid fields | <3s |
| run_tests | Tests pass | Tests fail, timeout | <5min |

**Error handling pattern:**

```
Action execution
↓
If error:
  - Log error details
  - Return user-friendly message:
    "Unable to create ticket. The ticketing system is temporarily unavailable. Please try again in a few minutes or contact support@company.com."
  - Optionally escalate to human
Else:
  - Return success message
```

**Outputs:**
- Actions implemented and tested
- Error handling validated
- Performance benchmarks met

---

### Phase 4: Agent Conversion (4-6 weeks)

**Objective**: Rebuild Claude Code agents as CoPilot agents

**Tasks:**

**4.1 Create declarative topics from agent workflows**

**Example: backend-developer agent → CoPilot topics**

**Claude Code agent (conceptual workflow):**
```
User: "Add authentication to user API"
↓
Agent reads current code (Read tool)
↓
Agent analyzes requirements
↓
Agent writes new code (Write tool)
↓
Agent runs tests (Bash tool)
↓
Agent commits changes (Bash tool)
```

**CoPilot declarative topics:**

**Topic 1: Add Feature (Guided Workflow)**
```
1. Trigger: "add feature", "implement"

2. Ask: "What feature should I add?"
   Store: feature_description

3. Ask: "Which file should I modify?"
   Entity: FilePath
   Store: target_file

4. Action: read_file (target_file)
   Store: current_code

5. Action: generate_code (feature_description, current_code)
   [Calls Azure OpenAI to generate new code]
   Store: new_code

6. Confirm: "Here's the generated code: {new_code}. Should I write this file?"
   Options: ["Yes", "No", "Modify"]

7. If Yes:
     Action: write_file (target_file, new_code)
     Action: run_tests ()
     If tests pass:
       Action: commit_changes ("feat: " + feature_description)
       Return: "Feature added and committed!"
     Else:
       Return: "Tests failed. Please review."
```

**4.2 Configure triggers**

**Map Claude Code agent descriptions to triggers:**

| Claude Agent | Description Keywords | CoPilot Triggers |
|--------------|---------------------|------------------|
| backend-developer | "backend", "API", "server-side" | "implement API", "add endpoint", "backend feature" |
| security-expert | "security", "vulnerability", "OWASP" | "security review", "check vulnerabilities", "OWASP compliance" |
| database-developer | "database", "SQL", "query" | "optimize query", "database schema", "SQL help" |

**Configure in CoPilot:**
1. Edit topic
2. Add trigger phrases (10-20 variations per topic)
3. Test trigger accuracy (does it match expected topic?)

**4.3 Wire up actions to topics**

**Process:**
1. Identify action needed in topic flow
2. Add "Call an action" node
3. Select action from dropdown
4. Map inputs from variables
5. Store outputs in variables

**Example:**
```
Topic flow:
  [User provides file path]
  ↓
  Call action: read_file
    Input: file_path = {var:user_file_path}
  ↓
  Store output: current_code = {action_output}
  ↓
  [Continue flow with current_code variable]
```

**4.4 Implement guardrails**

**Map Claude Code constraints to CoPilot guardrails:**

| Claude Constraint | CoPilot Guardrail |
|-------------------|-------------------|
| "Never modify production config files" | Content filter blocking "production.yaml" |
| "Always validate user input" | Entity validation + error handling |
| "No destructive operations without confirmation" | Confirmation step before irreversible actions |
| "Log all file modifications" | Custom telemetry on write_file action |

**Implementation:**
1. Configure content moderation level (High for production)
2. Add custom keyword blocklists
3. Enable PII detection
4. Add confirmation steps before destructive actions
5. Log critical operations to Application Insights

**4.5 Test conversation flows**

**Test cases:**

**Happy path:**
```
User: "Add authentication to user API"
Agent: "Which file should I modify?"
User: "src/api/user.py"
Agent: [Reads file, generates code]
Agent: "Here's the generated code... Should I write?"
User: "Yes"
Agent: [Writes file, runs tests, commits]
Agent: "Feature added! Commit SHA: abc123"
```

**Error handling:**
```
User: "Add authentication to user API"
Agent: "Which file should I modify?"
User: "nonexistent.py"
Agent: [Attempts read_file, fails]
Agent: "I couldn't find that file. Please check the path and try again."
```

**Edge cases:**
- User provides invalid file path
- Tests fail after code generation
- API dependencies unavailable
- User cancels midway through flow

**Outputs:**
- CoPilot agents created
- Topics configured with triggers
- Actions wired up
- Guardrails implemented
- Test results documented

---

### Phase 5: Testing and Validation (2-3 weeks)

**Objective**: Ensure migrated agents work correctly before production

**Tasks:**

**5.1 User Acceptance Testing (UAT)**

**Participants:** Real users who used Claude Code agents

**Test scenarios:**
1. Replicate common tasks from Claude Code usage
2. Test error handling (invalid inputs, failures)
3. Test edge cases (long conversations, complex queries)
4. Cross-channel testing (Teams, web, mobile)

**Feedback collection:**
- What works well?
- What's confusing or broken?
- What's missing compared to Claude Code?
- Suggestions for improvement

**5.2 Performance testing**

**Metrics:**
- Response time: <5 seconds for 95th percentile
- Concurrent conversations: Support 50+ simultaneous users
- Action execution time: <3 seconds per action
- Knowledge retrieval time: <2 seconds

**Tools:**
- Azure Load Testing
- Application Insights

**5.3 Security review**

**Checklist:**
- [ ] Authentication configured correctly (OAuth, managed identities)
- [ ] Input validation on all actions
- [ ] PII/PHI handling compliant
- [ ] Content moderation enabled
- [ ] Secrets stored in Key Vault (not hardcoded)
- [ ] Audit logging enabled

**Penetration testing (if required):**
- Attempt to bypass authentication
- Inject malicious payloads
- Attempt PII exfiltration
- Test rate limiting

**5.4 Accessibility testing**

**Tools:**
- Screen readers (NVDA, JAWS)
- Keyboard-only navigation
- High contrast mode

**Test:**
- Can user navigate agent with keyboard only?
- Are labels and descriptions clear for screen readers?
- Is contrast sufficient for visually impaired?

**5.5 Cross-channel validation**

**Test matrix:**

| Feature | Teams | Web | Mobile |
|---------|-------|-----|--------|
| Basic conversation | ✓ | ✓ | ✓ |
| File upload | ✓ | ✓ | ✓ |
| Authentication | ✓ | ✓ | ✓ |
| Adaptive cards (Teams) | ✓ | - | ✓ |
| Rich formatting | ✓ | Limited | ✓ |

**Outputs:**
- UAT feedback summary
- Performance benchmarks met
- Security assessment passed
- Accessibility validated
- Cross-channel compatibility confirmed

---

### Phase 6: Deployment (2-4 weeks)

**Objective**: Roll out CoPilot agents to production and decommission Claude Code

**Tasks:**

**6.1 Staged rollout**

**Pilot (10 users, 1 week):**
- Select early adopters
- Monitor closely for issues
- Collect feedback
- Fix critical bugs

**Limited (100 users, 2 weeks):**
- Expand to department or team
- Monitor usage and performance
- Refine based on feedback
- Prepare documentation and training

**Broad (1000+ users, 2-4 weeks):**
- Roll out to organization
- Provide training sessions
- Establish support channel
- Monitor for issues at scale

**6.2 Monitor usage and errors**

**Dashboards:**
- Conversation volume (by day, by channel)
- Active users (DAU, WAU, MAU)
- Error rate (by error type)
- Escalation rate (trend)
- User satisfaction (CSAT scores)

**Alerts:**
- Error rate >5% → Investigate immediately
- Escalation rate >30% → Review agent quality
- Response time >10s → Check infrastructure
- API failures → Check dependencies

**6.3 Iterate based on feedback**

**Review weekly:**
- User feedback (what's working, what's not)
- Error logs (common failures)
- Escalation reasons (opportunities for automation)
- Performance metrics (bottlenecks)

**Implement improvements:**
- Add topics for common escalations
- Improve knowledge sources (gaps in docs)
- Optimize slow actions
- Fix confusing conversation flows

**6.4 Decommission Claude Code agents**

**When ready:**
- CoPilot agents stable for 4+ weeks
- User satisfaction ≥80%
- Error rate <2%
- No critical missing features

**Process:**
1. Announce deprecation (30 days notice)
2. Disable Claude Code agent launches (prevent new usage)
3. Archive Claude Code configuration (for reference)
4. Document migration completion
5. Celebrate success!

**Outputs:**
- CoPilot agents in production
- Claude Code agents decommissioned
- Migration complete documentation
- Lessons learned documented

---

## Migration Challenges and Solutions

### Challenge 1: Complex Tool Usage

**Problem:** Claude Code agent uses 10+ different tools in complex orchestration

**Solution:**
- Break into multiple CoPilot topics
- Use custom agents for complex logic
- Implement orchestration via Power Automate
- Leverage Azure Functions for custom processing

### Challenge 2: File System Access

**Problem:** Claude Code Read/Write tools access local file system

**Solution:**
- Migrate files to OneDrive/SharePoint
- Build REST API for file operations
- Use Azure Blob Storage with API layer
- Use Power Automate file connectors

### Challenge 3: Custom Code Execution

**Problem:** Claude Code agent runs custom Python/JavaScript code

**Solution:**
- Implement as Azure Function
- Expose via API plugin
- Or migrate to Power Automate (if simple logic)
- Or build custom agent with code components

### Challenge 4: Large Context Windows

**Problem:** Claude Code agent maintains large conversation context

**Solution:**
- Use CoPilot session variables
- Store state in Dataverse (for complex state)
- Use conversation transcripts for history
- Implement external context service if needed

---

## Best Practices Summary

### Migration Planning
1. Assess before starting (don't dive in blind)
2. Prioritize high-value agents first
3. Migrate in phases (don't attempt all at once)
4. Test thoroughly before production

### Knowledge Migration
1. Organize content logically (by domain, topic)
2. Use SharePoint for versioning and search
3. Test retrieval accuracy
4. Iterate on metadata and keywords

### Action Implementation
1. Map tools to actions methodically
2. Implement error handling for all actions
3. Test performance (meet latency targets)
4. Secure authentication (no hardcoded secrets)

### Agent Conversion
1. Start with declarative agents (simplest)
2. Use custom agents only when necessary
3. Implement guardrails for safety
4. Test across all channels

### Deployment
1. Staged rollout (pilot → limited → broad)
2. Monitor closely in early stages
3. Iterate based on feedback
4. Don't decommission Claude Code until stable
