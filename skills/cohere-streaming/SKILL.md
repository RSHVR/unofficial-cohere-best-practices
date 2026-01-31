---
name: cohere-streaming
description: Cohere streaming reference for real-time text generation, tool use events, and RAG citations. Covers all stream event types and async streaming patterns.
---

# Cohere Streaming Reference

## Official Resources

- **Docs & Cookbooks**: https://github.com/cohere-ai/cohere-developer-experience
- **API Reference**: https://docs.cohere.com/reference/about

## Basic Streaming

```python
import cohere
co = cohere.ClientV2()

for event in co.chat_stream(
    model="command-a-03-2025",
    messages=[{"role": "user", "content": "Write a poem about AI"}]
):
    if event.type == "content-delta":
        print(event.delta.message.content.text, end="", flush=True)
```

### Async Streaming
```python
import asyncio

async_co = cohere.AsyncClientV2()

async def stream_response():
    async for event in async_co.chat_stream(
        model="command-a-03-2025",
        messages=[{"role": "user", "content": "Tell me a story"}]
    ):
        if event.type == "content-delta":
            print(event.delta.message.content.text, end="", flush=True)

asyncio.run(stream_response())
```

## Stream Event Types

| Event Type | Description | When Emitted |
|------------|-------------|--------------|
| `message-start` | Stream begins | First event |
| `content-start` | Content block begins | Before text generation |
| `content-delta` | Text chunk | Multiple times during generation |
| `content-end` | Content block ends | After text generation |
| `message-end` | Stream complete | Final event |
| `tool-plan-delta` | Tool planning text | When model plans tool use |
| `tool-call-start` | Tool call begins | Before each tool call |
| `tool-call-delta` | Tool call arguments | During tool call generation |
| `tool-call-end` | Tool call complete | After each tool call |
| `citation-generation` | Citation info | When citing documents |

## Handling All Event Types

```python
for event in co.chat_stream(
    model="command-a-03-2025",
    messages=[{"role": "user", "content": "Explain quantum computing"}]
):
    match event.type:
        case "message-start":
            print("Generation started...")
        case "content-delta":
            print(event.delta.message.content.text, end="", flush=True)
        case "message-end":
            print("\n--- Generation complete ---")
            final_response = event.response
```

## Collecting Full Response While Streaming

```python
full_text = []

for event in co.chat_stream(
    model="command-a-03-2025",
    messages=[{"role": "user", "content": "Write a haiku"}]
):
    if event.type == "content-delta":
        chunk = event.delta.message.content.text
        print(chunk, end="", flush=True)
        full_text.append(chunk)
    elif event.type == "message-end":
        final_response = event.response

complete_text = "".join(full_text)
```

## Tool Use Streaming

```python
for event in co.chat_stream(
    model="command-a-03-2025",
    messages=[{"role": "user", "content": "What's the weather in Tokyo?"}],
    tools=tools
):
    match event.type:
        case "tool-plan-delta":
            print(f"Planning: {event.delta.message.tool_plan}", end="")
        case "tool-call-start":
            print(f"\nTool call started")
        case "tool-call-delta":
            print(f"Args: {event.delta.message.tool_calls}", end="")
        case "tool-call-end":
            print("\nTool call complete")
        case "content-delta":
            print(event.delta.message.content.text, end="")
```

## RAG Citation Events

```python
documents = [
    {"id": "doc1", "data": {"title": "Report", "text": "Q3 revenue was $10M"}},
]

for event in co.chat_stream(
    model="command-a-03-2025",
    messages=[{"role": "user", "content": "What was Q3 revenue?"}],
    documents=documents
):
    match event.type:
        case "content-delta":
            print(event.delta.message.content.text, end="")
        case "citation-generation":
            citation = event.delta
            if citation:
                print(f"\n[Citation: {citation}]")
        case "message-end":
            if event.response.message.citations:
                print("\n\nSources:")
                for cite in event.response.message.citations:
                    print(f"  - '{cite.text}' from {cite.sources}")
```

## Streaming Chat UI Helper

```python
def stream_chat(messages: list, on_token=None, on_complete=None):
    full_text = []
    final_response = None

    for event in co.chat_stream(
        model="command-a-03-2025",
        messages=messages
    ):
        if event.type == "content-delta":
            chunk = event.delta.message.content.text
            full_text.append(chunk)
            if on_token:
                on_token(chunk)
        elif event.type == "message-end":
            final_response = event.response

    complete = "".join(full_text)
    if on_complete:
        on_complete(complete, final_response)

    return complete, final_response

# Usage
stream_chat(
    [{"role": "user", "content": "Tell me a joke"}],
    on_token=lambda t: print(t, end="", flush=True),
    on_complete=lambda text, resp: print(f"\n\n[Done: {len(text)} chars]")
)
```

## Error Handling in Streams

```python
from cohere.core import ApiError

def safe_stream(messages):
    try:
        for event in co.chat_stream(
            model="command-a-03-2025",
            messages=messages
        ):
            if event.type == "content-delta":
                yield event.delta.message.content.text
    except ApiError as e:
        print(f"API Error: {e.status_code} - {e.body}")
        yield f"[Error: {e.status_code}]"
    except Exception as e:
        print(f"Stream error: {e}")
        yield "[Stream interrupted]"

for chunk in safe_stream([{"role": "user", "content": "Hello"}]):
    print(chunk, end="")
```
