---
# LangGraph skill for building stateful AI agent workflows
name: ali-langgraph
description: |
  LangGraph patterns for stateful multi-agent AI applications. Use when:

  PLANNING: Designing agent workflows, planning state schemas, architecting
  multi-agent systems, choosing checkpointing strategies, evaluating graph topology

  IMPLEMENTATION: Building StateGraphs, implementing nodes and edges, adding
  checkpointing, creating tool-calling agents, implementing human-in-the-loop

  GUIDANCE: Asking about LangGraph best practices, state management patterns,
  when to use subgraphs, how to structure multi-agent systems

  REVIEW: Checking graph structure, validating state schemas, reviewing
  error handling, auditing checkpointing configuration
---

# LangGraph Development

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing agent workflows or state machines
- Architecting multi-agent systems (supervisor, hierarchical, collaborative)
- Planning state schemas and reducer strategies
- Evaluating checkpointing approaches (memory, SQLite, PostgreSQL)
- Considering human-in-the-loop requirements

**Implementation:**
- Building StateGraph with nodes and edges
- Implementing conditional routing logic
- Adding persistence with checkpointers
- Creating tool-calling agents
- Implementing streaming patterns
- Building subgraphs and nested workflows

**Guidance/Best Practices:**
- Asking about LangGraph patterns and conventions
- State management strategies
- When to use reducers vs replace mode
- Multi-agent coordination patterns
- Error handling strategies

**Review/Validation:**
- Reviewing graph structure for anti-patterns
- Validating state schema design
- Checking error handling completeness
- Auditing checkpointing configuration
- Verifying human-in-the-loop implementation

---

## Key Principles

- **Explicit state over implicit**: Define TypedDict schemas with clear types
- **Partial updates**: Nodes return only changed fields, never mutate input
- **Bounded execution**: Always define exit conditions and max iterations
- **Durable persistence**: Never use MemorySaver in production
- **Separate concerns**: Tool execution in dedicated nodes, not hidden in logic
- **Type-safe routing**: Use Literal types for conditional edge returns
- **Error visibility**: Track errors in state, never swallow silently

---

## Patterns We Use

### Basic StateGraph Structure

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict, Annotated
from langgraph.graph.message import add_messages

class AgentState(TypedDict):
    # Messages accumulate via add_messages reducer
    messages: Annotated[list, add_messages]
    # Simple fields use replace mode (last write wins)
    status: str
    error: dict | None

def process_node(state: AgentState) -> dict:
    """Return partial update - only changed fields."""
    return {
        "status": "processed"
    }

# Build graph
workflow = StateGraph(AgentState)
workflow.add_node("process", process_node)
workflow.add_edge(START, "process")
workflow.add_edge("process", END)

# Compile
app = workflow.compile()
```

### Conditional Routing with Type Safety

```python
from typing import Literal

def route_by_status(state: AgentState) -> Literal["continue", "error", "end"]:
    """Type-safe routing prevents typo bugs."""
    if state.get("error"):
        return "error"
    if state["status"] == "complete":
        return "end"
    return "continue"

workflow.add_conditional_edges(
    "check_status",
    route_by_status,
    {
        "continue": "process",
        "error": "error_handler",
        "end": END
    }
)
```

### Production Checkpointing

```python
from langgraph.checkpoint.postgres import PostgresSaver

# Production: PostgreSQL
checkpointer = PostgresSaver.from_conn_string(
    "postgresql://user:password@host:5432/langgraph"
)
checkpointer.setup()  # Create tables on first run

app = workflow.compile(checkpointer=checkpointer)

# Thread ID provides conversation continuity
config = {"configurable": {"thread_id": f"user-{user_id}-session-{session_id}"}}
result = app.invoke(inputs, config)
```

### Tool-Calling Agent Pattern

```python
from langchain_core.tools import tool
from langchain_core.messages import ToolMessage

@tool
def search_database(query: str, limit: int = 10) -> list[dict]:
    """Search database for matching records.

    Args:
        query: Search query string
        limit: Maximum results to return
    """
    # Implementation
    return results

# Bind tools to model
model_with_tools = model.bind_tools([search_database])

def call_model(state: AgentState) -> dict:
    """Call model - may request tool use."""
    response = model_with_tools.invoke(state["messages"])
    return {"messages": [response]}

def execute_tools(state: AgentState) -> dict:
    """Execute requested tools - separate node for visibility."""
    last_message = state["messages"][-1]
    tool_messages = []

    for tool_call in last_message.tool_calls:
        tool_func = tool_map[tool_call["name"]]
        result = tool_func.invoke(tool_call["args"])
        tool_messages.append(
            ToolMessage(content=str(result), tool_call_id=tool_call["id"])
        )

    return {"messages": tool_messages}

def should_continue(state: AgentState) -> Literal["tools", "end"]:
    """Route to tools or end based on tool calls."""
    last_message = state["messages"][-1]
    if hasattr(last_message, "tool_calls") and last_message.tool_calls:
        return "tools"
    return "end"

# Wire up
workflow.add_node("agent", call_model)
workflow.add_node("tools", execute_tools)
workflow.add_edge(START, "agent")
workflow.add_conditional_edges("agent", should_continue, {"tools": "tools", "end": END})
workflow.add_edge("tools", "agent")
```

### Human-in-the-Loop with interrupt()

```python
from langgraph.types import interrupt, Command

def approval_node(state: AgentState) -> dict:
    """Pause for human approval."""
    action = state["pending_action"]

    # Pause execution and wait for human
    approved = interrupt(
        value={
            "action": action,
            "prompt": "Approve this action? (yes/no)"
        }
    )

    return {
        "approved": approved == "yes",
        "status": "approved" if approved == "yes" else "rejected"
    }

# Resume after human responds
result = app.invoke(Command(resume="yes"), config)
```

### Supervisor Multi-Agent Pattern

```python
from langgraph_supervisor import create_supervisor
from langgraph.prebuilt import create_react_agent

# Specialized agents
research_agent = create_react_agent(
    model,
    tools=[search_tool, scrape_tool],
    name="research_agent"
)

analysis_agent = create_react_agent(
    model,
    tools=[calculator, plotter],
    name="analysis_agent"
)

# Supervisor coordinates
workflow = create_supervisor(
    agents=[research_agent, analysis_agent],
    model=model,
    prompt="You coordinate research and analysis tasks."
)

app = workflow.compile(checkpointer=checkpointer)
```

### Bounded Retry with Backoff

```python
import time

def retry_node(state: AgentState) -> dict:
    """Retry with exponential backoff - bounded."""
    attempt = state.get("retry_attempt", 0)
    max_attempts = 5

    if attempt > 0:
        delay = min(2 ** attempt, 60)  # Cap at 60 seconds
        time.sleep(delay)

    try:
        result = unstable_operation(state["input"])
        return {"result": result, "retry_attempt": 0, "error": None}
    except Exception as e:
        if attempt >= max_attempts:
            return {
                "error": {"type": type(e).__name__, "message": str(e)},
                "status": "failed"
            }
        return {
            "retry_attempt": attempt + 1,
            "error": {"type": type(e).__name__, "message": str(e)},
            "status": "retry"
        }
```

### Streaming Updates

```python
# Stream only state updates (efficient)
for chunk in app.stream(inputs, stream_mode="updates"):
    print(chunk)  # Only changed fields

# Stream messages for chat UI
for chunk in app.stream(inputs, stream_mode="messages"):
    print(chunk)  # New messages only

# Token-level streaming
async for event in app.astream_events(inputs):
    if event["event"] == "on_chat_model_stream":
        print(event["data"]["chunk"].content, end="", flush=True)
```

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Mutating input state | Breaks LangGraph's update model | Return partial dict with changes |
| MemorySaver in production | Data lost on restart | Use PostgresSaver or SqliteSaver |
| Unbounded retry loops | Can run forever | Track attempt count, set max |
| Hidden tool calls | Hard to debug and trace | Separate tools node |
| Random thread_id | Loses conversation history | Use stable user/session ID |
| Full state streaming | Wastes bandwidth | Use "updates" or "messages" mode |
| Silent error swallowing | Debugging nightmare | Track errors in state |
| Untyped routing | Typos cause silent failures | Use Literal return types |
| Bloated state | Memory issues, slow checkpoints | Keep state minimal |
| Cycles without exit | Infinite loops | Add max_iterations check |

---

## State Management Rules

### Reducers

| Field Type | Reducer | Behavior |
|------------|---------|----------|
| Messages | `add_messages` | Appends, handles tool responses |
| Lists to accumulate | `operator.add` | Concatenates lists |
| Counters | `operator.add` | Sums integers |
| Simple fields | None (default) | Last write wins |

### State Schema Guidelines

```python
from typing import TypedDict, Annotated
from operator import add
from langgraph.graph.message import add_messages

class WellDesignedState(TypedDict):
    # Messages - always use add_messages
    messages: Annotated[list, add_messages]

    # Accumulated errors for debugging
    error_history: Annotated[list[dict], add]

    # Simple fields - replace mode
    status: str
    current_step: str

    # Optional error tracking
    last_error: dict | None
    retry_count: int
```

---

## Checkpointer Selection

| Checkpointer | Use Case | Notes |
|--------------|----------|-------|
| MemorySaver | Tutorials, demos only | Never in production |
| SqliteSaver | Local dev, single-user | File-based persistence |
| PostgresSaver | Production | Scalable, durable |
| AsyncPostgresSaver | Async production | Use with async graphs |

**Required setup:**
```python
# Always call setup() on first use
checkpointer.setup()
```

---

## Quick Reference

### Imports

```python
# Core
from langgraph.graph import StateGraph, START, END
from typing import TypedDict, Annotated, Literal

# State
from langgraph.graph.message import add_messages
from operator import add

# Checkpointing
from langgraph.checkpoint.memory import MemorySaver  # Dev only
from langgraph.checkpoint.sqlite import SqliteSaver
from langgraph.checkpoint.postgres import PostgresSaver

# Human-in-the-loop
from langgraph.types import interrupt, Command

# Prebuilt agents
from langgraph.prebuilt import create_react_agent
from langgraph_supervisor import create_supervisor

# Tools
from langchain_core.tools import tool
from langchain_core.messages import ToolMessage, AIMessage
```

### Graph Building

```python
# Create graph
workflow = StateGraph(MyState)

# Add nodes
workflow.add_node("name", function)

# Add edges
workflow.add_edge(START, "first_node")
workflow.add_edge("node_a", "node_b")
workflow.add_edge("last_node", END)

# Conditional edges
workflow.add_conditional_edges("node", router_func, {"a": "node_a", "b": "node_b"})

# Compile
app = workflow.compile(checkpointer=checkpointer)

# Invoke
result = app.invoke(inputs, config={"configurable": {"thread_id": "..."}})

# Stream
for chunk in app.stream(inputs, stream_mode="updates"):
    ...
```

### Human-in-the-Loop

```python
# Pause with interrupt()
response = interrupt(value={"prompt": "Approve?"})

# Resume with Command
result = app.invoke(Command(resume="yes"), config)

# Graph-level interrupts
app = workflow.compile(
    checkpointer=checkpointer,
    interrupt_before=["sensitive_node"]
)
```

---

## References

- [LangGraph Documentation](https://python.langchain.com/docs/langgraph)
- [LangGraph GitHub](https://github.com/langchain-ai/langgraph)
- [LangGraph Supervisor](https://github.com/langchain-ai/langgraph-supervisor-py)
- [LangGraph Checkpointing Guide](https://python.langchain.com/docs/langgraph/persistence)
- [Building Agents from First Principles](https://blog.langchain.com/building-langgraph/)
