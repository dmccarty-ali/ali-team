# Nested Sessions: Invoking Claude Code from Within Claude Code

When Claude Code invokes another `claude` CLI subprocess from within an active session, three conflict layers cause the child process to exit immediately with code 1. This reference documents the causes and the official solution.

---

## The Problem

Running `claude -p` (or any `claude` subprocess) from inside an active Claude Code session — for example, from a hook, a LangGraph node, or an agent tool call — triggers three simultaneous conflicts:

**Layer 1: CLAUDECODE environment variable**

The parent session sets `CLAUDECODE=1` in its environment. Child processes inherit this variable. The child CLI detects `CLAUDECODE=1` and treats itself as a nested session, activating conflict-avoidance behavior that causes it to exit immediately.

**Layer 2: TMPDIR socket contention**

The parent session creates a socket file under the system TMPDIR for its IPC channel. A child CLI process attempts to use the same TMPDIR path for its own socket. The two processes contend for the same socket location, causing the child to fail on startup.

**Layer 3: ANTHROPIC_AUTH_TOKEN leak**

If `ANTHROPIC_AUTH_TOKEN` is set in the parent environment and leaks to the child process with a stale or session-scoped value, the child receives a 401 Unauthorized on its first API call.

**Symptom:** The subprocess exits with code 1 in approximately 3 seconds. There is no stdout output and no error message. The silent failure makes the root cause difficult to diagnose.

**Reference:** GitHub Issue #573 in anthropics/claude-agent-sdk-python documents this behavior and the official workaround.

---

## The Solution: claude-agent-sdk

The `claude-agent-sdk` Python package handles all three conflict layers natively. It is the correct tool for invoking Claude from within an active Claude Code session.

**Install:**

```bash
pip install claude-agent-sdk==0.1.44
```

**Core pattern (text generation, no file access):**

```python
from claude_agent_sdk import ClaudeAgentOptions, query, AssistantMessage
from claude_agent_sdk._errors import ClaudeSDKError

options = ClaudeAgentOptions(
    allowed_tools=[],
    permission_mode="bypassPermissions",
    env={
        "CLAUDECODE": "",  # Required: clears nested session signal
        # CLAUDE_CONFIG_DIR intentionally omitted — subprocess inherits project credentials
        # Set to "" only to force global ~/.claude/ credentials (rare — see CLAUDE_CONFIG_DIR section below)
    },
    max_turns=1,
)

async def _invoke_claude_code(prompt: str) -> str:
    result_text = ""
    try:
        async for message in query(prompt=prompt, options=options):
            if isinstance(message, AssistantMessage):
                # Consume content FIRST — error check comes after
                for block in message.content:
                    if hasattr(block, "text"):
                        result_text += block.text  # accumulate, don't overwrite
                # billing_error and other advisory errors can co-occur with valid
                # content on the same AssistantMessage. Only raise if no content
                # was produced. If content was produced, log a warning and continue.
                if message.error is not None:
                    error_type = (
                        message.error
                        if isinstance(message.error, str)
                        else getattr(message.error, "type", str(message.error))
                    )
                    if not result_text:
                        raise AIProviderError(
                            f"Claude returned error with no content: {error_type}"
                        )
                    else:
                        # Advisory error — content is usable, log for observability
                        logger.warning(
                            "assistant_error_with_content",
                            error_type=error_type,
                        )
    except ClaudeSDKError as e:
        raise AIProviderError(f"SDK error: {e}") from e

    if not result_text.strip():
        raise AIProviderError("Claude returned empty response")

    return result_text
```

**File-read variant (when the invoked Claude needs to read files):**

```python
options = ClaudeAgentOptions(
    allowed_tools=["Read"],
    permission_mode="bypassPermissions",
    env={
        "CLAUDECODE": "",
        # CLAUDE_CONFIG_DIR intentionally omitted
    },
    max_turns=3,
)
```

---

## Why Each Option Matters

| Option | Value | Why |
|--------|-------|-----|
| `env={"CLAUDECODE": ""}` | Empty string (not deleted) | Strips the nested session signal so the child CLI starts clean. An empty string overrides the inherited value without requiring the key to be absent. |
| `permission_mode="bypassPermissions"` | `bypassPermissions` | Without this, a headless subprocess prompts interactively for permission approval. There is no TTY available, so the subprocess exits immediately with code 1. |
| `max_turns=1` | `1` | Prevents runaway back-and-forth. Extraction and generation tasks are single-turn. Use a higher value only when the invoked Claude needs multiple tool calls. |
| `allowed_tools=[]` | Empty list | Minimal permission surface. Add `"Read"` only when the invoked Claude needs file access. Never grant Write or Bash unless explicitly required. |

---

## CLAUDE_CONFIG_DIR: Inherit or Clear?

This is a critical configuration decision. The wrong choice causes `billing_error` or authentication failure.

### The ali-ai launcher context

When a project is launched via `claude-ali`, the launcher sets `CLAUDE_CONFIG_DIR` to a project-specific config directory (e.g., `.claude-config/projects/...`). This directory contains the live session's OAuth credentials.

### Default behavior — inherit project credentials (correct for embedded services)

Do NOT add `CLAUDE_CONFIG_DIR` to the `env` dict. The subprocess will inherit `CLAUDE_CONFIG_DIR` from the parent environment and authenticate using the same project OAuth token as the parent session. This is the correct behavior for LangGraph nodes, extraction services, and generation workflows running inside a project session.

```python
env={
    "CLAUDECODE": "",  # Required: clears nested session signal
    # CLAUDE_CONFIG_DIR intentionally omitted — subprocess inherits project credentials
}
```

### When to clear (rare)

Only add `"CLAUDE_CONFIG_DIR": ""` if you explicitly want the subprocess to fall back to global `~/.claude/` credentials. Setting this incorrectly causes `billing_error` if the global credentials are from a different account, expired, or belong to a personal subscription that does not cover the invocation.

```python
env={
    "CLAUDECODE": "",
    "CLAUDE_CONFIG_DIR": "",  # Forces fallback to ~/.claude/ — use only when global creds are explicitly needed
}
```

### SDK env merges, does not replace

`ClaudeAgentOptions.env` merges with `os.environ` — it does not replace it. You cannot remove a variable by omitting it from `env={}`. To neutralize a var, you must set it to `""` explicitly. Verify that the Claude CLI handles `""` gracefully for each variable before relying on this — setting to `""` is not always equivalent to unset.

### billing_error diagnostic

If `billing_error` appears on `AssistantMessage.error`:

1. Check whether content was produced alongside the error. If `result_text` is non-empty after the loop: the error is advisory. The subprocess ran successfully; the error reflects an account quota condition. Check your error-ordering pattern — ensure content is consumed before the error check.
2. If `result_text` is empty and the parent session is running under project-specific credentials (launched via `claude-ali`): check whether `CLAUDE_CONFIG_DIR` is being cleared unnecessarily. Removing it from the env dict restores project credential inheritance.

---

## Critical: Response Validation

The SDK returns an async iterator. If no `AssistantMessage` is yielded, the result is silently empty — no exception is raised. Validation after the loop is mandatory.

**Rules:**

1. **Always validate after the loop.** If `result_text.strip()` is empty, raise an error — do not return an empty string silently.
2. **Always check `AssistantMessage.error` AFTER consuming all content blocks, not before.** `billing_error` and other advisory errors can co-occur with valid content on the same `AssistantMessage`. Checking before iterating discards the content and triggers a spurious error raise.
3. **Use `result_text +=` not `result_text =`.** The SDK may yield an `AssistantMessage` with multiple content blocks. Overwriting instead of accumulating truncates all but the last block.
4. **Do NOT use `ResultMessage.result` to overwrite `AssistantMessage` content.** `ResultMessage` is a different message type; mixing them corrupts the accumulated result.

### AssistantMessage.error Values

| Error Value | Cause | Behavior |
|-------------|-------|----------|
| `"authentication_failed"` | Auth token issue in child process | Fatal if no content |
| `"billing_error"` | Billing or quota issue — can be advisory | Log warning if content present; raise only if no content |
| `"rate_limit"` | Rate limiting applied | Fatal if no content |
| `"invalid_request"` | Bad prompt or options | Fatal if no content |
| `"server_error"` | Anthropic service error | Fatal if no content |
| `"unknown"` | Unclassified error | Fatal if no content |

Always raise with the specific error value, not a generic message. This makes log triage possible.

### Advisory vs Fatal Errors

`billing_error` is the primary observed case where `AssistantMessage.error` is set alongside valid content. It reflects a genuine account quota or billing condition — the OAuth account is at or near its limit. The CLI still produces a response (content is present), but records the billing state on the message. Treating it as fatal causes spurious failures; if content is present, use it and log the condition for operational visibility.

The rule: **error + no content = raise. Error + content = log warning and return content.**

This distinction was discovered in Tax AI Chat during SDK migration (2026-02-26). An initial fix placed the error check before content iteration, causing 8 chat flow E2E regressions because every first invocation produced `billing_error` alongside valid content — raising before reading the content discarded the response entirely. Moving the check after content iteration resolved all regressions.

---

## Anti-Pattern: Manual subprocess.Popen

**DANGER:** Do not attempt to invoke `claude` via `subprocess.Popen` with manual environment stripping.

```python
# WRONG — do not do this
import subprocess
import os

env = os.environ.copy()
del env["CLAUDECODE"]  # Insufficient — TMPDIR contention and auth token leak persist

result = subprocess.Popen(
    ["claude", "-p", "--output-format", "text", prompt],
    env=env,
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
)
stdout, stderr = result.communicate()
# result.returncode == 1, stdout == b"", no diagnostic output
```

Manual env stripping addresses only Layer 1 (the CLAUDECODE variable). It does not resolve the TMPDIR socket contention (Layer 2) or the auth token leak risk (Layer 3). The subprocess will still fail with exit code 1 in most environments.

`claude-agent-sdk` handles all three layers correctly. Use it.

---

## Integration Pattern (LangGraph / sync nodes)

LangGraph nodes are synchronous `def` functions. The `query()` function from `claude_agent_sdk` is an async generator. `asyncio.run()` is insufficient as a bridge because it closes the event loop before the SDK's internal async generator has finalized, cutting the subprocess stdout pipe early. This produces silent truncation — response completeness depends on timing.

Use a wrapper that explicitly calls `loop.shutdown_asyncgens()` before closing the loop:

```python
def _invoke_ai_sync(coro):
    """
    Sync wrapper for SDK async invocation.

    shutdown_asyncgens() is required — forces the SDK's internal async
    generator to finalize before loop.close(), preventing premature pipe
    cutoff. Plain asyncio.run() is insufficient.
    """
    loop = asyncio.new_event_loop()
    try:
        return loop.run_until_complete(coro)
    finally:
        loop.run_until_complete(loop.shutdown_asyncgens())
        loop.close()


# Usage in a LangGraph node
def my_langgraph_node(state: dict) -> dict:
    prompt = f"Summarize the following: {state['input']}"
    output = _invoke_ai_sync(_invoke_claude_code(prompt))
    return {**state, "summary": output}
```

**Important:** Do not use `asyncio.to_thread()` inside a LangGraph node to call the async SDK. That pattern creates a nested event loop context that can deadlock.

**Note:** The message `Loop ... is closed` appearing in test output is expected behavior. It confirms the loop was correctly finalized — it is not an error.

---

## Mocking in Unit Tests

Mock `claude_agent_sdk.query`, not `subprocess.Popen`. The implementation no longer uses Popen, so mocking Popen will not intercept calls.

```python
from unittest.mock import MagicMock, patch
from claude_agent_sdk import AssistantMessage


def make_async_gen(messages):
    async def _gen():
        for msg in messages:
            yield msg
    return _gen()


@patch("src.services.ai_service.query")
def test_invoke_claude_code(mock_query):
    mock_message = MagicMock(spec=AssistantMessage)
    mock_message.error = None  # Must be set explicitly — unset MagicMock attributes are truthy
    mock_message.content = [MagicMock(text="extracted result")]
    mock_query.return_value = make_async_gen([mock_message])

    result = _invoke_ai_sync(_invoke_claude_code("test prompt"))
    assert result == "extracted result"
```

**Key points:**

- Use `make_async_gen()` factory — `query()` is an async generator, not a coroutine. Mocking it as a plain coroutine or `AsyncMock()` will cause `TypeError` in the `async for` loop.
- Set `mock_message.error = None` explicitly. If left as a MagicMock attribute (truthy by default), error-handling branches will behave incorrectly in tests that verify the non-error path.
- Patch at the import location used by the module under test (`src.services.ai_service.query` if imported as `from claude_agent_sdk import query`).
- Assert on `ClaudeAgentOptions` fields to verify the correct safety options are always passed.

---

## See Also

`references/cli-reference.md` — Batch Mode section, **Gotcha 3** covers the high-level warning against running `claude -p` inside an active session. This reference file provides the full analysis, conflict layer breakdown, and `claude-agent-sdk` solution.

---

**Document Version:** 1.1 / **Last Updated:** 2026-02-26 / **Maintained By:** ALI AI Team
