---
name: cohere-python-sdk
description: Cohere Python SDK reference for chat, streaming, tool use, structured outputs, and RAG. Use when building Python applications with Cohere's Command models, embeddings, or reranking APIs.
---

# Cohere Native Python SDK Reference

## Official Resources

- **Docs & Cookbooks**: https://github.com/cohere-ai/cohere-developer-experience
- **API Reference**: https://docs.cohere.com/reference/about

## Table of Contents
1. [Client Setup](#client-setup)
2. [Chat API](#chat-api)
3. [Streaming](#streaming)
4. [Tool Use / Function Calling](#tool-use--function-calling)
5. [Multi-step Tool Use (Agents)](#multi-step-tool-use-agents)
6. [Structured Outputs](#structured-outputs)
7. [RAG with Documents](#rag-with-documents)
8. [Safety Modes](#safety-modes)

## Client Setup

### Basic Setup
```python
import cohere

# Option 1: Auto-read from CO_API_KEY env var
co = cohere.ClientV2()

# Option 2: Explicit API key
co = cohere.ClientV2(api_key="your-api-key")

# Option 3: Custom endpoint (private deployment)
co = cohere.ClientV2(
    api_key="your-api-key",
    base_url="https://your-deployment.com"
)
```

### Async Client
```python
import cohere

async_co = cohere.AsyncClientV2()

async def main():
    response = await async_co.chat(
        model="command-a-03-2025",
        messages=[{"role": "user", "content": "Hello!"}]
    )
    print(response.message.content[0].text)
```

## Chat API

### Basic Chat
```python
response = co.chat(
    model="command-a-03-2025",
    messages=[
        {"role": "user", "content": "What is machine learning?"}
    ]
)
print(response.message.content[0].text)
```

### With System Message
```python
response = co.chat(
    model="command-a-03-2025",
    messages=[
        {"role": "system", "content": "You are a helpful coding assistant."},
        {"role": "user", "content": "Write a Python hello world"}
    ]
)
```

### Multi-turn Conversation
```python
messages = [
    {"role": "user", "content": "My name is Veer"},
    {"role": "assistant", "content": "Nice to meet you, Veer!"},
    {"role": "user", "content": "What's my name?"}
]
response = co.chat(model="command-a-03-2025", messages=messages)
```

### Parameters
```python
response = co.chat(
    model="command-a-03-2025",
    messages=[{"role": "user", "content": "Write a story"}],
    temperature=0.7,           # 0.0-1.0, higher = more creative
    max_tokens=500,            # Max response length
    p=0.9,                     # Top-p sampling
    k=50,                      # Top-k sampling
    seed=42,                   # For reproducibility
    stop_sequences=["END"],    # Stop generation at these
)
```

## Reasoning Model (Command A Reasoning)

The `command-a-reasoning-2025` model includes extended thinking capabilities with controllable token budgets:

### Basic Usage
```python
response = co.chat(
    model="command-a-reasoning-2025",
    messages=[{"role": "user", "content": "Solve this step by step: What is 15% of 340?"}],
    thinking={
        "type": "enabled",
        "budget_tokens": 5000  # Max tokens for internal reasoning
    }
)
print(response.message.content[0].text)
```

### Disable Reasoning (Lower Latency)
```python
response = co.chat(
    model="command-a-reasoning-2025",
    messages=[{"role": "user", "content": "Quick question: capital of France?"}],
    thinking={"type": "disabled"}  # Skip reasoning for simple queries
)
```

## Streaming

### Basic Streaming
```python
response = co.chat_stream(
    model="command-a-03-2025",
    messages=[{"role": "user", "content": "Write a poem about AI"}]
)

for event in response:
    if event.type == "content-delta":
        print(event.delta.message.content.text, end="", flush=True)
```

### Streaming Event Types
```python
for event in co.chat_stream(model="command-a-03-2025", messages=messages):
    match event.type:
        case "message-start":
            print("Generation started")
        case "content-delta":
            print(event.delta.message.content.text, end="")
        case "message-end":
            print("Generation complete")
        case "tool-plan-delta":
            print(f"Tool plan: {event.delta.message.tool_plan}")
        case "tool-call-start":
            print(f"Tool call started: {event.delta.message.tool_calls}")
```

## Tool Use / Function Calling

### Step 1: Define Tools
```python
def get_weather(location: str) -> dict:
    return {"temperature": "20°C", "condition": "sunny"}

functions_map = {"get_weather": get_weather}

tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "Get current weather for a location",
            "parameters": {
                "type": "object",
                "properties": {
                    "location": {
                        "type": "string",
                        "description": "City name, e.g., 'Toronto'"
                    }
                },
                "required": ["location"]
            }
        }
    }
]
```

### Step 2: Generate and Execute Tool Calls
```python
import json

messages = [{"role": "user", "content": "What's the weather in Toronto?"}]

response = co.chat(model="command-a-03-2025", messages=messages, tools=tools)

if response.message.tool_calls:
    messages.append({
        "role": "assistant",
        "tool_plan": response.message.tool_plan,
        "tool_calls": response.message.tool_calls
    })

    for tc in response.message.tool_calls:
        args = json.loads(tc.function.arguments)
        result = functions_map[tc.function.name](**args)
        messages.append({
            "role": "tool",
            "tool_call_id": tc.id,
            "content": [{"type": "document", "document": {"data": json.dumps(result)}}]
        })

final_response = co.chat(model="command-a-03-2025", messages=messages, tools=tools)
print(final_response.message.content[0].text)
```

### Controlling Tool Behavior
```python
response = co.chat(
    model="command-a-03-2025",
    messages=messages,
    tools=tools,
    tool_choice="REQUIRED"  # Must call tool. Options: AUTO, REQUIRED, NONE
)
```

## Structured Outputs

### JSON Mode
```python
response = co.chat(
    model="command-a-03-2025",
    messages=[{"role": "user", "content": "List 3 fruits as JSON"}],
    response_format={"type": "json_object"}
)
```

### JSON Schema
```python
response = co.chat(
    model="command-a-03-2025",
    messages=[{"role": "user", "content": "Extract person info from: John is 30"}],
    response_format={
        "type": "json_object",
        "json_schema": {
            "type": "object",
            "properties": {
                "name": {"type": "string"},
                "age": {"type": "integer"}
            },
            "required": ["name", "age"]
        }
    }
)
```

### Strict Tool Parameters
```python
response = co.chat(
    model="command-a-03-2025",
    messages=[{"role": "user", "content": "..."}],
    tools=tools,
    strict_tools=True  # Eliminates tool name/param hallucinations
)
```

## RAG with Documents

```python
documents = [
    {"id": "doc1", "data": {"title": "Report", "text": "Q3 revenue was $10M"}},
    {"id": "doc2", "data": {"title": "Summary", "text": "Growth rate: 15%"}}
]

response = co.chat(
    model="command-a-03-2025",
    messages=[{"role": "user", "content": "What was Q3 revenue?"}],
    documents=documents
)

for citation in response.message.citations or []:
    print(f"'{citation.text}' cited from {citation.sources}")
```

## Safety Modes

```python
response = co.chat(
    model="command-a-03-2025",
    messages=[{"role": "user", "content": "..."}],
    safety_mode="CONTEXTUAL"  # Default, or "STRICT" or "OFF"
)
```

## Error Handling

```python
from cohere.core import ApiError

try:
    response = co.chat(model="command-a-03-2025", messages=messages)
except ApiError as e:
    print(f"API Error: {e.status_code} - {e.body}")
except Exception as e:
    print(f"Error: {e}")
```
