# Cohere Embeddings Reference

## Table of Contents
1. [Models Overview](#models-overview)
2. [Native SDK Embeddings](#native-sdk-embeddings)
3. [LangChain CohereEmbeddings](#langchain-cohereembeddings)
4. [Input Types](#input-types)
5. [Embedding Dimensions](#embedding-dimensions)
6. [Multimodal Embeddings](#multimodal-embeddings)
7. [Batch Processing](#batch-processing)

## Models Overview

| Model | Context | Dimensions | Features |
|-------|---------|------------|----------|
| `embed-v4` | 128K tokens | 256/512/1024/1536 | Multimodal (text+image), Matryoshka |
| `embed-english-v3.0` | 512 tokens | 1024 | English-only, fast |
| `embed-multilingual-v3.0` | 512 tokens | 1024 | 100+ languages |
| `embed-english-light-v3.0` | 512 tokens | 384 | Lightweight, fastest |

## Native SDK Embeddings

### Basic Text Embedding
```python
import cohere

co = cohere.ClientV2()

response = co.embed(
    model="embed-english-v3.0",
    texts=["Hello world", "Machine learning is cool"],
    input_type="search_document"
)

embeddings = response.embeddings.float_
print(f"Embedding shape: {len(embeddings)} x {len(embeddings[0])}")
```

### With Embed v4 (Matryoshka Dimensions)
```python
response = co.embed(
    model="embed-v4",
    texts=["Your text here"],
    input_type="search_document",
    output_dimension=512  # Options: 256, 512, 1024, 1536
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

# Access different types
float_emb = response.embeddings.float_
int8_emb = response.embeddings.int8
binary_emb = response.embeddings.binary
```

## LangChain CohereEmbeddings

### Basic Usage
```python
from langchain_cohere import CohereEmbeddings

embeddings = CohereEmbeddings(model="embed-english-v3.0")

# Single text
vector = embeddings.embed_query("What is machine learning?")

# Multiple documents
vectors = embeddings.embed_documents([
    "Document 1 content",
    "Document 2 content"
])
```

### With Vector Store
```python
from langchain_cohere import CohereEmbeddings
from langchain_community.vectorstores import FAISS
from langchain_text_splitters import RecursiveCharacterTextSplitter

embeddings = CohereEmbeddings(model="embed-english-v3.0")

# Split documents
text_splitter = RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=50)
docs = text_splitter.split_documents(your_documents)

# Create vector store
vectorstore = FAISS.from_documents(docs, embeddings)

# Search
results = vectorstore.similarity_search("your query", k=5)
```

### Async Embeddings
```python
from langchain_cohere import CohereEmbeddings

embeddings = CohereEmbeddings(model="embed-english-v3.0")

# Async embedding
vector = await embeddings.aembed_query("What is AI?")
vectors = await embeddings.aembed_documents(["doc1", "doc2"])
```

## Input Types

> **CRITICAL**: Using the wrong `input_type` will give poor search results. Cohere trains **asymmetric embeddings** where documents and queries are embedded differently.

The `input_type` parameter optimizes embeddings for different use cases:

| Input Type | Use Case |
|------------|----------|
| `search_document` | Documents stored in vector DB for retrieval |
| `search_query` | User queries searching against documents |
| `classification` | Text classification tasks |
| `clustering` | Clustering similar documents |
| `image` | Image inputs (Embed v4 only) |

### Example: Search Pipeline (MUST match types)
```python
# INDEXING: Use search_document for docs you're storing
doc_response = co.embed(
    model="embed-english-v3.0",
    texts=documents,
    input_type="search_document"  # For storage
)

# QUERYING: Use search_query for user queries
query_response = co.embed(
    model="embed-english-v3.0",
    texts=[user_query],
    input_type="search_query"  # For retrieval
)
```

## Embedding Dimensions

### Embed v4 Matryoshka Embeddings
Embed v4 supports flexible output dimensions without quality loss:

```python
# High precision (default)
response = co.embed(
    model="embed-v4",
    texts=["text"],
    input_type="search_document",
    output_dimension=1536
)

# Balanced (3x faster search)
response = co.embed(
    model="embed-v4",
    texts=["text"],
    input_type="search_document",
    output_dimension=512
)

# Compact (6x faster search)
response = co.embed(
    model="embed-v4",
    texts=["text"],
    input_type="search_document",
    output_dimension=256
)
```

## Multimodal Embeddings

### Embed v4 Image Embeddings
```python
import base64

# Load image as base64
with open("image.jpg", "rb") as f:
    image_base64 = base64.b64encode(f.read()).decode()

image_uri = f"data:image/jpeg;base64,{image_base64}"

response = co.embed(
    model="embed-v4",
    images=[image_uri],
    input_type="image"
)
```

### Mixed Content (Embed v4)
```python
response = co.embed(
    model="embed-v4",
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
Cohere's embedding API has a **hard limit of 96 items per request**. Always batch:

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

### Embed Jobs API (For Very Large Datasets)
```python
# Create embed job for large datasets
job = co.embed_jobs.create(
    model="embed-english-v3.0",
    dataset_id="your-dataset-id",
    input_type="search_document"
)

# Check status
status = co.embed_jobs.get(job.job_id)
print(status.status)  # "processing", "complete", "failed"

# Get results when complete
results = co.embed_jobs.list()
```

## Best Practices

1. **Match input types**: Always use `search_document` for stored docs and `search_query` for queries. This is critical.
2. **Batch efficiently**: Hard limit of 96 texts per request for v3 models
3. **Choose dimensions wisely**: Lower dimensions = faster search but slightly less precision
4. **Truncate long texts**: Consider chunking at ~8000 chars. Long texts auto-truncate which may cut off important content.
5. **Use truncation strategically**: For long docs, chunk first for better semantic coverage

```python
# Recommended chunking for long documents
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,       # ~500 tokens for safety
    chunk_overlap=50,
    length_function=len,  # Character-based
)

# Or for longer chunks (still under 8K char limit)
splitter = RecursiveCharacterTextSplitter(
    chunk_size=6000,      # Safe margin under 8K
    chunk_overlap=200
)
chunks = splitter.split_text(long_document)
```
