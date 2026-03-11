# Skill Distribution and API Usage

## Current Distribution Model (January 2026)

### Individual Users

**Upload to Claude.ai:**
1. Download skill folder from repository
2. Zip folder if needed (e.g., `my-skill/` becomes `my-skill.zip`)
3. Navigate to Settings > Capabilities > Skills
4. Upload the zip file
5. Enable the skill

**Claude Code CLI:**
1. Download skill folder
2. Place in Claude Code skills directory:
   - User: `~/.claude/skills/`
   - Project: `.claude/skills/`
3. Skill loads automatically on next session

### Organization-Level Skills

**Centralized Deployment (Shipped December 18, 2025):**
- Admins deploy skills workspace-wide
- Automatic updates to all users
- Centralized management console
- Version control and rollback support

**Benefits:**
- Consistent skill versions across team
- No manual user setup required
- Security and compliance control
- Usage analytics

### Recommended Distribution Approach

**1. Host on GitHub**
- Public repository for open-source skills
- Clear README with:
  - What the skill does (outcomes, not features)
  - Installation instructions (download, upload, enable)
  - Example use cases with before/after
  - Prerequisites (MCP servers, tools, credentials)
- Example: `https://github.com/your-org/braid-skill`

**2. Document in Your MCP Repo**
- Link to skill from MCP server documentation
- Explain combined value:
  - "MCP gives tool access, skill teaches workflows"
  - "Together they enable AI-powered project management"
- Show example conversations with skill + MCP working together

**3. Create an Installation Guide**

Step-by-step guide:

```markdown
# Installing the Braid Skill

## Prerequisites
- Claude.ai account or Claude Code CLI
- Braid MCP server configured (see MCP setup)

## Installation

### For Claude.ai
1. Download skill: [braid-skill.zip](link)
2. Go to Settings > Capabilities > Skills
3. Click "Upload Skill"
4. Select braid-skill.zip
5. Enable the skill

### For Claude Code
1. Clone repository:
   ```
   git clone https://github.com/your-org/braid-skill.git
   cd braid-skill
   ```
2. Copy to skills directory:
   ```
   cp -r braid-skill ~/.claude/skills/
   ```
3. Launch Claude Code:
   ```
   claude
   ```

## Verification

Ask Claude: "Help me create a new BRAID risk item"

Expected: Skill activates, provides workflow guidance, uses BRAID MCP tools.
```

---

## Using Skills via API

For programmatic use in applications, agents, and automated workflows.

### Key Capabilities

**Skills API Endpoint:**
- `/v1/skills` - List and manage skills
- Version control through Claude Console
- Programmatic skill enablement/disablement

**Messages API Integration:**
- `container.skills` parameter to specify skills
- Works with existing Messages API calls
- Compatible with Claude Agent SDK

### When to Use API vs Claude.ai/Claude Code

| Use Case | Best Surface | Why |
|----------|--------------|-----|
| End users interacting directly | Claude.ai / Claude Code | Manual workflows, testing, exploration |
| Manual skill testing during development | Claude.ai / Claude Code | Faster iteration, immediate feedback |
| Applications using skills programmatically | API | Automation, scale, integration |
| Production deployments at scale | API | Reliability, monitoring, control |
| Automated pipelines and agent systems | API | Headless operation, CI/CD |

### Example: Adding Skills to API Requests

**Python example using Anthropic client:**

```python
from anthropic import Anthropic

client = Anthropic()

response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    container={
        "skills": ["braid-skill", "sentry-skill"]
    },
    messages=[{
        "role": "user",
        "content": "Create a risk item for the authentication bug"
    }]
)
```

**Key parameters:**
- `container.skills`: List of skill names to activate
- Skills must be deployed via Console first
- Version controlled through Console versioning

**Best practices:**
- Pin skill versions for production stability
- Test skill combinations before deployment
- Monitor API usage and token consumption
- Handle skill loading failures gracefully

### API Use Cases

**1. Automated Agent Systems**
- Multi-agent workflows with specialized skills per agent
- Dynamic skill loading based on task routing
- Skill-based agent specialization

**2. Production Applications**
- User-facing apps with embedded Claude + skills
- Backend automation with domain-specific skills
- Workflow engines orchestrating skill-powered tasks

**3. CI/CD Pipelines**
- Code review agents with security/performance skills
- Documentation generation with style guide skills
- Test generation with domain-specific skills

---

## Positioning Your Skill

### Focus on Outcomes, Not Features

**Bad (feature-focused):**
> "This skill integrates with the Braid MCP server and provides commands for creating RAID items with type validation and workstream assignment."

**Good (outcome-focused):**
> "Manage project risks, issues, and decisions in seconds. Skip the RAID log spreadsheet -- just describe the problem and get a tracked item with proper metadata."

**Formula:**
- What users accomplish (outcome)
- How much time/effort saved (quantify)
- Pain point eliminated (before state)

### Highlight MCP + Skills Story

**The Value Proposition:**

| Component | Role | Value |
|-----------|------|-------|
| MCP Server | Tool access | Connects Claude to your systems |
| Skill | Workflow guidance | Teaches Claude how to use tools effectively |
| Together | AI-powered work | Users skip integration complexity |

**Example messaging:**
> "The Braid MCP server gives Claude access to your RAID log. The Braid skill teaches Claude your team's RAID process -- how to write good risk descriptions, when to escalate, what metadata to capture. Together, they turn 'I have a risk' into a tracked, categorized RAID item with zero training."

### Positioning by Audience

**For Developers:**
- Emphasize skipping integration code
- Show before/after code complexity
- Highlight maintenance reduction

**Example:**
> "Without skill: 50 lines of integration code per feature. With skill: one sentence."

**For End Users:**
- Focus on speed and simplicity
- Show conversation examples
- Emphasize no technical knowledge required

**Example:**
> "Just describe what you need. Claude handles the rest -- no forms, no dropdowns, no documentation to read."

**For Product Teams:**
- Quantify support load reduction
- Show adoption metrics
- Emphasize faster time-to-value

**Example:**
> "75% reduction in support tickets. Users productive in minutes, not days."

---

## Distribution Checklist

### Before Release

- [ ] Skill thoroughly tested (trigger tests, functional tests, baseline comparison)
- [ ] README with clear installation steps
- [ ] Example use cases documented
- [ ] Prerequisites clearly stated
- [ ] License specified
- [ ] Versioning strategy defined

### For GitHub Distribution

- [ ] Repository created (public or private)
- [ ] README with installation, examples, prerequisites
- [ ] Releases with downloadable zips
- [ ] Issues enabled for user feedback
- [ ] Link to MCP server documentation (if applicable)

### For Organization Deployment

- [ ] Admin access to Claude Console
- [ ] Skill uploaded to Console
- [ ] Version tagged and described
- [ ] Team notified of new skill
- [ ] Usage tracking configured
- [ ] Rollback plan documented

### Documentation

- [ ] Installation guide (step-by-step)
- [ ] Use case examples (before/after conversations)
- [ ] Troubleshooting section (common issues)
- [ ] Integration guide (if requires MCP or other setup)
- [ ] Changelog (version history)

---

## References

- [Claude Skills Documentation](https://docs.anthropic.com/claude-code/skills)
- [Messages API - Skills Parameter](https://docs.anthropic.com/api/messages#skills)
- [Claude Agent SDK](https://docs.anthropic.com/agent-sdk)
- [Organization Skills Management](https://docs.anthropic.com/console/skills)
