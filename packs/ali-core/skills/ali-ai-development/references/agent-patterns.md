# ali-ai-development - Agent Patterns Reference

Detailed agent implementation patterns for single-agent and multi-agent systems.

---

## Single Agent with Tools

```python
class Agent:
    """Simple agent with tool use."""

    def __init__(self, tools: list, system_prompt: str):
        self.client = anthropic.Anthropic()
        self.tools = tools
        self.system = system_prompt
        self.messages = []

    def run(self, task: str) -> str:
        self.messages.append({"role": "user", "content": task})

        while True:
            response = self.client.messages.create(
                model="claude-sonnet-4-20250514",
                max_tokens=4096,
                system=self.system,
                tools=self.tools,
                messages=self.messages
            )

            self.messages.append({"role": "assistant", "content": response.content})

            # Check for tool use
            tool_uses = [b for b in response.content if b.type == "tool_use"]

            if not tool_uses:
                # No tools called - agent is done
                return response.content[0].text

            # Execute tools and add results
            tool_results = []
            for tool_use in tool_uses:
                result = self.execute_tool(tool_use.name, tool_use.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": tool_use.id,
                    "content": json.dumps(result)
                })

            self.messages.append({"role": "user", "content": tool_results})

    def execute_tool(self, name: str, input: dict):
        """Execute tool by name."""
        # Tool registry pattern
        tools = {
            "search": self.search,
            "calculate": self.calculate,
            "send_email": self.send_email,
        }

        if name not in tools:
            raise ValueError(f"Unknown tool: {name}")

        return tools[name](**input)
```

---

## Multi-Agent Orchestration

```python
class Orchestrator:
    """Coordinate multiple specialized agents."""

    def __init__(self):
        self.agents = {
            "researcher": ResearchAgent(),
            "coder": CodingAgent(),
            "reviewer": ReviewAgent(),
        }

    def process(self, task: str) -> dict:
        # Plan the workflow
        plan = self.plan(task)

        results = {}
        for step in plan:
            agent = self.agents[step["agent"]]
            input_data = self.prepare_input(step, results)
            results[step["name"]] = agent.run(input_data)

        return self.synthesize(results)

    def plan(self, task: str) -> list:
        """Use Claude to plan workflow."""
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            system="You are a task planner. Break complex tasks into steps.",
            messages=[{
                "role": "user",
                "content": f"""Plan how to accomplish this task: {task}

Available agents:
- researcher: Find information
- coder: Write code
- reviewer: Review code quality

Return plan as JSON array of steps with agent and description."""
            }]
        )

        return json.loads(response.content[0].text)
```

---

## Delegation Pattern

```python
# Main agent delegates to specialized sub-agents
def delegate_to_expert(task_type: str, context: str) -> str:
    """Delegate task to appropriate expert agent."""

    expert_prompts = {
        "security": "You are a security expert...",
        "performance": "You are a performance engineer...",
        "architecture": "You are a solutions architect...",
    }

    # Use appropriate model for expert
    model = "claude-sonnet-4-20250514"
    if task_type == "architecture":
        model = "claude-opus-4-20250514"  # Complex reasoning

    response = client.messages.create(
        model=model,
        max_tokens=2048,
        system=expert_prompts[task_type],
        messages=[{"role": "user", "content": context}]
    )

    return response.content[0].text
```

---

## Stateful Agent Pattern

```python
class StatefulAgent:
    """Agent that maintains state across interactions."""

    def __init__(self):
        self.client = anthropic.Anthropic()
        self.conversation = []
        self.state = {
            "context": {},
            "completed_actions": [],
            "pending_actions": []
        }

    def interact(self, user_input: str) -> str:
        """Process user input with state awareness."""

        # Build context-aware prompt
        system_prompt = f"""You are a stateful assistant.

Current state:
- Context: {json.dumps(self.state['context'])}
- Completed: {len(self.state['completed_actions'])} actions
- Pending: {len(self.state['pending_actions'])} actions

Maintain continuity with previous interactions."""

        self.conversation.append({"role": "user", "content": user_input})

        response = self.client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=2048,
            system=system_prompt,
            messages=self.conversation
        )

        assistant_message = response.content[0].text
        self.conversation.append({"role": "assistant", "content": assistant_message})

        # Update state based on response
        self.update_state(response)

        return assistant_message

    def update_state(self, response):
        """Extract state updates from response."""
        # Parse structured state changes from response
        # This could use a tool for structured output
        pass
```

---

## Feedback Loop Pattern

```python
class FeedbackAgent:
    """Agent that refines output based on validation."""

    def run(self, task: str, max_iterations: int = 3) -> str:
        """Run task with self-correction loop."""

        attempt = 1
        while attempt <= max_iterations:
            # Generate output
            output = self.generate(task)

            # Validate
            validation = self.validate(output)

            if validation["valid"]:
                return output

            # Provide feedback for next attempt
            feedback = validation["feedback"]
            task = f"{task}\n\nPrevious attempt failed: {feedback}\nTry again."
            attempt += 1

        raise RuntimeError(f"Failed after {max_iterations} attempts")

    def generate(self, task: str) -> str:
        """Generate output."""
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=2048,
            messages=[{"role": "user", "content": task}]
        )
        return response.content[0].text

    def validate(self, output: str) -> dict:
        """Validate output."""
        # Use separate validation logic (schema, tests, etc.)
        pass
```

---

## ReAct Pattern (Reasoning + Acting)

```python
class ReActAgent:
    """Agent that alternates reasoning and action."""

    def run(self, task: str) -> str:
        """Execute task with ReAct pattern."""
        messages = [{
            "role": "user",
            "content": f"""Solve this task using ReAct pattern:

Task: {task}

Format:
Thought: [your reasoning]
Action: [action to take]
Observation: [result of action]
... (repeat until solved)
Final Answer: [solution]"""
        }]

        while True:
            response = self.client.messages.create(
                model="claude-sonnet-4-20250514",
                max_tokens=2048,
                tools=self.tools,
                messages=messages
            )

            # Check if final answer reached
            if "Final Answer:" in response.content[0].text:
                return self.extract_final_answer(response.content[0].text)

            # Execute actions
            messages.append({"role": "assistant", "content": response.content})

            # Add observations
            observations = self.execute_actions(response)
            messages.append({"role": "user", "content": observations})
```

---

**Last Updated:** 2026-02-16
