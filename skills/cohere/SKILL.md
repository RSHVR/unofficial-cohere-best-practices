---
name: unofficial-cohere-best-practices
description: Unofficial best practices guide for Cohere's AI APIs. Use when working with Cohere models for chat/text generation (Command A, Command R+, Command R), embeddings (Embed v4, v3), reranking (Rerank v4, v3.5), streaming, structured outputs, RAG, tool use/function calling, or agents. Supports Python, TypeScript, Java, and Go SDKs, plus LangChain/LangGraph integrations. Triggers on mentions of Cohere API, Command models, CohereEmbeddings, ChatCohere, CohereRerank, cohere-ai, or any Cohere-related development task.
---

# Unofficial Cohere Best Practices

This skill provides patterns and code for building with Cohere's AI models across multiple languages: **Python**, **TypeScript**, **Java**, and **Go**, plus LangChain/LangGraph integrations.

## Prerequisites

**Cohere API Key Required** - Get your key at https://dashboard.cohere.com

Add to `~/.claude/settings.json`:
```json
{
  "env": {
    "CO_API_KEY": "your-api-key",
    "COHERE_API_KEY": "your-api-key"
  }
}
```

Restart Claude Code after adding your API key.

## Quick Reference

### Current Models (2025)

| Category | Model | Context | Notes |
|----------|-------|---------|-------|
| **Chat** | `command-a-03-2025` | 256K | Best overall, 111B params |
| | `command-r7b-12-2024` | 128K | Small/fast, 7B params |
| | `command-r-plus-08-2024` | 128K | Strong reasoning |
| | `command-r-08-2024` | 128K | Balanced |
| **Reasoning** | `command-a-reasoning-08-2025` | 256K | Extended thinking with `budget_tokens` |
| **Vision** | `command-a-vision-07-2025` | 128K | Images + text (charts, OCR, docs) |
| **Translate** | `command-a-translate-08-2025` | 8K | 23 languages |
| **Embed** | `embed-v4.0` | 128K | Multimodal, Matryoshka dims (256-1536) |
| | `embed-english-v3.0` | 512 | English, 1024 dims |
| | `embed-multilingual-v3.0` | 512 | 100+ languages, 1024 dims |
| **Rerank** | `rerank-v4.0-pro` | 32K | Best quality |
| | `rerank-v4.0-fast` | 32K | Speed optimized |
| | `rerank-v3.5` | 4K | Balanced |

### Reasoning Model
The `command-a-reasoning-08-2025` model supports controllable "thinking" for complex tasks:
```python
response = co.chat(
    model="command-a-reasoning-08-2025",
    messages=[{"role": "user", "content": "Complex problem..."}],
    thinking={"type": "enabled", "budget_tokens": 8000}
)
```
- Set `budget_tokens` to control depth (min 1024, higher = more thorough)
- Use `"type": "disabled"` for simple queries (lower latency)

### Installation

**Python:**
```bash
pip install cohere                    # Native SDK (v5+)
pip install langchain-cohere          # LangChain integration (v0.5+)
pip install langgraph                 # For agents
```

**TypeScript/JavaScript:**
```bash
npm install cohere-ai
```

**Java (Maven):**
```xml
<dependency>
  <groupId>com.cohere</groupId>
  <artifactId>cohere-java</artifactId>
  <version>1.x.x</version>
</dependency>
```

**Go:**
```bash
go get github.com/cohere-ai/cohere-go/v2
```

### Environment Setup
```bash
export CO_API_KEY="your-api-key"      # SDK auto-reads this
export COHERE_API_KEY="your-api-key"  # LangChain uses this
```

## Integration Selection Guide

| Use Case | Recommended Approach |
|----------|---------------------|
| Simple chat/completion | Native SDK `ClientV2` |
| Embeddings for vector DB | Native SDK or `CohereEmbeddings` |
| RAG pipeline | LangChain `CohereRagRetriever` |
| Tool use / Function calling | Native SDK (more control) or LangChain |
| Multi-step agents | LangGraph with `create_cohere_react_agent` |
| Simple tool workflows | Generic `create_react_agent` (provider-agnostic) |
| Reranking search results | Native SDK or `CohereRerank` |
| Structured JSON output | Native SDK with `response_format` |
| Image understanding | Native SDK with Vision model |

### Agent Choice: Cohere-Specific vs Generic
- **`create_cohere_react_agent`**: Use for complex multi-step tasks, Cohere-specific features (citations, connectors), better token efficiency on long reasoning chains
- **Generic `create_react_agent`**: Fine for simple tool-calling, consistent cross-provider behavior, when you're already getting good results

### LangChain Model Compatibility
> **Note**: Command A Reasoning and Command A Vision are **not yet supported** in LangChain. Use the native SDK for these models.

## Detailed Documentation

Based on your use case, read the appropriate reference file:

### By Language
- **Python SDK**: See [references/python-sdk.md](references/python-sdk.md)
- **TypeScript SDK**: See [references/typescript-sdk.md](references/typescript-sdk.md)
- **Java SDK**: See [references/java-sdk.md](references/java-sdk.md)
- **Go SDK**: See [references/go-sdk.md](references/go-sdk.md)

### By Feature
- **Embeddings**: See [references/embeddings.md](references/embeddings.md)
- **Reranking**: See [references/rerank.md](references/rerank.md)
- **Streaming**: See [references/streaming.md](references/streaming.md)
- **Structured Outputs**: See [references/structured-outputs.md](references/structured-outputs.md)

### Python Frameworks
- **LangChain integration**: See [references/langchain.md](references/langchain.md)
- **LangGraph agents**: See [references/langgraph.md](references/langgraph.md)

### Examples
- **Cookbooks & Examples**: See [references/cookbooks.md](references/cookbooks.md)

## Common Patterns

### Native SDK: Basic Chat
```python
import cohere
co = cohere.ClientV2()  # Reads CO_API_KEY from env

response = co.chat(
    model="command-a-03-2025",
    messages=[{"role": "user", "content": "Hello!"}]
)
print(response.message.content[0].text)
```

### Native SDK: Streaming
```python
for event in co.chat_stream(
    model="command-a-03-2025",
    messages=[{"role": "user", "content": "Write a poem"}]
):
    if event.type == "content-delta":
        print(event.delta.message.content.text, end="")
```

### Native SDK: Structured JSON Output
```python
response = co.chat(
    model="command-a-03-2025",
    messages=[{"role": "user", "content": "Extract: John is 30 years old"}],
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

### LangChain: ChatCohere
```python
from langchain_cohere import ChatCohere
from langchain_core.messages import HumanMessage

llm = ChatCohere(model="command-a-03-2025")
response = llm.invoke([HumanMessage(content="Hello!")])
print(response.content)
```

### LangGraph: ReAct Agent
```python
from langchain_cohere import ChatCohere, create_cohere_react_agent
from langchain.agents import AgentExecutor
from langchain_core.prompts import ChatPromptTemplate

llm = ChatCohere()
tools = [your_tools_here]
prompt = ChatPromptTemplate.from_template("{input}")

agent = create_cohere_react_agent(llm, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools, verbose=True)
result = executor.invoke({"input": "Your query"})
```

## API v1 vs v2 Notes

Cohere has migrated to API v2. Key differences:
- Use `ClientV2()` instead of `Client()`
- Tool calls now include IDs (`tool_call_id`)
- RAG documents use `documents` parameter with structured format
- Streaming events have new types (`content-delta`, `tool-plan-delta`, etc.)
- Tool results must use document format:
```python
messages.append({
    "role": "tool",
    "tool_call_id": tc.id,
    "content": [{"type": "document", "document": {"data": json.dumps(result)}}]
})
```

## Critical Gotchas

### 1. Embedding Input Types (MUST GET RIGHT)
Cohere uses **asymmetric embeddings**. Using wrong input_type = poor search results:
```python
# For STORING documents in vector DB
embedding = co.embed(texts=docs, input_type="search_document", ...)

# For QUERYING against stored documents
embedding = co.embed(texts=[query], input_type="search_query", ...)
```

### 2. Batch Limits
- Embeddings: **96 items per request** (API hard limit)
- Rerank: **1,000 documents recommended** (10K max but slower)

### 3. Tool Calling Differences
Cohere's tool calling format differs from Claude/OpenAI. LangChain/LangGraph handle this, but:
- Lower temperature (0.3) recommended for predictable tool calls
- System prompts may need Cohere-specific adjustments
- Watch for edge cases in complex multi-tool scenarios

### 4. Two-Stage Retrieval Pattern (Recommended)
```python
# Stage 1: Fast retrieval with embeddings (cast wide net)
candidates = vectorstore.similarity_search(query, k=30)

# Stage 2: Precise reranking (narrow down)
reranked = co.rerank(query=query, documents=candidates, top_n=10)
```

### 5. Text Truncation
Long texts get auto-truncated. Consider chunking at ~8000 chars for embeddings.

### 6. Embed v4 Dimensions
Use `output_dimension` parameter with Embed v4 for flexible sizing:
```python
co.embed(model="embed-v4.0", texts=texts, input_type="search_document", output_dimension=512)
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `ModuleNotFoundError: cohere` | `pip install cohere` |
| API key not found | Set `CO_API_KEY` env var or pass `api_key` param |
| LangChain import errors | Use `from langchain_cohere import ...` (not `langchain_community`) |
| Rate limits | Check dashboard for limits, implement exponential backoff |
| Tool use errors | Ensure tool results are JSON strings or list of document objects |
| Poor search results | Verify `input_type` matches use case (search_document vs search_query) |
| Agent not using tools correctly | Lower temperature to 0.3, check system prompt |
| Embed failed | Check API key, rate limits, batch size <= 96 |
| Reasoning model slow | Reduce `budget_tokens` or use `"type": "disabled"` |
| Vision model errors | Ensure image is base64 Data URL format |

## Quick Test Snippets
```python
# Test embedding
import cohere
co = cohere.ClientV2()
resp = co.embed(model="embed-v4.0", texts=["test"], input_type="search_query", embedding_types=["float"])
print(f"Dim: {len(resp.embeddings.float_[0])}")  # 1536

# Test LLM via LangChain
from langchain_cohere import ChatCohere
llm = ChatCohere(model="command-a-03-2025", temperature=0.3)
print(llm.invoke("Say hello").content)
```
