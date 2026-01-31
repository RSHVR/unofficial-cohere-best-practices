---
name: cohere-rerank
description: Cohere reranking reference for two-stage retrieval, semantic search improvement, and RAG pipelines. Covers Rerank v4 models, structured data reranking, and LangChain integration.
---

# Cohere Rerank Reference

## Official Resources

- **Docs & Cookbooks**: https://github.com/cohere-ai/cohere-developer-experience
- **API Reference**: https://docs.cohere.com/reference/about

## Models Overview

| Model | Context | Languages | Notes |
|-------|---------|-----------|-------|
| `rerank-v4.0-pro` | 32K tokens | 100+ | Best quality, slower |
| `rerank-v4.0-fast` | 32K tokens | 100+ | Optimized for speed |
| `rerank-v3.5` | 4K tokens | 100+ | Good balance |

## Two-Stage Retrieval Pattern (Recommended)

The proven pattern for production search:
1. **Stage 1**: Fast retrieval (embeddings/BM25) for top 30 candidates
2. **Stage 2**: Precise reranking for final top 10 results

```python
import cohere
co = cohere.ClientV2()

# Stage 1: Cast a wide net with embeddings
candidates = vectorstore.similarity_search(query, k=30)

# Stage 2: Precise reranking narrows to best results
reranked = co.rerank(
    model="rerank-v4.0-fast",
    query=query,
    documents=[doc.page_content for doc in candidates],
    top_n=10
)

final_docs = [candidates[r.index] for r in reranked.results]
```

## Native SDK Reranking

### Basic Reranking
```python
query = "What is machine learning?"
documents = [
    "Machine learning is a subset of AI that enables systems to learn from data.",
    "The weather today is sunny with clear skies.",
    "Deep learning uses neural networks with many layers.",
]

response = co.rerank(
    model="rerank-v3.5",
    query=query,
    documents=documents,
    top_n=3
)

for result in response.results:
    print(f"Index: {result.index}, Score: {result.relevance_score:.4f}")
    print(f"Document: {documents[result.index]}\n")
```

### With Return Documents
```python
response = co.rerank(
    model="rerank-v3.5",
    query=query,
    documents=documents,
    top_n=3,
    return_documents=True
)

for result in response.results:
    print(f"Score: {result.relevance_score:.4f}")
    print(f"Text: {result.document.text}\n")
```

## Structured Data Reranking

### JSON/Dict Documents
```python
import yaml

documents = [
    {"title": "ML Guide", "author": "John", "content": "Machine learning basics..."},
    {"title": "Weather Report", "author": "Jane", "content": "Today's forecast..."},
]

yaml_docs = [yaml.dump(doc, sort_keys=False) for doc in documents]

response = co.rerank(
    model="rerank-v3.5",
    query="machine learning tutorial",
    documents=yaml_docs,
    top_n=2
)
```

### Specify Rank Fields
```python
response = co.rerank(
    model="rerank-v3.5",
    query="machine learning",
    documents=[
        {"title": "ML Guide", "author": "John", "text": "Introduction to ML..."},
        {"title": "Weather", "author": "Jane", "text": "Sunny skies..."}
    ],
    rank_fields=["title", "text"]  # Only consider these fields
)
```

## LangChain Integration

### Basic Usage
```python
from langchain_cohere import CohereRerank
from langchain_core.documents import Document

reranker = CohereRerank(model="rerank-v3.5", top_n=3)

documents = [
    Document(page_content="Machine learning is a subset of AI..."),
    Document(page_content="The weather is sunny today..."),
]

reranked = reranker.compress_documents(
    documents=documents,
    query="What is machine learning?"
)

for doc in reranked:
    print(f"Score: {doc.metadata['relevance_score']:.4f}")
```

### With Contextual Compression Retriever
```python
from langchain_cohere import CohereEmbeddings, CohereRerank
from langchain_community.vectorstores import FAISS
from langchain.retrievers.contextual_compression import ContextualCompressionRetriever

embeddings = CohereEmbeddings(model="embed-english-v3.0")
vectorstore = FAISS.from_documents(docs, embeddings)
base_retriever = vectorstore.as_retriever(search_kwargs={"k": 20})

reranker = CohereRerank(model="rerank-v3.5", top_n=5)
retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=base_retriever
)

results = retriever.invoke("Your query here")
```

## Score Interpretation

Relevance scores are normalized to [0, 1]:
- **0.9+**: Highly relevant
- **0.5-0.9**: Moderately relevant
- **<0.5**: Low relevance

```python
threshold = 0.5
relevant = [r for r in response.results if r.relevance_score >= threshold]
```

## Best Practices

1. **Use two-stage retrieval**: Embeddings for recall, rerank for precision
2. **Batch large requests**: Max 10,000 documents per request
3. **Use YAML for structured data**: `yaml.dump(doc, sort_keys=False)` preserves field order
4. **Filter by score threshold**: Don't use low-relevance results
