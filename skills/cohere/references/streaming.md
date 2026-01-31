# Cohere Streaming Reference

## Table of Contents
1. [Basic Streaming](#basic-streaming)
2. [Stream Event Types](#stream-event-types)
3. [Text Generation Events](#text-generation-events)
4. [Tool Use Events](#tool-use-events)
5. [RAG Events](#rag-events)
6. [Complete Examples](#complete-examples)

---

## Basic Streaming

Streaming allows you to display partial results as they're generated, reducing perceived latency.

### Simple Streaming
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
import cohere
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

---

## Stream Event Types

When streaming, the API sends events with different `type` values. Handle each appropriately:

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

---

## Text Generation Events

### Handling All Event Types
```python
for event in co.chat_stream(
    model="command-a-03-2025",
    messages=[{"role": "user", "content": "Explain quantum computing"}]
):
    match event.type:
        case "message-start":
            print("Generation started...")

        case "content-start":
            print("Content block started")

        case "content-delta":
            # Main text output
            print(event.delta.message.content.text, end="", flush=True)

        case "content-end":
            print("\nContent block ended")

        case "message-end":
            print("\n--- Generation complete ---")
            # Access final response object
            final_response = event.response
            print(f"Finish reason: {event.delta.finish_reason}")
```

### Collecting Full Response While Streaming
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
print(f"\n\nFull response ({len(complete_text)} chars): {complete_text}")
```

---

## Tool Use Events

When tools are involved, additional event types are emitted:

### Tool Planning Events
```python
for event in co.chat_stream(
    model="command-a-03-2025",
    messages=[{"role": "user", "content": "What's the weather in Tokyo?"}],
    tools=tools
):
    match event.type:
        case "tool-plan-delta":
            # Model's reasoning about which tools to use
            print(f"Planning: {event.delta.message.tool_plan}", end="")

        case "tool-call-start":
            # Tool call initiated
            tool_call = event.delta.message.tool_calls
            print(f"\nTool call started: {tool_call}")

        case "tool-call-delta":
            # Tool arguments being generated
            print(f"Args: {event.delta.message.tool_calls}", end="")

        case "tool-call-end":
            print("\nTool call complete")

        case "content-delta":
            print(event.delta.message.content.text, end="")
```

### Complete Tool Use with Streaming
```python
import cohere
import json

co = cohere.ClientV2()

def get_weather(location: str) -> dict:
    return {"temperature": "22°C", "condition": "sunny"}

tools = [{
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": "Get weather for a location",
        "parameters": {
            "type": "object",
            "properties": {
                "location": {"type": "string", "description": "City name"}
            },
            "required": ["location"]
        }
    }
}]

functions_map = {"get_weather": get_weather}

def stream_with_tools(user_message: str):
    messages = [{"role": "user", "content": user_message}]

    while True:
        tool_calls_buffer = []
        tool_plan = []

        for event in co.chat_stream(
            model="command-a-03-2025",
            messages=messages,
            tools=tools
        ):
            match event.type:
                case "tool-plan-delta":
                    tool_plan.append(event.delta.message.tool_plan or "")

                case "tool-call-start":
                    tool_calls_buffer.append({
                        "id": None,
                        "name": None,
                        "args": []
                    })

                case "tool-call-delta":
                    tc = event.delta.message.tool_calls
                    if tc:
                        current = tool_calls_buffer[-1]
                        if hasattr(tc, 'id') and tc.id:
                            current["id"] = tc.id
                        if hasattr(tc, 'function'):
                            if tc.function.name:
                                current["name"] = tc.function.name
                            if tc.function.arguments:
                                current["args"].append(tc.function.arguments)

                case "content-delta":
                    print(event.delta.message.content.text, end="", flush=True)

                case "message-end":
                    response = event.response

        # Check if we have tool calls to execute
        if response.message.tool_calls:
            messages.append({
                "role": "assistant",
                "tool_plan": "".join(tool_plan),
                "tool_calls": response.message.tool_calls
            })

            # Execute tools
            for tc in response.message.tool_calls:
                args = json.loads(tc.function.arguments)
                result = functions_map[tc.function.name](**args)
                messages.append({
                    "role": "tool",
                    "tool_call_id": tc.id,
                    "content": [{"type": "document", "document": {"data": json.dumps(result)}}]
                })
        else:
            # No more tool calls, we're done
            break

    print()  # Final newline

stream_with_tools("What's the weather in Tokyo?")
```

---

## RAG Events

When using RAG with connectors or documents, additional events provide citation information:

### Citation Events
```python
documents = [
    {"id": "doc1", "data": {"title": "Report", "text": "Q3 revenue was $10M"}},
    {"id": "doc2", "data": {"title": "Summary", "text": "Growth rate: 15%"}}
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
            # Citations link response text to source documents
            citation = event.delta
            if citation:
                print(f"\n[Citation: {citation}]")

        case "message-end":
            # Final citations in response
            if event.response.message.citations:
                print("\n\nSources:")
                for cite in event.response.message.citations:
                    print(f"  - '{cite.text}' from {cite.sources}")
```

---

## Complete Examples

### Streaming Chat UI Helper
```python
def stream_chat(messages: list, on_token=None, on_complete=None):
    """
    Stream a chat response with callbacks.

    Args:
        messages: List of message dicts
        on_token: Callback for each token (str) -> None
        on_complete: Callback when done (full_text, response) -> None
    """
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
def print_token(t):
    print(t, end="", flush=True)

def done(text, resp):
    print(f"\n\n[Done: {len(text)} chars]")

stream_chat(
    [{"role": "user", "content": "Tell me a joke"}],
    on_token=print_token,
    on_complete=done
)
```

### Streaming with Timeout
```python
import signal

class StreamTimeout(Exception):
    pass

def timeout_handler(signum, frame):
    raise StreamTimeout("Stream timed out")

def stream_with_timeout(messages, timeout_seconds=30):
    signal.signal(signal.SIGALRM, timeout_handler)
    signal.alarm(timeout_seconds)

    try:
        for event in co.chat_stream(
            model="command-a-03-2025",
            messages=messages
        ):
            if event.type == "content-delta":
                print(event.delta.message.content.text, end="")
        print()
    except StreamTimeout:
        print("\n[Stream timed out]")
    finally:
        signal.alarm(0)  # Cancel alarm
```

### Error Handling in Streams
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

# Usage
for chunk in safe_stream([{"role": "user", "content": "Hello"}]):
    print(chunk, end="")
```
