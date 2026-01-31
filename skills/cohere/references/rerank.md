# Cohere Rerank Reference

## Table of Contents
1. [Models Overview](#models-overview)
2. [Native SDK Reranking](#native-sdk-reranking)
3. [LangChain CohereRerank](#langchain-coherererank)
4. [Structured Data Reranking](#structured-data-reranking)
5. [Best Practices](#best-practices)

## Models Overview

| Model | Context | Languages | Notes |
|-------|---------|-----------|-------|
| `rerank-v4.0-pro` | 32K tokens | 100+ | Best quality, slower |
| `rerank-v4.0-fast` | 32K tokens | 100+ | Optimized for speed |
| `rerank-v3.5` | 4K tokens | 100+ | Good balance |
| `rerank-english-v3.0` | 4K tokens | English | Legacy |
| `rerank-multilingual-v3.0` | 4K tokens | 100+ | Legacy |

## Native SDK Reranking

### Basic Reranking
```python
import cohere

co = cohere.ClientV2()

query = "What is machine learning?"
documents = [
    "Machine learning is a subset of AI that enables systems to learn from data.",
    "The weather today is sunny with clear skies.",
    "Deep learning uses neural networks with many layers.",
    "Python is a popular programming language."
]

response = co.rerank(
    model="rerank-v3.5",
    query=query,
    documents=documents,
    top_n=3  # Return top 3 results
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
    return_documents=True  # Include document text in response
)

for result in response.results:
    print(f"Score: {result.relevance_score:.4f}")
    print(f"Text: {result.document.text}\n")
```

### Control Token Limits
```python
response = co.rerank(
    model="rerank-v3.5",
    query=query,
    documents=documents,
    max_tokens_per_doc=1024  # Default is 4096
)
```

## LangChain CohereRerank

### Basic Usage
```python
from langchain_cohere import CohereRerank
from langchain_core.documents import Document

reranker = CohereRerank(model="rerank-v3.5", top_n=3)

documents = [
    Document(page_content="Machine learning is a subset of AI..."),
    Document(page_content="The weather is sunny today..."),
    Document(page_content="Deep learning uses neural networks...")
]

reranked = reranker.compress_documents(
    documents=documents,
    query="What is machine learning?"
)

for doc in reranked:
    print(f"Score: {doc.metadata['relevance_score']:.4f}")
    print(f"Content: {doc.page_content}\n")
```

### With Contextual Compression Retriever
```python
from langchain_cohere import CohereEmbeddings, CohereRerank
from langchain_community.vectorstores import FAISS
from langchain.retrievers.contextual_compression import ContextualCompressionRetriever

# Create base retriever
embeddings = CohereEmbeddings(model="embed-english-v3.0")
vectorstore = FAISS.from_documents(docs, embeddings)
base_retriever = vectorstore.as_retriever(search_kwargs={"k": 20})

# Add reranking
reranker = CohereRerank(model="rerank-v3.5", top_n=5)
compression_retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=base_retriever
)

# Use in retrieval
results = compression_retriever.invoke("Your query here")
```

### Full RAG Pipeline with Reranking
```python
from langchain_cohere import ChatCohere, CohereEmbeddings, CohereRerank
from langchain_community.vectorstores import FAISS
from langchain.retrievers.contextual_compression import ContextualCompressionRetriever
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

# Setup components
embeddings = CohereEmbeddings(model="embed-english-v3.0")
vectorstore = FAISS.from_documents(docs, embeddings)
base_retriever = vectorstore.as_retriever(search_kwargs={"k": 20})

reranker = CohereRerank(model="rerank-v3.5", top_n=5)
retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=base_retriever
)

llm = ChatCohere(model="command-a-03-2025")

# Build chain
prompt = ChatPromptTemplate.from_template("""
Answer based on the context:

Context: {context}

Question: {question}

Answer:
""")

def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

answer = chain.invoke("Your question here")
```

## Structured Data Reranking

### JSON/Dict Documents
Rerank v3.5+ supports structured data. Format as YAML strings for best results:

```python
import yaml

documents = [
    {"title": "ML Guide", "author": "John", "content": "Machine learning basics..."},
    {"title": "Weather Report", "author": "Jane", "content": "Today's forecast..."},
    {"title": "DL Tutorial", "author": "Bob", "content": "Neural networks..."}
]

# Convert to YAML strings (maintains key order)
yaml_docs = [yaml.dump(doc, sort_keys=False) for doc in documents]

response = co.rerank(
    model="rerank-v3.5",
    query="machine learning tutorial",
    documents=yaml_docs,
    top_n=2
)
```

### Specify Rank Fields
For JSON documents, specify which fields to consider:

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

### Tabular Data (CSV/DataFrame)
```python
import pandas as pd
import yaml

df = pd.DataFrame({
    "name": ["Alice", "Bob", "Charlie"],
    "role": ["ML Engineer", "Data Analyst", "Backend Dev"],
    "skills": ["Python, TensorFlow", "SQL, Tableau", "Java, Go"]
})

# Convert rows to YAML
yaml_docs = [yaml.dump(row.to_dict(), sort_keys=False) for _, row in df.iterrows()]

response = co.rerank(
    model="rerank-v3.5",
    query="machine learning engineer",
    documents=yaml_docs,
    top_n=2
)
```

## Best Practices

### 1. Score Interpretation
Relevance scores are normalized to [0, 1]. Scores are relative to the query:
- 0.9+ : Highly relevant
- 0.5-0.9 : Moderately relevant
- <0.5 : Low relevance

```python
# Filter by score threshold
threshold = 0.5
relevant = [r for r in response.results if r.relevance_score >= threshold]
```

### 2. Document Limits
- Max 10,000 documents per request
- Long documents auto-chunked at `max_tokens_per_doc`
- For >10K docs, batch your requests

```python
def rerank_batched(query, documents, batch_size=1000, top_n=10):
    all_results = []
    
    for i in range(0, len(documents), batch_size):
        batch = documents[i:i + batch_size]
        response = co.rerank(
            model="rerank-v3.5",
            query=query,
            documents=batch,
            top_n=min(top_n, len(batch))
        )
        # Adjust indices for global position
        for r in response.results:
            r.index += i
        all_results.extend(response.results)
    
    # Sort by score and take top_n
    all_results.sort(key=lambda x: x.relevance_score, reverse=True)
    return all_results[:top_n]
```

### 3. Two-Stage Retrieval Pattern (Recommended)
The proven pattern from production agents:
1. First stage: Fast retrieval (BM25 or vector search) for top 30 candidates
2. Second stage: Rerank for final top 10 results

```python
# Stage 1: Cast a wide net with embeddings
candidates = vectorstore.similarity_search(query, k=30)

# Stage 2: Precise reranking narrows to best results
reranked = co.rerank(
    model="rerank-v4.0-fast",  # Use 'fast' for speed, 'pro' for quality
    query=query,
    documents=[doc.page_content for doc in candidates],
    top_n=10
)

# Get the reranked documents
final_docs = [candidates[r.index] for r in reranked.results]
```

This pattern balances speed (embeddings are fast) with precision (reranker is accurate).

### 4. YAML for Structured Data
Always use YAML format with `sort_keys=False` to maintain field order:

```python
import yaml

doc = {"title": "Important", "body": "Content here"}
yaml_str = yaml.dump(doc, sort_keys=False)  # Preserves order
```
