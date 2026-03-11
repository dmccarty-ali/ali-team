# ali-ai-development - Claude API Patterns Reference

Detailed Claude API implementation patterns. Consult when building Claude integrations.

---

## Basic Message API

```python
import anthropic

client = anthropic.Anthropic()

# Simple completion
message = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "Explain quantum computing in simple terms"}
    ]
)
print(message.content[0].text)
```

---

## System Prompts

```python
# System prompt sets behavior, personality, constraints
message = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    system="""You are a senior software architect reviewing code.

    Guidelines:
    - Be direct and constructive
    - Focus on security, performance, and maintainability
    - Provide specific, actionable feedback
    - Reference industry best practices
    """,
    messages=[
        {"role": "user", "content": "Review this authentication code: ..."}
    ]
)
```

---

## Streaming Responses

```python
# Stream for better UX on longer responses
with client.messages.stream(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Write a detailed analysis..."}]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
```

---

## Multi-Turn Conversations

```python
# Maintain conversation history
conversation = []

def chat(user_message: str) -> str:
    conversation.append({"role": "user", "content": user_message})

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        messages=conversation
    )

    assistant_message = response.content[0].text
    conversation.append({"role": "assistant", "content": assistant_message})

    return assistant_message
```

---

## Vision API (Multi-Modal)

```python
import base64

# Send image with text
with open("image.jpg", "rb") as f:
    image_data = base64.standard_b64encode(f.read()).decode("utf-8")

message = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    messages=[{
        "role": "user",
        "content": [
            {
                "type": "image",
                "source": {
                    "type": "base64",
                    "media_type": "image/jpeg",
                    "data": image_data
                }
            },
            {
                "type": "text",
                "text": "What's in this image?"
            }
        ]
    }]
)
```

---

## Document Analysis

```python
# Process PDFs and documents
import pypdf

def extract_pdf_text(pdf_path: str) -> str:
    """Extract text from PDF for Claude analysis."""
    reader = pypdf.PdfReader(pdf_path)
    text = ""
    for page in reader.pages:
        text += page.extract_text() + "\n\n"
    return text

# Send document to Claude
pdf_text = extract_pdf_text("report.pdf")

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=2048,
    messages=[{
        "role": "user",
        "content": f"""Analyze this document and provide a summary:

{pdf_text}

Focus on:
- Key findings
- Action items
- Risks identified"""
    }]
)
```

---

**Last Updated:** 2026-02-16
