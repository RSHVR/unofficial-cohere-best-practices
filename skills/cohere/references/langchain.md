# Cohere LangChain Integration Reference

> **Model Compatibility Warning**: Command A Reasoning (`command-a-reasoning-*`) and Command A Vision (`command-a-vision-*`) are **not supported** in LangChain. Use the native Cohere SDK for these models. Standard chat models (`command-a-03-2025`, `command-r-plus-*`, `command-r-*`) work normally.

## Table of Contents
1. [Installation and Setup](#installation-and-setup)
2. [ChatCohere](#chatcohere)
3. [CohereEmbeddings](#cohereembeddings)
4. [CohereRerank](#coherererank)
5. [CohereRagRetriever](#cohereragretriever)
6. [Tool Calling](#tool-calling)
7. [Structured Output](#structured-output)
8. [Private Deployment](#private-deployment)

## Installation and Setup

### Installation
```bash
pip install langchain-cohere langchain langchain-core
```

### Environment Setup
```python
import os
os.environ["COHERE_API_KEY"] = "your-api-key"

# Optional: LangSmith tracing
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "your-langsmith-key"
```

### Import Map (v0.5+)
```python
# All Cohere integrations from langchain-cohere
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

### With System Message
```python
response = llm.invoke([
    SystemMessage(content="You are a helpful coding assistant."),
    HumanMessage(content="Write a Python hello world")
])
```

### Streaming
```python
for chunk in llm.stream([HumanMessage(content="Write a poem")]):
    print(chunk.content, end="", flush=True)
```

### Async
```python
response = await llm.ainvoke([HumanMessage(content="Hello")])

async for chunk in llm.astream([HumanMessage(content="Tell a story")]):
    print(chunk.content, end="")
```

### Parameters
```python
llm = ChatCohere(
    model="command-a-03-2025",
    temperature=0.7,          # Use 0.3 for agents/tool calling
    max_tokens=4096,
    cohere_api_key="your-key",  # Or use env var
    timeout=60,
    max_retries=3
)
```

### For Agents (Recommended Settings)
```python
# Lower temperature = more predictable tool calls
llm = ChatCohere(
    model="command-a-03-2025",
    temperature=0.3,          # Critical for reliable tool calling
    max_tokens=4096
)
```

### With Prompt Templates
```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

llm = ChatCohere(model="command-a-03-2025")

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a {role}."),
    ("human", "{input}")
])

chain = prompt | llm | StrOutputParser()

result = chain.invoke({
    "role": "helpful assistant",
    "input": "What is Python?"
})
```

## CohereEmbeddings

### Basic Usage
```python
from langchain_cohere import CohereEmbeddings

embeddings = CohereEmbeddings(model="embed-english-v3.0")

# Single query
query_vector = embeddings.embed_query("What is AI?")

# Multiple documents
doc_vectors = embeddings.embed_documents([
    "First document",
    "Second document"
])
```

### With Vector Store
```python
from langchain_cohere import CohereEmbeddings
from langchain_community.vectorstores import FAISS, Chroma, Pinecone

embeddings = CohereEmbeddings(model="embed-english-v3.0")

# FAISS
vectorstore = FAISS.from_texts(texts, embeddings)

# Chroma
vectorstore = Chroma.from_texts(texts, embeddings)

# Search
results = vectorstore.similarity_search("query", k=5)
```

## CohereRerank

### Basic Usage
```python
from langchain_cohere import CohereRerank
from langchain_core.documents import Document

reranker = CohereRerank(model="rerank-v3.5", top_n=3)

docs = [
    Document(page_content="ML is a subset of AI..."),
    Document(page_content="Weather is sunny..."),
    Document(page_content="Neural networks are...")
]

reranked = reranker.compress_documents(docs, query="What is ML?")
```

### With Retriever
```python
from langchain.retrievers.contextual_compression import ContextualCompressionRetriever
from langchain_cohere import CohereRerank

base_retriever = vectorstore.as_retriever(search_kwargs={"k": 20})
reranker = CohereRerank(model="rerank-v3.5", top_n=5)

retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=base_retriever
)

results = retriever.invoke("Your query")
```

## CohereRagRetriever

Built-in RAG with web search or custom documents:

### With Web Search
```python
from langchain_cohere import ChatCohere, CohereRagRetriever

llm = ChatCohere()
rag = CohereRagRetriever(llm=llm)

# Uses Cohere's web search connector
docs = rag.get_relevant_documents("What is the latest news about AI?")
```

### With Custom Documents
```python
from langchain_cohere import ChatCohere, CohereRagRetriever
from langchain_core.documents import Document

llm = ChatCohere()

documents = [
    Document(page_content="Our Q3 revenue was $10M"),
    Document(page_content="Customer count: 5000")
]

rag = CohereRagRetriever(
    llm=llm,
    connectors=[]  # Disable web search
)

# Pass documents in the query
response = rag.invoke(
    "What was Q3 revenue?",
    documents=documents
)
```

## Tool Calling

### Define and Bind Tools
```python
from langchain_cohere import ChatCohere
from langchain_core.tools import tool

@tool
def get_weather(location: str) -> str:
    """Get weather for a location."""
    return f"Weather in {location}: 20°C, sunny"

@tool
def search_database(query: str) -> str:
    """Search the company database."""
    return f"Results for '{query}': ..."

llm = ChatCohere(model="command-a-03-2025")
llm_with_tools = llm.bind_tools([get_weather, search_database])

response = llm_with_tools.invoke("What's the weather in Toronto?")

# Check for tool calls
if response.tool_calls:
    for tc in response.tool_calls:
        print(f"Tool: {tc['name']}, Args: {tc['args']}")
```

### Execute Tool Calls
```python
from langchain_core.messages import HumanMessage, ToolMessage

tools = [get_weather, search_database]
tool_map = {t.name: t for t in tools}

messages = [HumanMessage(content="What's the weather in Tokyo?")]
response = llm_with_tools.invoke(messages)

if response.tool_calls:
    messages.append(response)
    
    for tc in response.tool_calls:
        result = tool_map[tc["name"]].invoke(tc["args"])
        messages.append(ToolMessage(
            content=result,
            tool_call_id=tc["id"]
        ))
    
    final_response = llm_with_tools.invoke(messages)
    print(final_response.content)
```

## Structured Output

### With Pydantic
```python
from langchain_cohere import ChatCohere
from pydantic import BaseModel, Field

class Person(BaseModel):
    name: str = Field(description="Person's name")
    age: int = Field(description="Person's age")

llm = ChatCohere(model="command-a-03-2025")
structured_llm = llm.with_structured_output(Person)

result = structured_llm.invoke("John is 30 years old")
print(result)  # Person(name='John', age=30)
```

### With JSON Schema
```python
schema = {
    "type": "object",
    "properties": {
        "name": {"type": "string"},
        "age": {"type": "integer"}
    },
    "required": ["name", "age"]
}

structured_llm = llm.with_structured_output(schema)
result = structured_llm.invoke("Extract: Sarah is 25")
print(result)  # {"name": "Sarah", "age": 25}
```

## Private Deployment

### Custom Client
```python
from langchain_cohere import ChatCohere
from cohere import ClientV2

# Create custom client for private deployment
custom_client = ClientV2(
    api_key="your-key",
    base_url="https://your-deployment.example.com"
)

llm = ChatCohere(client=custom_client)
```

### Azure / AWS / GCP
```python
# Azure
from cohere import ClientV2
client = ClientV2(
    api_key="your-azure-key",
    base_url="https://your-resource.openai.azure.com/..."
)

# AWS Bedrock (use boto3)
# GCP (use google-cloud SDK)
```

## SQL Agent

Cohere provides a specialized SQL agent for database querying:

```python
from langchain_cohere import ChatCohere, create_sql_agent
from langchain_community.utilities import SQLDatabase

# Connect to database
db = SQLDatabase.from_uri("sqlite:///Chinook.db")

# Create agent
llm = ChatCohere(model="command-a-03-2025", temperature=0)
agent_executor = create_sql_agent(
    llm=llm,
    db=db,
    verbose=True,
    top_k=10,              # Max rows to return
    max_iterations=15      # Max reasoning steps
)

# Query in natural language
result = agent_executor.invoke({
    "input": "What are the top 5 albums by sales?"
})
print(result["output"])
```

### With Few-Shot Examples
```python
from langchain_core.example_selectors import SemanticSimilarityExampleSelector
from langchain_cohere import CohereEmbeddings

# Define examples
examples = [
    {"input": "How many artists?", "query": "SELECT COUNT(*) FROM Artist"},
    {"input": "Top selling album?", "query": "SELECT ... ORDER BY sales DESC LIMIT 1"}
]

# Create selector
selector = SemanticSimilarityExampleSelector.from_examples(
    examples,
    CohereEmbeddings(model="embed-english-v3.0"),
    k=2
)

agent_executor = create_sql_agent(
    llm=llm,
    db=db,
    # Pass custom prompt with examples
)
```

## Summarize Chain

For document summarization:

```python
from langchain_cohere import ChatCohere
from langchain_cohere.chains.summarize import load_summarize_chain
from langchain_core.documents import Document

llm = ChatCohere(model="command-a-03-2025")

# Load summarization chain
chain = load_summarize_chain(llm)

# Summarize documents
docs = [Document(page_content="Long document text here...")]
summary = chain.invoke({"input_documents": docs})
```

## CohereCitation

Handle fine-grained citations from Cohere responses:

```python
from langchain_cohere.common import CohereCitation

# Citations are returned with start/end positions and source document IDs
# Useful for building RAG systems with source attribution
citation = CohereCitation(
    start=0,
    end=50,
    text="The quoted text from the source",
    document_ids=["doc_1"]
)
```

## Full RAG Chain Example

```python
from langchain_cohere import ChatCohere, CohereEmbeddings, CohereRerank
from langchain_community.vectorstores import FAISS
from langchain.retrievers.contextual_compression import ContextualCompressionRetriever
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

# 1. Setup embeddings and vector store
embeddings = CohereEmbeddings(model="embed-english-v3.0")
vectorstore = FAISS.from_texts(your_texts, embeddings)

# 2. Setup retriever with reranking
base_retriever = vectorstore.as_retriever(search_kwargs={"k": 20})
reranker = CohereRerank(model="rerank-v3.5", top_n=5)
retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=base_retriever
)

# 3. Setup LLM
llm = ChatCohere(model="command-a-03-2025")

# 4. Build chain
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

# 5. Use
answer = chain.invoke("Your question here")
```
