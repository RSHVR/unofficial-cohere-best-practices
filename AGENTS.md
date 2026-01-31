# AGENTS.md

This file provides guidance to AI coding agents (Claude Code, Cursor, Copilot, Codex, etc.) when working with code that uses Cohere's APIs.

## Project overview

Unofficial Cohere Best Practices is a collection of Agent Skills for building applications with Cohere's AI APIs. Covers chat/text generation, embeddings, reranking, streaming, structured outputs, RAG, tool use, and agents.

## Directory structure

```
unofficial-cohere-best-practices/
├── README.md                    # Project documentation
├── AGENTS.md                    # This file
├── CLAUDE.md                    # Claude-specific guidance
├── LICENSE                      # MIT License
├── .claude-plugin/
│   └── plugin.json              # Claude Code plugin manifest
└── skills/
    └── cohere/
        ├── SKILL.md             # Main skill file
        └── references/
            ├── native-sdk.md    # Chat, streaming, tool use
            ├── embeddings.md    # Embed v4/v3, input types
            ├── rerank.md        # Reranking, two-stage retrieval
            ├── streaming.md     # Event types, patterns
            ├── structured-outputs.md  # JSON mode, schemas
            ├── langchain.md     # LangChain integration
            ├── langgraph.md     # Agents with LangGraph
            └── cookbooks.md     # Official cookbook links
```

## Key models (2025)

| Category | Model | Context | Notes |
|----------|-------|---------|-------|
| Chat | `command-a-03-2025` | 256K | Best overall |
| Reasoning | `command-a-reasoning-08-2025` | 256K | Extended thinking |
| Vision | `command-a-vision-07-2025` | 128K | Images + text |
| Embeddings | `embed-v4.0` | 128K | Multimodal |
| Rerank | `rerank-v4.0-pro` | 32K | Best quality |

## Critical patterns

### 1. Client setup (ALWAYS use V2)

```python
import cohere

# Correct
co = cohere.ClientV2()  # Reads CO_API_KEY from env

# Wrong - deprecated
co = cohere.Client()
```

### 2. Embedding input types (CRITICAL)

Cohere uses asymmetric embeddings. Using wrong `input_type` = poor search results.

```python
# For STORING documents in vector DB
doc_embeddings = co.embed(
    model="embed-v4.0",
    texts=documents,
    input_type="search_document"  # MUST use this for docs
)

# For USER QUERIES searching against docs
query_embedding = co.embed(
    model="embed-v4.0",
    texts=[user_query],
    input_type="search_query"  # MUST use this for queries
)
```

### 3. Tool use response format

```python
# After executing a tool, format result as document
messages.append({
    "role": "tool",
    "tool_call_id": tool_call.id,  # Required in V2
    "content": [{"type": "document", "document": {"data": json.dumps(result)}}]
})
```

### 4. Two-stage retrieval (recommended)

```python
# Stage 1: Fast retrieval with embeddings (cast wide net)
candidates = vectorstore.similarity_search(query, k=30)

# Stage 2: Precise reranking (narrow down)
reranked = co.rerank(
    model="rerank-v4.0-pro",
    query=query,
    documents=[doc.page_content for doc in candidates],
    top_n=10
)
final_docs = [candidates[r.index] for r in reranked.results]
```

## Batch limits

| API | Limit |
|-----|-------|
| Embeddings | 96 items per request (hard limit) |
| Rerank | 1,000 recommended (10K max) |

```python
# Batch embeddings correctly
def embed_batched(texts, batch_size=96):
    all_embeddings = []
    for i in range(0, len(texts), batch_size):
        batch = texts[i:i + batch_size]
        response = co.embed(model="embed-v4.0", texts=batch, input_type="search_document")
        all_embeddings.extend(response.embeddings.float_)
    return all_embeddings
```

## LangChain integration

```python
from langchain_cohere import ChatCohere, CohereEmbeddings, CohereRerank

# Chat
llm = ChatCohere(model="command-a-03-2025", temperature=0.3)

# Embeddings
embeddings = CohereEmbeddings(model="embed-english-v3.0")

# Rerank
reranker = CohereRerank(model="rerank-v3.5", top_n=5)
```

**Note:** Reasoning (`command-a-reasoning-*`) and Vision (`command-a-vision-*`) models are NOT supported in LangChain. Use native SDK.

## Priority rules

When working with Cohere APIs, prioritize:

1. **Always use ClientV2** - V1 is deprecated
2. **Match embedding input types** - Most common cause of poor search
3. **Use lower temperature (0.3) for tool calling** - More predictable
4. **Implement two-stage retrieval** - Embeddings + reranking
5. **Respect batch limits** - 96 for embeddings

## Environment setup

```bash
export CO_API_KEY="your-api-key"      # Native SDK
export COHERE_API_KEY="your-api-key"  # LangChain
```

## Reference files

For detailed documentation, see `skills/cohere/references/`:

- **native-sdk.md** - Complete SDK reference
- **embeddings.md** - Embedding models and patterns
- **rerank.md** - Reranking and retrieval
- **streaming.md** - Streaming event types
- **structured-outputs.md** - JSON mode, schemas
- **langchain.md** - LangChain integration
- **langgraph.md** - Building agents
- **cookbooks.md** - Official examples
