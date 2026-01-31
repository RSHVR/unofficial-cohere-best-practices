# CLAUDE.md

This file provides guidance to Claude Code when working with this repository.

## Project summary

Unofficial best practices for Cohere's AI APIs. Covers chat, embeddings, reranking, streaming, structured outputs, RAG, tool use, and agents. Supports both native Cohere Python SDK (`cohere.ClientV2`) and LangChain/LangGraph integrations.

## Quick reference

### Skill location

| Skill | Location | Triggers |
|-------|----------|----------|
| cohere | `skills/cohere/` | "Cohere", "Command A", "Command R", "embed", "rerank", "ChatCohere", "CohereEmbeddings" |

### Key models (2025)

| Use Case | Model | Notes |
|----------|-------|-------|
| Chat | `command-a-03-2025` | Best overall, 256K context |
| Reasoning | `command-a-reasoning-08-2025` | Use `thinking.budget_tokens` |
| Vision | `command-a-vision-07-2025` | Native SDK only |
| Embeddings | `embed-v4.0` | Multimodal, Matryoshka dims |
| Rerank | `rerank-v4.0-pro` | 32K context |

### Critical gotchas

1. **Embedding input types (MUST get right):**
   - `input_type="search_document"` for storing docs
   - `input_type="search_query"` for user queries
   - Mixing these = poor search results

2. **Batch limits:**
   - Embeddings: 96 items per request (hard limit)
   - Rerank: 1,000 recommended (10K max)

3. **API version:**
   - Use `ClientV2()` not `Client()`
   - Tool results need document format with `tool_call_id`

4. **LangChain compatibility:**
   - Reasoning and Vision models NOT supported in LangChain
   - Use native SDK for these

## Reference files

| File | Topics |
|------|--------|
| `python-sdk.md` | Chat, streaming, tool use, structured outputs, RAG |
| `embeddings.md` | Embed v4/v3, input types, batch processing |
| `rerank.md` | Reranking models, two-stage retrieval |
| `streaming.md` | Event types, async patterns |
| `structured-outputs.md` | JSON mode, schemas, strict_tools |
| `langchain.md` | ChatCohere, CohereEmbeddings, CohereRerank |
| `langgraph.md` | ReAct agents, memory, human-in-the-loop |
| `cookbooks.md` | Curated official cookbook links |

## Common tasks

### Basic chat
```python
import cohere
co = cohere.ClientV2()
response = co.chat(
    model="command-a-03-2025",
    messages=[{"role": "user", "content": "Hello!"}]
)
print(response.message.content[0].text)
```

### Embeddings
```python
# For STORING documents
co.embed(model="embed-v4.0", texts=docs, input_type="search_document")

# For QUERYING
co.embed(model="embed-v4.0", texts=[query], input_type="search_query")
```

### Two-stage retrieval
```python
# Stage 1: Fast retrieval
candidates = vectorstore.similarity_search(query, k=30)

# Stage 2: Precise reranking
reranked = co.rerank(model="rerank-v4.0-pro", query=query, documents=candidates, top_n=10)
```

## Dependencies

- `cohere` (v5+) - Native SDK
- `langchain-cohere` (v0.5+) - LangChain integration
- `langgraph` - For agents
