# ali-ai-development - Tool Use Patterns Reference

Detailed tool use (function calling) implementation patterns.

---

## Defining Tools

```python
tools = [
    {
        "name": "get_weather",
        "description": "Get current weather for a location",
        "input_schema": {
            "type": "object",
            "properties": {
                "location": {
                    "type": "string",
                    "description": "City and state, e.g. San Francisco, CA"
                },
                "unit": {
                    "type": "string",
                    "enum": ["celsius", "fahrenheit"],
                    "description": "Temperature unit"
                }
            },
            "required": ["location"]
        }
    }
]

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=tools,
    messages=[{"role": "user", "content": "What's the weather in Tokyo?"}]
)
```

---

## Handling Tool Calls

```python
def process_response(response):
    """Process response, execute tools, continue conversation."""

    for block in response.content:
        if block.type == "tool_use":
            # Execute the tool
            tool_name = block.name
            tool_input = block.input

            if tool_name == "get_weather":
                result = get_weather(**tool_input)

            # Return result to Claude
            return client.messages.create(
                model="claude-sonnet-4-20250514",
                max_tokens=1024,
                tools=tools,
                messages=[
                    {"role": "user", "content": original_query},
                    {"role": "assistant", "content": response.content},
                    {
                        "role": "user",
                        "content": [
                            {
                                "type": "tool_result",
                                "tool_use_id": block.id,
                                "content": json.dumps(result)
                            }
                        ]
                    }
                ]
            )

    return response
```

---

## Structured Output with Tools

```python
# Use tools to enforce structured output
structured_output_tool = {
    "name": "submit_analysis",
    "description": "Submit the completed analysis in structured format",
    "input_schema": {
        "type": "object",
        "properties": {
            "summary": {"type": "string"},
            "severity": {"type": "string", "enum": ["low", "medium", "high", "critical"]},
            "findings": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "issue": {"type": "string"},
                        "location": {"type": "string"},
                        "recommendation": {"type": "string"}
                    }
                }
            }
        },
        "required": ["summary", "severity", "findings"]
    }
}

# Force tool use for structured response
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[structured_output_tool],
    tool_choice={"type": "tool", "name": "submit_analysis"},
    messages=[{"role": "user", "content": "Analyze this code for security issues: ..."}]
)

# Extract structured data
analysis = response.content[0].input  # Already parsed JSON
```

---

## Tool Execution Loop

```python
def run_agent_with_tools(task: str, tools: list, max_turns: int = 10) -> str:
    """Run agent with tool use loop."""
    messages = [{"role": "user", "content": task}]

    for turn in range(max_turns):
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            tools=tools,
            messages=messages
        )

        messages.append({"role": "assistant", "content": response.content})

        # Check if we're done
        if response.stop_reason == "end_turn":
            # Extract final answer
            for block in response.content:
                if block.type == "text":
                    return block.text

        # Execute any tool calls
        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                result = execute_tool(block.name, block.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": json.dumps(result)
                })

        if tool_results:
            messages.append({"role": "user", "content": tool_results})
        else:
            break  # No tools to execute

    raise RuntimeError(f"Agent exceeded max turns ({max_turns})")
```

---

## Multiple Tools Example

```python
tools = [
    {
        "name": "search_database",
        "description": "Search customer database",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {"type": "string"}
            },
            "required": ["query"]
        }
    },
    {
        "name": "send_email",
        "description": "Send email to customer",
        "input_schema": {
            "type": "object",
            "properties": {
                "to": {"type": "string"},
                "subject": {"type": "string"},
                "body": {"type": "string"}
            },
            "required": ["to", "subject", "body"]
        }
    },
    {
        "name": "create_ticket",
        "description": "Create support ticket",
        "input_schema": {
            "type": "object",
            "properties": {
                "customer_id": {"type": "string"},
                "issue": {"type": "string"},
                "priority": {
                    "type": "string",
                    "enum": ["low", "medium", "high"]
                }
            },
            "required": ["customer_id", "issue"]
        }
    }
]

# Claude can use multiple tools in sequence
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=2048,
    tools=tools,
    messages=[{
        "role": "user",
        "content": "Customer john@example.com has a high-priority billing issue. Look them up and create a ticket."
    }]
)
```

---

**Last Updated:** 2026-02-16
