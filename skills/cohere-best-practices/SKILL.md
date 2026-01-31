---
name: cohere-best-practices
description: Production best practices for Cohere AI APIs. Covers model selection, API configuration, error handling, cost optimization, and architectural patterns for chat, RAG, and agentic applications.
---

# Cohere Best Practices Reference

## Official Resources

- **Docs & Cookbooks**: https://github.com/cohere-ai/cohere-developer-experience
- **API Reference**: https://docs.cohere.com/reference/about

## Model Selection Guide

| Use Case | Model | Notes |
|----------|-------|-------|
| General chat/reasoning | `command-a-03-2025` | Latest Command A model |
| RAG with citations | `command-r-plus-08-2024` | Excellent grounded generation |
| Cost-sensitive tasks | `command-r-08-2024` | Good balance of quality/cost |
| Embeddings (English) | `embed-english-v3.0` | Best for English-only |
| Embeddings (Multilingual) | `embed-multilingual-v3.0` | 100+ languages |
| Reranking | `rerank-v3.5` | Good balance |
| Reranking (Quality) | `rerank-v4.0-pro` | Best quality, slower |
| Reranking (Speed) | `rerank-v4.0-fast` | Optimized for latency |

## API Configuration Best Practices

### Use Client V2
```python
import cohere

# Correct: Use ClientV2 for all new projects
co = cohere.ClientV2()

# Deprecated: Don't use the old client
# co = cohere.Client()  # Avoid
```

### Temperature Settings
```python
# For agents/tool calling - lower temperature for reliability
co.chat(model="command-a-03-2025", temperature=0.3, ...)

# For creative tasks - higher temperature
co.chat(model="command-a-03-2025", temperature=0.7, ...)

# For deterministic outputs - zero temperature
co.chat(model="command-a-03-2025", temperature=0, ...)
```

## Embedding Best Practices

### Always Specify input_type
```python
# For documents being indexed
doc_embeddings = co.embed(
    texts=documents,
    model="embed-english-v3.0",
    input_type="search_document",  # Critical!
    embedding_types=["float"]
)

# For search queries
query_embedding = co.embed(
    texts=[query],
    model="embed-english-v3.0",
    input_type="search_query",  # Must match at query time
    embedding_types=["float"]
)
```

> **Critical**: Mismatched `input_type` between indexing and querying will degrade search quality significantly.

## RAG Best Practices

### Two-Stage Retrieval Pattern
```python
# Stage 1: Broad retrieval with embeddings
candidates = vectorstore.similarity_search(query, k=30)

# Stage 2: Precise reranking
reranked = co.rerank(
    model="rerank-v3.5",
    query=query,
    documents=[doc.page_content for doc in candidates],
    top_n=5
)

# Use reranked results for generation
final_docs = [candidates[r.index] for r in reranked.results]
```

### Grounded Generation with Citations
```python
response = co.chat(
    model="command-r-plus-08-2024",
    messages=[{"role": "user", "content": question}],
    documents=[
        {"id": f"doc_{i}", "data": {"text": doc}}
        for i, doc in enumerate(final_docs)
    ]
)

# Access citations
for citation in response.message.citations:
    print(f"'{citation.text}' from {citation.sources}")
```

## Error Handling

```python
from cohere.core import ApiError

def safe_chat(messages, max_retries=3):
    for attempt in range(max_retries):
        try:
            return co.chat(
                model="command-a-03-2025",
                messages=messages
            )
        except ApiError as e:
            if e.status_code == 429:  # Rate limit
                time.sleep(2 ** attempt)
                continue
            elif e.status_code >= 500:  # Server error
                time.sleep(1)
                continue
            else:
                raise
    raise Exception("Max retries exceeded")
```

## Cost Optimization

1. **Use appropriate models**: Don't use Command A for simple tasks
2. **Batch embeddings**: Embed multiple texts in one call (up to 96 texts)
3. **Cache embeddings**: Store computed embeddings in a vector database
4. **Use reranking wisely**: Only rerank when quality matters
5. **Stream for UX**: Streaming doesn't cost more but improves perceived latency

## Production Checklist

- [ ] Use `ClientV2` for all API calls
- [ ] Set appropriate `temperature` for your use case
- [ ] Always specify `input_type` for embeddings
- [ ] Implement retry logic with exponential backoff
- [ ] Use two-stage retrieval for RAG
- [ ] Cache embeddings to reduce API calls
- [ ] Monitor token usage and costs
- [ ] Handle rate limits gracefully
