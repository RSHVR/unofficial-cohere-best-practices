---
name: cohere-langchain
description: Cohere LangChain integration reference for ChatCohere, CohereEmbeddings, CohereRerank, and CohereRagRetriever. Use for building RAG pipelines, chains, and tool-calling workflows with LangChain.
---

# Cohere LangChain Integration Reference

## Official Resources

- **Docs & Cookbooks**: https://github.com/cohere-ai/cohere-developer-experience
- **API Reference**: https://docs.cohere.com/reference/about

> **Model Compatibility**: Command A Reasoning and Command A Vision are **not supported** in LangChain. Use the native Cohere SDK for these models.

## Installation

```bash
pip install langchain-cohere langchain langchain-core
```

## Import Map (v0.5+)
```python
from langchain_cohere import (
    ChatCohere,
    CohereEmbeddings,
    CohereRerank,
    CohereRagRetriever,
    create_cohere_react_agent
)
# NOT from langchain_community (deprecated)
```

## ChatCohere

### Basic Usage
```python
from langchain_cohere import ChatCohere
from langchain_core.messages import HumanMessage, SystemMessage

llm = ChatCohere(model="command-a-03-2025")

response = llm.invoke([
    HumanMessage(content="What is machine learning?")
])
print(response.content)
```

### Streaming
```python
for chunk in llm.stream([HumanMessage(content="Write a poem")]):
    print(chunk.content, end="", flush=True)
```

### For Agents (Recommended Settings)
```python
llm = ChatCohere(
    model="command-a-03-2025",
    temperature=0.3,  # Critical for reliable tool calling
    max_tokens=4096
)
```

### With Prompt Templates
```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a {role}."),
    ("human", "{input}")
])

chain = prompt | llm | StrOutputParser()
result = chain.invoke({"role": "helpful assistant", "input": "What is Python?"})
```

## CohereEmbeddings

```python
from langchain_cohere import CohereEmbeddings

embeddings = CohereEmbeddings(model="embed-english-v3.0")

query_vector = embeddings.embed_query("What is AI?")
doc_vectors = embeddings.embed_documents(["First document", "Second document"])
```

### With Vector Store
```python
from langchain_community.vectorstores import FAISS

vectorstore = FAISS.from_texts(texts, embeddings)
results = vectorstore.similarity_search("query", k=5)
```

## CohereRerank

```python
from langchain_cohere import CohereRerank
from langchain_core.documents import Document

reranker = CohereRerank(model="rerank-v3.5", top_n=3)

docs = [
    Document(page_content="ML is a subset of AI..."),
    Document(page_content="Weather is sunny..."),
]

reranked = reranker.compress_documents(docs, query="What is ML?")
```

### With Contextual Compression Retriever
```python
from langchain.retrievers.contextual_compression import ContextualCompressionRetriever

base_retriever = vectorstore.as_retriever(search_kwargs={"k": 20})
reranker = CohereRerank(model="rerank-v3.5", top_n=5)

retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=base_retriever
)

results = retriever.invoke("Your query")
```

## Tool Calling

```python
from langchain_core.tools import tool

@tool
def get_weather(location: str) -> str:
    """Get weather for a location."""
    return f"Weather in {location}: 20°C, sunny"

llm = ChatCohere(model="command-a-03-2025")
llm_with_tools = llm.bind_tools([get_weather])

response = llm_with_tools.invoke("What's the weather in Toronto?")

if response.tool_calls:
    for tc in response.tool_calls:
        print(f"Tool: {tc['name']}, Args: {tc['args']}")
```

## Structured Output

```python
from pydantic import BaseModel, Field

class Person(BaseModel):
    name: str = Field(description="Person's name")
    age: int = Field(description="Person's age")

llm = ChatCohere(model="command-a-03-2025")
structured_llm = llm.with_structured_output(Person)

result = structured_llm.invoke("John is 30 years old")
print(result)  # Person(name='John', age=30)
```

## Full RAG Chain Example

```python
from langchain_cohere import ChatCohere, CohereEmbeddings, CohereRerank
from langchain_community.vectorstores import FAISS
from langchain.retrievers.contextual_compression import ContextualCompressionRetriever
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

# Setup
embeddings = CohereEmbeddings(model="embed-english-v3.0")
vectorstore = FAISS.from_texts(your_texts, embeddings)
base_retriever = vectorstore.as_retriever(search_kwargs={"k": 20})

reranker = CohereRerank(model="rerank-v3.5", top_n=5)
retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=base_retriever
)

llm = ChatCohere(model="command-a-03-2025")

prompt = ChatPromptTemplate.from_template("""
Answer based on context:
Context: {context}
Question: {question}
""")

def format_docs(docs):
    return "\n\n".join(d.page_content for d in docs)

chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

answer = chain.invoke("Your question here")
```
