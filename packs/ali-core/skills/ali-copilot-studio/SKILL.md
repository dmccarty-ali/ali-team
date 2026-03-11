---
name: ali-copilot-studio
description: |
  Microsoft CoPilot Studio platform for building declarative and custom agents,
  actions, knowledge sources, and orchestration. Use when:

  PLANNING: Designing CoPilot agent architectures, planning migrations to CoPilot
  Studio from other platforms, evaluating CoPilot capabilities for requirements,
  architecting knowledge source integrations, planning action implementations.

  IMPLEMENTATION: Building declarative agents with topics and triggers, creating
  custom agents with code components, implementing actions via API plugins or Power
  Automate, configuring knowledge sources from SharePoint/files/websites, setting
  up authentication and connectors, deploying agents to channels.

  GUIDANCE: Asking about CoPilot Studio best practices, guardrails and safety
  patterns, orchestration strategies, authentication approaches, deployment options,
  governance models for enterprise rollout.

  REVIEW: Validating CoPilot agent implementations, auditing knowledge source
  configurations, checking action security, reviewing orchestration flows, assessing
  governance and compliance posture.

  Do NOT use for: MCP server development (use ali-mcp), generic OAuth 2.0 outside
  CoPilot context (use ali-external-integrations), Claude Code agent development
  (use ali-claude-code), SharePoint admin unrelated to CoPilot knowledge sources,
  Power Automate flow development unrelated to CoPilot actions.
---

# Microsoft CoPilot Studio

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing CoPilot Studio agent architectures
- Planning migrations from Claude Code to CoPilot Studio
- Evaluating CoPilot capabilities for requirements
- Architecting knowledge source integrations
- Planning action/plugin implementations
- Designing enterprise governance models

**Implementation:**
- Building declarative agents with topics and triggers
- Creating custom agents with code components
- Implementing actions via API plugins or Power Automate
- Configuring knowledge sources (SharePoint, files, websites)
- Setting up authentication and connectors
- Deploying agents to Teams, web, mobile channels
- Implementing guardrails and content filters

**Guidance/Best Practices:**
- Asking about CoPilot Studio best practices
- Learning orchestration strategies
- Understanding authentication approaches
- Evaluating deployment options
- Designing for enterprise governance

**Review/Validation:**
- Validating CoPilot agent implementations
- Auditing knowledge source configurations
- Checking action security patterns
- Reviewing orchestration flows
- Assessing governance compliance

---

## Key Principles

- **Declarative-first**: Use declarative agents for standard conversational scenarios, custom agents for specialized logic
- **Knowledge grounding**: Ground responses in knowledge sources to reduce hallucination
- **Action security**: Actions must validate inputs, use secure authentication, and implement least-privilege access
- **Orchestration is stateful**: CoPilot handles conversation state, context, and orchestration automatically
- **Guardrails are mandatory**: Content moderation, safety filters, and response validation required for production
- **Channel-agnostic design**: Agent should work across Teams, web, mobile without code changes
- **Governance scales**: Enterprise deployment requires topic approval workflows, access controls, audit logging
- **Test in isolation**: Test topics, actions, and knowledge independently before integration

---

## CoPilot Studio Architecture

### Platform Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Microsoft CoPilot Studio                       │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │  Agent Types                                                  ││
│  │  ┌──────────────────┐  ┌──────────────────────────────────┐  ││
│  │  │ Declarative      │  │ Custom Agents                     │  ││
│  │  │ - Topics         │  │ - Code components                 │  ││
│  │  │ - Triggers       │  │ - Custom orchestration            │  ││
│  │  │ - Entities       │  │ - Advanced integrations           │  ││
│  │  └──────────────────┘  └──────────────────────────────────┘  ││
│  └──────────────────────────────────────────────────────────────┘│
│  ┌──────────────────────────────────────────────────────────────┐│
│  │  Actions & Plugins                                            ││
│  │  ┌──────────────┐ ┌──────────────┐ ┌────────────────────┐   ││
│  │  │ API Plugins  │ │ OpenAPI Spec │ │ Power Automate     │   ││
│  │  └──────────────┘ └──────────────┘ └────────────────────┘   ││
│  └──────────────────────────────────────────────────────────────┘│
│  ┌──────────────────────────────────────────────────────────────┐│
│  │  Knowledge Sources                                            ││
│  │  ┌──────────────┐ ┌──────────────┐ ┌────────────────────┐   ││
│  │  │ SharePoint   │ │ File Upload  │ │ Website Crawl      │   ││
│  │  └──────────────┘ └──────────────┘ └────────────────────┘   ││
│  └──────────────────────────────────────────────────────────────┘│
│  ┌──────────────────────────────────────────────────────────────┐│
│  │  Guardrails & Safety                                          ││
│  │  - Content moderation - Safety filters - Response validation  ││
│  └──────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Deployment Channels                          │
│    Teams  │  Web  │  Mobile  │  Power Apps  │  Dynamics 365     │
└─────────────────────────────────────────────────────────────────┘
```

### Core Components

| Component | Purpose | Example Use Case |
|-----------|---------|------------------|
| **Declarative Agents** | Conversational flows with topics and triggers | FAQ bot, support agent, HR assistant |
| **Custom Agents** | Code-driven agents with custom logic | Complex workflows, external system integration |
| **Actions** | External operations via APIs or workflows | Create ticket, fetch data, send notification |
| **Knowledge Sources** | Grounding data for responses | Product docs, policies, internal wikis |
| **Guardrails** | Safety and compliance controls | Content filtering, PII detection, response validation |
| **Connectors** | Authentication and authorization | OAuth, API keys, service accounts |

---

## Agent Types - When to Use What

### Declarative Agents

**Use when:** Standard conversational scenarios, FAQ bots, guided workflows

**Key characteristics:**
- Built with visual designer (low-code)
- Topics define conversation flows
- Triggers activate based on user input
- Built-in orchestration and state management
- No custom code required

**Example:** IT Support Bot with password reset, VPN access, escalation topics

### Custom Agents

**Use when:** Complex logic, external integrations, advanced orchestration

**Key characteristics:**
- Code components (TypeScript, C#)
- Full control over conversation flow
- Custom state management
- Direct API integrations
- Developer skillset required

**Example:** Multi-step approval workflows, legacy system integration, complex data transformations

---

## Migration from Claude Code

### High-Level Comparison

| Feature | Claude Code | CoPilot Studio |
|---------|-------------|----------------|
| **Agent definition** | Markdown files | Visual designer + JSON |
| **Skills** | SKILL.md files | Knowledge sources + actions |
| **Tools** | Read, Write, Bash, etc. | API plugins, Power Automate |
| **Context** | File-based, manual management | Automatic conversation state |
| **Deployment** | CLI, local/cloud | Teams, web, mobile channels |
| **Orchestration** | Agent spawning (Task tool) | Built-in topic routing |

### Six-Phase Migration Strategy

**Phase 1: Assessment (1-2 weeks)**
- Inventory Claude Code agents and skills
- Map to CoPilot agent types
- Identify required actions
- Evaluate knowledge sources

**Phase 2: Knowledge Migration (2-3 weeks)**
- Export CLAUDE.md, skill files to SharePoint
- Configure SharePoint sync
- Test knowledge retrieval accuracy

**Phase 3: Action Implementation (3-4 weeks)**
- Map Read/Write/Bash operations to Power Automate
- Expose external APIs via OpenAPI specs
- Test action execution and error handling

**Phase 4: Agent Conversion (4-6 weeks)**
- Create declarative topics from agent workflows
- Configure triggers based on agent descriptions
- Wire up actions to topics
- Implement guardrails

**Phase 5: Testing and Validation (2-3 weeks)**
- UAT with real users
- Performance testing
- Security review
- Cross-channel validation

**Phase 6: Deployment (2-4 weeks)**
- Staged rollout (pilot → broader audience)
- Monitor usage and errors
- Iterate based on feedback
- Decommission Claude Code agents when stable

**For detailed migration steps, consult references/migration-details.md**

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| **Hardcoded credentials in actions** | Security risk, credential leaks | Use connectors with OAuth or Key Vault |
| **No guardrails in production** | Unsafe responses, compliance risk | Enable content moderation and safety filters |
| **Single environment for dev/prod** | Test changes affect live users | Separate dev/test/prod environments |
| **Knowledge source > 100 files** | Slow retrieval, poor relevance | Consolidate docs, use SharePoint with folders |
| **Custom agents for simple Q&A** | Over-engineering, higher maintenance | Use declarative agents with knowledge sources |
| **No escalation path** | Users stuck, poor experience | Always provide "talk to human" option |
| **Unvalidated API inputs** | Injection risks, data corruption | Validate all inputs in action implementations |
| **No logging or analytics** | Blind to usage, errors | Enable Application Insights, track key metrics |
| **Client-side authentication only** | Spoofable, not secure | Use server-side OAuth with managed identities |
| **Ignoring accessibility** | Excludes disabled users | Test with screen readers, keyboard navigation |

---

## Detailed Reference Material

This skill provides core concepts and decision-making context. For detailed implementation guidance:

### Actions and Plugins
For API plugin patterns, OpenAPI specifications, Power Automate flows, security requirements, and examples, consult **references/actions-and-plugins.md** covering:
- OpenAPI specification patterns
- Security requirements for plugins
- Power Automate action patterns
- Common action examples (SharePoint, Teams, Dataverse)

### Knowledge Sources
For SharePoint integration, file upload configuration, website crawl setup, and migration from Claude Code CLAUDE.md files, consult **references/knowledge-sources.md** covering:
- SharePoint setup and best practices
- File upload limitations and formats
- Website crawl configuration
- Dataverse integration

### Orchestration Patterns
For built-in orchestration features, topic design patterns, state management, and debugging, consult **references/orchestration-patterns.md** covering:
- Intent recognition and context management
- Guided workflow patterns
- Knowledge-grounded Q&A patterns
- Escalation patterns
- Multi-agent orchestration

### Authentication
For OAuth 2.0 configuration, API key management, managed identities, and Power Platform connectors, consult **references/authentication.md** covering:
- Authentication methods comparison
- OAuth 2.0 setup steps
- Security requirements
- Power Platform connector catalog

### Guardrails and Safety
For content moderation levels, safety filters, PII/PHI handling, and compliance controls, consult **references/guardrails-and-safety.md** covering:
- Content moderation configuration
- Microsoft-managed and custom filters
- GDPR, HIPAA, SOC 2 compliance
- Testing safety controls

### Deployment
For Teams deployment, web embed, mobile integration, and cross-channel considerations, consult **references/deployment.md** covering:
- Teams app manifest configuration
- Web embed customization
- Mobile deployment patterns
- Staged rollout strategies

### Governance
For enterprise governance models, COE structure, development lifecycle, and approval workflows, consult **references/governance.md** covering:
- Center of Excellence setup
- Agent registry management
- Seven-phase development lifecycle
- Environment strategy

### Monitoring
For analytics, custom telemetry, error logging, and dashboards, consult **references/monitoring.md** covering:
- Built-in analytics metrics
- Application Insights integration
- Real-time monitoring
- Alerting strategies

### Migration Details
For detailed Claude Code → CoPilot Studio migration steps, phase-by-phase guidance, and challenge solutions, consult **references/migration-details.md** covering:
- Feature mapping comparison
- Phase 1-6 detailed tasks
- Migration challenges and solutions
- Best practices summary

### Example Patterns
For complete implementation examples including SharePoint knowledge hub, Jira API integration, and multi-turn expense approval, consult **references/example-patterns.md** covering:
- SharePoint Knowledge Hub pattern
- API Action with Authentication pattern
- Multi-Turn Expense Approval pattern

### Quick Reference
For code snippets, entity types, debugging commands, and checklists, consult **references/quick-reference.md** covering:
- Common action YAML examples
- Entity type reference table
- PowerShell debugging commands
- Power Automate expression cheat sheet
- Best practices checklist

---

## Your Codebase

| Purpose | Location (Hypothetical) |
|---------|----------|
| Agent manifests | SharePoint: /CoPilot Agents/Manifests/ |
| OpenAPI specs | Git repo: /copilot-actions/openapi/ |
| Power Automate flows | Power Automate environment exports |
| Knowledge sources | SharePoint: /Knowledge Hub/ |
| Deployment scripts | Git repo: /deployment/copilot/ |

---

## References

- [Microsoft CoPilot Studio Documentation](https://learn.microsoft.com/en-us/microsoft-copilot-studio/)
- [Power Platform Admin Center](https://admin.powerplatform.microsoft.com/)
- [Power Automate Documentation](https://learn.microsoft.com/en-us/power-automate/)
- [OpenAPI Specification](https://swagger.io/specification/)
- [Microsoft Teams App Development](https://learn.microsoft.com/en-us/microsoftteams/platform/)
- [Azure Active Directory OAuth](https://learn.microsoft.com/en-us/azure/active-directory/develop/)
- [Dataverse API Reference](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/webapi/overview)

---

**Document Version:** 2.0
**Last Updated:** 2026-02-16
**Maintained By:** ALI AI Team
