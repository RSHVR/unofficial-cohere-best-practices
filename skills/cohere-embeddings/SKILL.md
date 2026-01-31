---
name: cohere-embeddings
description: Cohere embeddings reference for vector search, semantic similarity, and RAG. Covers Embed v4 (multimodal, Matryoshka dimensions), input types (CRITICAL for search quality), batch processing, and LangChain integration.
---

# Cohere Embeddings Reference

## Official Resources

- **Docs & Cookbooks**: https://github.com/cohere-ai/cohere-developer-experience
- **API Reference**: https://docs.cohere.com/reference/about

## Models Overview

| Model | Context | Dimensions | Features |
|-------|---------|------------|----------|
| `embed-v4.0` | 128K tokens | 256/512/1024/1536 | Multimodal (text+image), Matryoshka |
| `embed-english-v3.0` | 512 tokens | 1024 | English-only, fast |
| `embed-multilingual-v3.0` | 512 tokens | 1024 | 100+ languages |
| `embed-english-light-v3.0` | 512 tokens | 384 | Lightweight, fastest |

## Input Types (CRITICAL)

> **Using the wrong `input_type` will silently degrade search quality.** Cohere uses asymmetric embeddings where documents and queries are embedded differently.

| Input Type | Use Case |
|------------|----------|
| `search_document` | Documents stored in vector DB for retrieval |
| `search_query` | User queries searching against documents |
| `classification` | Text classification tasks |
| `clustering` | Clustering similar documents |
| `image` | Image inputs (Embed v4 only) |

### Example: Search Pipeline
```python
import cohere
co = cohere.ClientV2()

# INDEXING: Use search_document for docs you're storing
doc_response = co.embed(
    model="embed-english-v3.0",
    texts=documents,
    input_type="search_document"  # MUST use for storage
)

# QUERYING: Use search_query for user queries
query_response = co.embed(
    model="embed-english-v3.0",
    texts=[user_query],
    input_type="search_query"  # MUST use for retrieval
)
```

## Native SDK Embeddings

### Basic Text Embedding
```python
response = co.embed(
    model="embed-english-v3.0",
    texts=["Hello world", "Machine learning is cool"],
    input_type="search_document"
)

embeddings = response.embeddings.float_
print(f"Embedding shape: {len(embeddings)} x {len(embeddings[0])}")
```

### Embed v4 with Matryoshka Dimensions
```python
# High precision (default)
response = co.embed(
    model="embed-v4.0",
    texts=["text"],
    input_type="search_document",
    output_dimension=1536
)

# Balanced (3x faster search)
response = co.embed(
    model="embed-v4.0",
    texts=["text"],
    input_type="search_document",
    output_dimension=512
)

# Compact (6x faster search)
response = co.embed(
    model="embed-v4.0",
    texts=["text"],
    input_type="search_document",
    output_dimension=256
)
```

### Different Embedding Types
```python
response = co.embed(
    model="embed-english-v3.0",
    texts=["Hello"],
    input_type="search_document",
    embedding_types=["float", "int8", "uint8", "binary", "ubinary"]
)

float_emb = response.embeddings.float_
int8_emb = response.embeddings.int8
binary_emb = response.embeddings.binary
```

## Multimodal Embeddings (Embed v4)

### Image Embeddings
```python
import base64

with open("image.jpg", "rb") as f:
    image_base64 = base64.b64encode(f.read()).decode()

image_uri = f"data:image/jpeg;base64,{image_base64}"

response = co.embed(
    model="embed-v4.0",
    images=[image_uri],
    input_type="image"
)
```

### Mixed Content
```python
response = co.embed(
    model="embed-v4.0",
    inputs=[
        {"text": "A description of the product"},
        {"image": image_uri},
        {"text": "Another text chunk"}
    ],
    input_type="search_document"
)
```

## Batch Processing

### Hard Limit: 96 Items Per Request
```python
def embed_in_batches(texts: list, batch_size: int = 96):
    """Embed texts in batches of 96 (Cohere API limit)."""
    all_embeddings = []

    for i in range(0, len(texts), batch_size):
        batch = texts[i:i + batch_size]
        response = co.embed(
            model="embed-english-v3.0",
            texts=batch,
            input_type="search_document"
        )
        all_embeddings.extend(response.embeddings.float_)

    return all_embeddings
```

### Embed Jobs API (Large Datasets)
```python
job = co.embed_jobs.create(
    model="embed-english-v3.0",
    dataset_id="your-dataset-id",
    input_type="search_document"
)

status = co.embed_jobs.get(job.job_id)
print(status.status)  # "processing", "complete", "failed"
```

## LangChain Integration

### Basic Usage
```python
from langchain_cohere import CohereEmbeddings

embeddings = CohereEmbeddings(model="embed-english-v3.0")

vector = embeddings.embed_query("What is machine learning?")
vectors = embeddings.embed_documents(["Document 1", "Document 2"])
```

### With Vector Store
```python
from langchain_cohere import CohereEmbeddings
from langchain_community.vectorstores import FAISS
from langchain_text_splitters import RecursiveCharacterTextSplitter

embeddings = CohereEmbeddings(model="embed-english-v3.0")

text_splitter = RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=50)
docs = text_splitter.split_documents(your_documents)

vectorstore = FAISS.from_documents(docs, embeddings)
results = vectorstore.similarity_search("your query", k=5)
```

## Best Practices

1. **Match input types**: Always use `search_document` for stored docs and `search_query` for queries
2. **Batch efficiently**: Hard limit of 96 texts per request
3. **Choose dimensions wisely**: Lower dimensions = faster search but slightly less precision
4. **Chunk long texts**: Consider chunking at ~6000 chars (texts auto-truncate at 8K)

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=6000,
    chunk_overlap=200
)
chunks = splitter.split_text(long_document)
```
