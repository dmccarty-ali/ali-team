---
name: ali-mcp
description: |
  Model Context Protocol (MCP) development patterns for cross-platform AI tool integration. Use when:

  PLANNING: Designing MCP servers, planning tool schemas, architecting UI extensions,
  evaluating transport types, considering security requirements

  IMPLEMENTATION: Building MCP servers, implementing tools, creating MCP Apps UI,
  writing resource handlers, configuring transports

  GUIDANCE: Best practices for MCP development, tool schema design, security patterns,
  testing strategies, client compatibility

  REVIEW: Auditing MCP server security, validating tool schemas, reviewing UI implementations,
  checking transport configurations

  Do NOT use for Microsoft CoPilot Studio agent development or CoPilot-specific
  actions and connectors (use ali-copilot-studio instead)
---

# Model Context Protocol (MCP)

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing MCP servers or tool architectures
- Planning tool schemas and input validation
- Architecting UI extensions (MCP Apps)
- Evaluating transport types (stdio, HTTP, SSE)
- Considering security requirements for MCP integrations

**Implementation:**
- Building MCP servers with tool implementations
- Creating interactive UI components via MCP Apps
- Writing resource handlers and prompts
- Configuring transport layers
- Implementing authentication and authorization

**Guidance/Best Practices:**
- Asking about MCP development patterns
- Tool schema design best practices
- Security patterns for MCP servers
- Testing strategies for MCP tools
- Client compatibility considerations

**Review/Validation:**
- Auditing MCP server security
- Validating tool input schemas
- Reviewing UI implementations for security
- Checking transport configurations
- Verifying cross-client compatibility

---

## Key Principles

- **MCP is cross-platform:** Works with Claude, Goose, VS Code, ChatGPT - don't assume Claude-specific features
- **Tools are the core primitive:** Every MCP server exposes capabilities as tools with defined schemas
- **Schema validation is mandatory:** All tool inputs must have JSON Schema definitions
- **Transport determines deployment:** stdio for local, HTTP/SSE for remote, choose based on use case
- **UI is optional but powerful:** MCP Apps enables rich interactive UIs within conversations
- **Security is layered:** Sandboxing, content review, message auditing, explicit user approval
- **Resources complement tools:** Use resources for read-only data, tools for operations
- **Error messages are user-facing:** Format errors for display, include actionable guidance

---

## MCP Architecture

### Protocol Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      MCP Clients                              │
│  Claude Code │ Claude Desktop │ VS Code │ Goose │ ChatGPT   │
└──────────────────────┬──────────────────────────────────────┘
                       │ MCP Protocol (JSON-RPC 2.0)
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                      MCP Server                               │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────────┐│
│  │   Tools     │ │  Resources  │ │       Prompts           ││
│  │ (Actions)   │ │ (Read-only) │ │ (Guided interactions)   ││
│  └─────────────┘ └─────────────┘ └─────────────────────────┘│
│  ┌─────────────────────────────────────────────────────────┐│
│  │                    MCP Apps (UI Extensions)             ││
│  │  Interactive dashboards, forms, viewers, visualizations ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

### Core Primitives

| Primitive | Purpose | Example |
|-----------|---------|---------|
| **Tools** | Actions with side effects | Create item, run query, send message |
| **Resources** | Read-only data access | File contents, database schemas |
| **Prompts** | Guided interactions | Templates, workflows |
| **Apps** | Interactive UI components | Dashboards, forms, viewers |

---

## Transport Types

### stdio (Local Process)

**Use when:** Local development, CLI tools, file-based servers

```json
{
  "mcpServers": {
    "braid": {
      "type": "stdio",
      "command": "python",
      "args": ["-m", "braid_mcp_server"],
      "env": {
        "BRAID_PROJECT_DIR": "${workspaceFolder}"
      }
    }
  }
}
```

**Characteristics:**
- Process spawned by client
- Communication via stdin/stdout
- No network exposure
- Environment variables passed at spawn
- Process lifecycle managed by client

### HTTP (Remote Server)

**Use when:** Shared services, cloud deployment, multi-user access

```json
{
  "mcpServers": {
    "api": {
      "type": "http",
      "url": "https://api.example.com/mcp",
      "headers": {
        "Authorization": "Bearer ${API_TOKEN}"
      }
    }
  }
}
```

**Characteristics:**
- Stateless request/response
- Standard HTTP authentication
- Load balancer compatible
- Requires HTTPS in production

### SSE (Server-Sent Events)

**Use when:** Real-time updates, long-running operations, streaming results

```json
{
  "mcpServers": {
    "monitor": {
      "type": "sse",
      "url": "https://monitor.example.com/mcp/sse"
    }
  }
}
```

**Characteristics:**
- Server-initiated messages
- Persistent connection
- Progress updates for long operations
- Event-driven architecture

---

## Tool Development

### Tool Schema Pattern

```python
from mcp import Server
from mcp.types import Tool, TextContent
import json

server = Server("my-server")

@server.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(
            name="my_tool",
            description="Brief description for AI to understand when to use",
            inputSchema={
                "type": "object",
                "properties": {
                    "required_param": {
                        "type": "string",
                        "description": "What this parameter does"
                    },
                    "optional_param": {
                        "type": "integer",
                        "description": "Optional with default",
                        "default": 10
                    }
                },
                "required": ["required_param"]
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    if name == "my_tool":
        result = do_something(arguments["required_param"])
        return [TextContent(type="text", text=json.dumps(result))]
    raise ValueError(f"Unknown tool: {name}")
```

### Schema Best Practices

| Practice | Why | Example |
|----------|-----|---------|
| Descriptive names | AI uses names to select tools | `braid_add_item` not `add` |
| Clear descriptions | AI reads descriptions for context | "Create a new RAID item with type, title, and optional fields" |
| Typed properties | Enables validation | `"type": "integer"` not `"type": "string"` for numbers |
| Enums for choices | Constrains valid values | `"enum": ["risk", "issue", "action"]` |
| Default values | Reduces required inputs | `"default": 10` |
| Required array | Explicit required fields | `"required": ["title", "type"]` |

### Tool Response Patterns

```python
# Success response
return [TextContent(
    type="text",
    text="Created item #42: 'Fix authentication bug'"
)]

# Structured data response
return [TextContent(
    type="text",
    text=json.dumps({
        "status": "success",
        "item_num": 42,
        "title": "Fix authentication bug"
    }, indent=2)
)]

# Error response (still successful tool call, but indicates business error)
return [TextContent(
    type="text",
    text="Error: Item #99 not found. Use braid_list_items to see available items."
)]
```

---

## MCP Apps (UI Extensions)

### Overview

MCP Apps is the official UI extension enabling tools to deliver interactive components directly within conversations. **Production-ready as of January 2026.**

**Capabilities:**
- Interactive dashboards with filtering and drill-down
- Configuration forms with dependent field logic
- Document viewers with inline annotations
- Real-time monitoring displays
- Multi-step workflows and visualizations

**Client Support:**
- Claude (web and desktop)
- Goose
- VS Code Insiders
- ChatGPT

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      MCP Host (Client)                        │
│  ┌─────────────────────────────────────────────────────────┐│
│  │  Tool Result with UI Metadata                           ││
│  │  { "_meta": { "ui": { "resourceUri": "ui://..." } } }   ││
│  └──────────────────────────┬──────────────────────────────┘│
│                             │ Fetch UI Resource              │
│  ┌──────────────────────────▼──────────────────────────────┐│
│  │  Sandboxed iframe                                        ││
│  │  ┌────────────────────────────────────────────────────┐ ││
│  │  │  HTML/JavaScript UI Component                       │ ││
│  │  │  - Receives tool results                            │ ││
│  │  │  - Sends tool invocations                          │ ││
│  │  │  - Updates model context                           │ ││
│  │  └────────────────────────────────────────────────────┘ ││
│  │  postMessage (JSON-RPC)                                  ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

### Two Primitives

1. **Tools with UI metadata** - Include `_meta.ui.resourceUri` field referencing UI resources
2. **UI Resources** - Server-side resources delivered via `ui://` scheme containing bundled HTML/JavaScript

### Implementation with SDK

```python
from mcp import Server
from mcp.types import Tool, TextContent
from modelcontextprotocol.ext_apps import App

server = Server("dashboard-server")

@server.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(
            name="open_dashboard",
            description="Open interactive project dashboard",
            inputSchema={
                "type": "object",
                "properties": {
                    "project_id": {"type": "string"}
                },
                "required": ["project_id"]
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    if name == "open_dashboard":
        project_data = load_project(arguments["project_id"])
        return [TextContent(
            type="text",
            text=json.dumps(project_data),
            _meta={
                "ui": {
                    "resourceUri": f"ui://dashboard/{arguments['project_id']}"
                }
            }
        )]

@server.list_resources()
async def list_resources():
    return [
        Resource(
            uri="ui://dashboard/{project_id}",
            name="Project Dashboard",
            mimeType="text/html"
        )
    ]

@server.read_resource()
async def read_resource(uri: str):
    if uri.startswith("ui://dashboard/"):
        return DASHBOARD_HTML  # Bundled HTML/JS
```

### App Class (Client-Side JavaScript)

```javascript
import { App } from '@modelcontextprotocol/ext-apps';

const app = new App();

// Receive tool results from host
app.onToolResult((result) => {
    const data = JSON.parse(result.text);
    renderDashboard(data);
});

// Invoke server tools from UI
async function refreshData() {
    const result = await app.invokeTool('get_metrics', { period: '7d' });
    updateMetrics(result);
}

// Update model context based on user interaction
function onItemSelect(item) {
    app.updateContext({
        selectedItem: item.id,
        action: 'user selected item for discussion'
    });
}
```

### Security Framework

| Layer | Protection |
|-------|------------|
| **Iframe sandboxing** | Restricted permissions, isolated execution |
| **Content review** | Hosts pre-render and review HTML before display |
| **Message auditing** | JSON-RPC messages logged for inspection |
| **User approval** | Optional explicit approval for UI-initiated tool calls |

### UI Component Examples

| Component Type | Use Case | Tools Provided |
|----------------|----------|----------------|
| **Dashboard** | Project metrics, KPIs | Refresh, filter, export |
| **Form** | Configuration, data entry | Validate, submit, cancel |
| **Viewer** | Documents, PDFs, images | Navigate, annotate, zoom |
| **Monitor** | Real-time metrics | Start, stop, configure alerts |
| **Visualization** | Charts, graphs, maps | Zoom, filter, export |
| **Workflow** | Multi-step processes | Next, back, save draft |

---

## Security Patterns

### Input Validation

```python
from pydantic import BaseModel, Field, validator

class AddItemInput(BaseModel):
    item_type: str = Field(..., pattern="^(risk|issue|action|decision)$")
    title: str = Field(..., min_length=1, max_length=200)
    priority: int = Field(default=3, ge=1, le=5)

    @validator('title')
    def sanitize_title(cls, v):
        # Remove potential injection attempts
        return v.replace('<', '&lt;').replace('>', '&gt;')

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "add_item":
        validated = AddItemInput(**arguments)  # Raises on invalid
        return create_item(validated)
```

### File Path Safety

```python
import os
from pathlib import Path

ALLOWED_BASE = Path("/data/projects")

def safe_path(user_path: str) -> Path:
    """Resolve path and verify it's within allowed directory."""
    resolved = (ALLOWED_BASE / user_path).resolve()
    if not str(resolved).startswith(str(ALLOWED_BASE)):
        raise ValueError(f"Path traversal attempt: {user_path}")
    return resolved
```

### Credential Handling

```python
import os

# Environment variables for secrets (never in code)
API_KEY = os.environ.get("API_KEY")
if not API_KEY:
    raise RuntimeError("API_KEY environment variable required")

# Never log credentials
logger.info(f"Connecting to API...")  # Good
logger.info(f"Using key: {API_KEY}")  # BAD - never do this
```

### Rate Limiting

```python
from functools import lru_cache
from time import time

call_timestamps: dict[str, list[float]] = {}
RATE_LIMIT = 100  # calls per minute

def check_rate_limit(tool_name: str) -> bool:
    now = time()
    timestamps = call_timestamps.get(tool_name, [])
    timestamps = [t for t in timestamps if now - t < 60]  # Last minute
    if len(timestamps) >= RATE_LIMIT:
        return False
    timestamps.append(now)
    call_timestamps[tool_name] = timestamps
    return True
```

---

## Testing Patterns

### Unit Testing Tools

```python
import pytest
from unittest.mock import Mock, patch

@pytest.fixture
def mock_store():
    """Mock data store for isolated tests."""
    store = Mock()
    store.load.return_value = {"items": []}
    return store

def test_add_item_success(mock_store):
    with patch('my_server.store', mock_store):
        result = add_item("risk", "Security vulnerability")
        assert "Created" in result
        mock_store.save.assert_called_once()

def test_add_item_invalid_type(mock_store):
    with patch('my_server.store', mock_store):
        with pytest.raises(ValueError):
            add_item("invalid", "Test")
```

### Integration Testing

```python
import pytest
from mcp import ClientSession
from mcp.client.stdio import stdio_client

@pytest.fixture
async def mcp_session():
    """Create real MCP session for integration tests."""
    async with stdio_client(
        command="python",
        args=["-m", "my_server"]
    ) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            yield session

@pytest.mark.asyncio
async def test_list_tools(mcp_session):
    tools = await mcp_session.list_tools()
    assert len(tools) > 0
    assert any(t.name == "my_tool" for t in tools)

@pytest.mark.asyncio
async def test_call_tool(mcp_session):
    result = await mcp_session.call_tool("my_tool", {"param": "value"})
    assert result.content[0].type == "text"
```

### UI Component Testing

```javascript
// test/dashboard.test.js
import { render, fireEvent, waitFor } from '@testing-library/react';
import Dashboard from '../components/Dashboard';

test('dashboard filters by date range', async () => {
    const mockData = [{ id: 1, date: '2026-01-15' }];
    render(<Dashboard data={mockData} />);

    fireEvent.click(screen.getByText('Last 7 days'));

    await waitFor(() => {
        expect(screen.getByTestId('item-count')).toHaveTextContent('1');
    });
});

test('dashboard invokes refresh tool', async () => {
    const mockInvokeTool = jest.fn();
    render(<Dashboard onInvokeTool={mockInvokeTool} />);

    fireEvent.click(screen.getByText('Refresh'));

    expect(mockInvokeTool).toHaveBeenCalledWith('refresh_data', {});
});
```

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| No input schema | Clients can't validate, AI guesses | Define complete JSON Schema |
| Generic tool names | AI can't distinguish tools | Use specific names: `braid_add_item` |
| Untyped responses | Clients can't parse reliably | Return structured JSON |
| Hardcoded credentials | Security risk, not portable | Use environment variables |
| No error handling | Cryptic failures | Return actionable error messages |
| Sync blocking in async | Server hangs | Use async/await properly |
| Exposing file system | Path traversal risk | Validate and restrict paths |
| No rate limiting | DoS vulnerability | Implement per-tool limits |
| UI without sandboxing | XSS risk | Always use iframe sandbox |
| Assuming single client | Breaks compatibility | Test with multiple clients |

---

## Patterns We Use

### BRAID MCP Server Pattern

Our braid-mcp-server demonstrates production patterns:

```python
# Tool grouping by domain
tools/
├── raid_tools.py      # CRUD operations (7 tools)
├── budget_tools.py    # Budget tracking (4 tools)
├── reporting_tools.py # Reports and exports (6 tools)
└── advanced_tools.py  # Search, dependencies (4 tools)

# Tool implementation pattern
def braid_add_item(
    item_type: str,
    title: str,
    description: str = None,
    workstream: str = None,
    assigned_to: str = None
) -> str:
    """Create new BRAID item.

    Args:
        item_type: One of: risk, issue, action, decision, deliverable, plan_item
        title: Brief title for the item
        description: Detailed description (optional)
        workstream: Project workstream (optional)
        assigned_to: Assignee name (optional)

    Returns:
        Formatted string confirming creation with item number
    """
    project_file = get_project_file()
    store = YamlStore()
    data = store.load_raid_log(project_file)

    item = Item(
        item_num=data.metadata.next_item_num,
        type=ItemType(item_type),
        title=title,
        description=description,
        workstream=workstream,
        assigned_to=assigned_to
    )

    data.items.append(item)
    data.metadata.next_item_num += 1
    store.save_raid_log(project_file, data)

    return f"Created {item_type} #{item.item_num}: '{title}'"
```

### Dashboard UI Pattern (Opportunity for MCP Apps)

Current implementation opens browser:
```python
def braid_open_dashboard() -> str:
    """Generate and open HTML dashboard in browser."""
    html = generate_dashboard_html(data)
    path = save_report(html, "dashboard.html")
    webbrowser.open(f"file://{path}")
    return f"Dashboard opened: {path}"
```

With MCP Apps, could be inline:
```python
def braid_open_dashboard() -> TextContent:
    """Open interactive dashboard in conversation."""
    data = load_project_data()
    return TextContent(
        type="text",
        text=json.dumps(data),
        _meta={"ui": {"resourceUri": "ui://braid/dashboard"}}
    )
```

---

## Quick Reference

### MCP Python SDK

```bash
# Install
pip install mcp

# Create server
from mcp import Server
server = Server("my-server")

# Run server (stdio)
from mcp.server.stdio import stdio_server
async with stdio_server() as (read, write):
    await server.run(read, write)
```

### Tool Registration

```python
@server.list_tools()
async def list_tools() -> list[Tool]:
    return [Tool(name="...", description="...", inputSchema={...})]

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    ...
```

### Resource Registration

```python
@server.list_resources()
async def list_resources() -> list[Resource]:
    return [Resource(uri="...", name="...", mimeType="...")]

@server.read_resource()
async def read_resource(uri: str) -> str:
    ...
```

### MCP Apps SDK

```bash
npm install @modelcontextprotocol/ext-apps
```

```javascript
import { App } from '@modelcontextprotocol/ext-apps';
const app = new App();
app.onToolResult((result) => { ... });
await app.invokeTool('tool_name', { args });
app.updateContext({ key: 'value' });
```

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| Production MCP server | ~/ali-ai/projects/braid-mcp-server/ |
| Server implementation | braid_mcp_server/server.py |
| Tool implementations | braid_mcp_server/tools/*.py |
| Data models | braid_mcp_server/core/models.py |
| Integration tests | braid_mcp_server/tests/integration/ |

---

## References

- [MCP Specification](https://spec.modelcontextprotocol.io/)
- [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk)
- [MCP Apps Announcement](https://blog.modelcontextprotocol.io/posts/2026-01-26-mcp-apps/)
- [MCP Apps SDK](https://github.com/modelcontextprotocol/ext-apps)
- [Claude MCP Documentation](https://docs.anthropic.com/claude-code/mcp)

---

**Document Version:** 1.0
**Last Updated:** 2026-01-27
**Maintained By:** ALI AI Team
