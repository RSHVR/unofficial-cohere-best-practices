# Cohere Native Python SDK Reference

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

### Token Budget Control
```python
# Low budget = faster, less thorough
response = co.chat(
    model="command-a-reasoning-2025",
    messages=messages,
    thinking={"type": "enabled", "budget_tokens": 1024}  # Minimum budget
)

# High budget = slower, more thorough
response = co.chat(
    model="command-a-reasoning-2025", 
    messages=messages,
    thinking={"type": "enabled", "budget_tokens": 16000}  # Deep reasoning
)
```

### Access Thinking Output
```python
response = co.chat(
    model="command-a-reasoning-2025",
    messages=[{"role": "user", "content": "Complex reasoning task..."}],
    thinking={"type": "enabled", "budget_tokens": 8000}
)

# The thinking process may be accessible depending on API version
# Check response structure for thinking content
for content in response.message.content:
    if hasattr(content, 'type'):
        print(f"Type: {content.type}")
    print(content.text)
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
        case "content-start":
            print("Content block started")
        case "content-delta":
            print(event.delta.message.content.text, end="")
        case "content-end":
            print("\nContent block ended")
        case "message-end":
            print("Generation complete")
            # Access final response: event.response
        case "tool-plan-delta":
            print(f"Tool plan: {event.delta.message.tool_plan}")
        case "tool-call-start":
            print(f"Tool call started: {event.delta.message.tool_calls}")
        case "tool-call-delta":
            print(f"Tool args: {event.delta.message.tool_calls}")
        case "tool-call-end":
            print("Tool call complete")
```

## Tool Use / Function Calling

### Step 1: Define Tools
```python
# The actual function
def get_weather(location: str) -> dict:
    # Your implementation here
    return {"temperature": "20°C", "condition": "sunny"}

# Map function names to functions
functions_map = {"get_weather": get_weather}

# Tool schema for the API
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

### Step 2: Generate Tool Calls
```python
import json

messages = [{"role": "user", "content": "What's the weather in Toronto?"}]

response = co.chat(
    model="command-a-03-2025",
    messages=messages,
    tools=tools
)

# Check if model wants to call tools
if response.message.tool_calls:
    # Add assistant message with tool calls
    messages.append({
        "role": "assistant",
        "tool_plan": response.message.tool_plan,
        "tool_calls": response.message.tool_calls
    })
```

### Step 3: Execute Tools and Return Results
```python
for tc in response.message.tool_calls:
    # Execute the function
    args = json.loads(tc.function.arguments)
    result = functions_map[tc.function.name](**args)
    
    # Format result as document
    messages.append({
        "role": "tool",
        "tool_call_id": tc.id,
        "content": [
            {
                "type": "document",
                "document": {"data": json.dumps(result)}
            }
        ]
    })
```

### Step 4: Generate Final Response
```python
final_response = co.chat(
    model="command-a-03-2025",
    messages=messages,
    tools=tools
)
print(final_response.message.content[0].text)

# Citations are included automatically
for citation in final_response.message.citations or []:
    print(f"Citation: '{citation.text}' from {citation.sources}")
```

### Controlling Tool Behavior with tool_choice
```python
# Force the model to use tools (never respond directly)
response = co.chat(
    model="command-a-03-2025",
    messages=messages,
    tools=tools,
    tool_choice="REQUIRED"  # Must call at least one tool
)

# Force the model to NOT use tools (respond directly)
response = co.chat(
    model="command-a-03-2025",
    messages=messages,
    tools=tools,
    tool_choice="NONE"  # Ignore tools, respond directly
)

# Let model decide (default behavior)
response = co.chat(
    model="command-a-03-2025",
    messages=messages,
    tools=tools,
    tool_choice="AUTO"  # Model decides whether to use tools
)
```

### Complete Tool Use Example
```python
import cohere
import json

co = cohere.ClientV2()

def search_database(query: str) -> list:
    return [
        {"id": 1, "title": "Result 1", "content": "..."},
        {"id": 2, "title": "Result 2", "content": "..."}
    ]

tools = [{
    "type": "function",
    "function": {
        "name": "search_database",
        "description": "Search internal database",
        "parameters": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "Search query"}
            },
            "required": ["query"]
        }
    }
}]

def run_tool_use(user_message: str):
    messages = [{"role": "user", "content": user_message}]
    
    response = co.chat(model="command-a-03-2025", messages=messages, tools=tools)
    
    while response.message.tool_calls:
        messages.append({
            "role": "assistant",
            "tool_plan": response.message.tool_plan,
            "tool_calls": response.message.tool_calls
        })
        
        for tc in response.message.tool_calls:
            args = json.loads(tc.function.arguments)
            result = search_database(**args)
            messages.append({
                "role": "tool",
                "tool_call_id": tc.id,
                "content": [{"type": "document", "document": {"data": json.dumps(result)}}]
            })
        
        response = co.chat(model="command-a-03-2025", messages=messages, tools=tools)
    
    return response.message.content[0].text
```

## Multi-step Tool Use (Agents)

For complex queries, the model may need multiple tool calls in sequence:

```python
def run_agent(user_message: str, max_steps: int = 10):
    messages = [{"role": "user", "content": user_message}]
    
    for step in range(max_steps):
        response = co.chat(model="command-a-03-2025", messages=messages, tools=tools)
        
        if not response.message.tool_calls:
            # No more tool calls, return final response
            return response.message.content[0].text
        
        # Process tool calls
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
    
    return "Max steps reached"
```

## Structured Outputs

### JSON Mode
```python
response = co.chat(
    model="command-a-03-2025",
    messages=[{"role": "user", "content": "List 3 fruits as JSON"}],
    response_format={"type": "json_object"}
)
# Response will be valid JSON
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
# Guarantees tool calls match schema exactly (no hallucinated params)
response = co.chat(
    model="command-a-03-2025",
    messages=[{"role": "user", "content": "..."}],
    tools=tools,
    strict_tools=True  # Eliminates tool name/param hallucinations
)
# Note: strict_tools requires at least one required parameter per tool
# Max 200 fields across all tools in a single call
```

### When to Use What
| Feature | Use Case |
|---------|----------|
| `response_format={"type": "json_object"}` | Simple JSON output, model determines structure |
| `response_format` with `json_schema` | Guaranteed schema compliance for text generation |
| `strict_tools=True` | Guaranteed schema compliance for tool/function calls |

## RAG with Documents

### Inline Documents
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

# Citations reference document IDs
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
