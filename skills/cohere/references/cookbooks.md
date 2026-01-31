# Cohere Cookbooks Reference

> Official Cohere cookbooks with working code examples. All cookbooks available at: https://docs.cohere.com/page/cookbooks

## Quick Links by Category

### Agents & Tool Use (Start Here)
| Cookbook | Description | Key Concepts |
|----------|-------------|--------------|
| [Basic Tool Use](https://docs.cohere.com/page/basic-tool-use) | Single tool calling | Tool schema, function execution |
| [Basic Multi-Step](https://docs.cohere.com/page/basic-multi-step) | Agentic loops | Multi-turn tool use, planning |
| [Data Analyst Agent](https://docs.cohere.com/page/data-analyst-agent) | LangGraph + Cohere | Complex agent architecture |
| [CSV Agent](https://docs.cohere.com/page/csv-agent) | CSV analysis with tools | Pandas integration |
| [SQL Agent](https://docs.cohere.com/page/sql-agent) | Database querying | SQL generation, execution |
| [Calendar Agent](https://docs.cohere.com/page/calendar-agent) | Schedule management | Time-based tools |
| [Agent API Calls](https://docs.cohere.com/page/agent-api-calls) | External API integration | HTTP tools |
| [Agent Short-Term Memory](https://docs.cohere.com/page/agent-short-term-memory) | Stateful agents | Memory patterns |

### RAG (Retrieval-Augmented Generation)
| Cookbook | Description | Key Concepts |
|----------|-------------|--------------|
| [Basic RAG](https://docs.cohere.com/page/basic-rag) | Foundational RAG pattern | Documents, citations |
| [RAG with Chat + Embed](https://docs.cohere.com/page/rag-with-chat-embed) | Full RAG pipeline | End-to-end implementation |
| [Agentic Multi-Stage RAG](https://docs.cohere.com/page/agentic-multi-stage-rag) | Advanced RAG | Query decomposition |
| [Agentic RAG Mixed Data](https://docs.cohere.com/page/agentic-rag-mixed-data) | Multiple data sources | Heterogeneous retrieval |
| [RAG + MongoDB](https://docs.cohere.com/page/rag-cohere-mongodb) | MongoDB integration | Vector search |
| [Grounded Summarization](https://docs.cohere.com/page/grounded-summarization) | Citation-based summaries | Document grounding |
| [RAG Evaluation Deep Dive](https://docs.cohere.com/page/rag-evaluation-deep-dive) | Evaluating RAG systems | Metrics, testing |

### Semantic Search & Embeddings
| Cookbook | Description | Key Concepts |
|----------|-------------|--------------|
| [Basic Semantic Search](https://docs.cohere.com/page/basic-semantic-search) | Search fundamentals | Embeddings, similarity |
| [Multilingual Search](https://docs.cohere.com/page/multilingual-search) | Cross-language search | embed-multilingual |
| [Wikipedia Search (Weaviate)](https://docs.cohere.com/page/wikipedia-search-with-weaviate) | Weaviate integration | Vector DB |
| [Wikipedia Semantic Search](https://docs.cohere.com/page/wikipedia-semantic-search) | Large-scale search | Scalability patterns |
| [Elasticsearch + Cohere](https://docs.cohere.com/page/elasticsearch-and-cohere) | Elasticsearch integration | Hybrid search |
| [Article Recommender](https://docs.cohere.com/page/article-recommender-with-text-embeddings) | Recommendation system | Similarity-based recs |

### Reranking
| Cookbook | Description | Key Concepts |
|----------|-------------|--------------|
| [Rerank Demo](https://docs.cohere.com/page/rerank-demo) | Reranking fundamentals | Two-stage retrieval |

### Document Processing
| Cookbook | Description | Key Concepts |
|----------|-------------|--------------|
| [PDF Extractor](https://docs.cohere.com/page/pdf-extractor) | PDF processing | Document parsing |
| [Document Parsing for Enterprises](https://docs.cohere.com/page/document-parsing-for-enterprises) | Enterprise documents | Complex formats |
| [Financial Forms Analysis](https://docs.cohere.com/page/analysis-of-financial-forms) | Structured data extraction | Form processing |
| [Chunking Strategies](https://docs.cohere.com/page/chunking-strategies) | Text segmentation | Optimal chunk sizes |

### Classification & Analysis
| Cookbook | Description | Key Concepts |
|----------|-------------|--------------|
| [Text Classification (Embeddings)](https://docs.cohere.com/page/text-classification-using-embeddings) | Classification pipelines | Zero-shot, few-shot |
| [Topic Modeling AI Papers](https://docs.cohere.com/page/topic-modeling-ai-papers) | Topic extraction | Clustering |
| [Analyzing Hacker News](https://docs.cohere.com/page/analyzing-hacker-news) | Sentiment analysis | Real-world data |

### Production & Deployment
| Cookbook | Description | Key Concepts |
|----------|-------------|--------------|
| [Embed Jobs](https://docs.cohere.com/page/embed-jobs) | Batch embedding | Large datasets |
| [Embed Jobs + Pinecone](https://docs.cohere.com/page/embed-jobs-serverless-pinecone) | Serverless embedding | Pinecone integration |
| [Deploy on AWS Marketplace](https://docs.cohere.com/page/deploy-finetuned-model-aws-marketplace) | AWS deployment | Fine-tuned models |
| [Finetune on SageMaker](https://docs.cohere.com/page/finetune-on-sagemaker) | AWS SageMaker | Training |

### Specialized Use Cases
| Cookbook | Description | Key Concepts |
|----------|-------------|--------------|
| [Command A Translate](https://docs.cohere.com/page/command-a-translate-cookbook) | Translation | Multilingual |
| [Aya Vision Intro](https://docs.cohere.com/page/aya_vision_intro) | Vision capabilities | Multimodal |
| [Q&A Bot](https://docs.cohere.com/page/creating-a-qa-bot) | Chatbot creation | Conversational AI |
| [Generative Content](https://docs.cohere.com/page/fueling-generative-content) | Content generation | Creative writing |
| [Long-Form Strategies](https://docs.cohere.com/page/long-form-general-strategies) | Long documents | Context handling |

### Migration & Evaluation
| Cookbook | Description | Key Concepts |
|----------|-------------|--------------|
| [Migrating Prompts](https://docs.cohere.com/page/migrating-prompts) | Prompt migration | From other providers |
| [Migrate CSV Agent](https://docs.cohere.com/page/migrate-csv-agent) | Agent migration | LangChain updates |
| [Retrieval Eval (Pydantic AI)](https://docs.cohere.com/page/retrieval-eval-pydantic-ai) | Evaluation framework | Testing |
| [Summarization Evals](https://docs.cohere.com/page/summarization-evals) | Summary quality | Metrics |

## Recommended Learning Path

### Beginner
1. **Basic Tool Use** - Understand single tool calling
2. **Basic RAG** - Learn document-grounded generation
3. **Basic Semantic Search** - Master embeddings

### Intermediate
4. **Basic Multi-Step** - Build agentic loops
5. **CSV Agent** - Practical data analysis agent
6. **Rerank Demo** - Two-stage retrieval

### Advanced
7. **Data Analyst Agent** - Complex LangGraph architecture
8. **Agentic Multi-Stage RAG** - Production RAG patterns
9. **RAG Evaluation Deep Dive** - Quality assurance

## GitHub Repository

All cookbook source code: https://github.com/cohere-ai/cohere-developer-experience/tree/main/fern/pages/cookbooks

```bash
# Clone for local experimentation
git clone https://github.com/cohere-ai/cohere-developer-experience.git
cd cohere-developer-experience/fern/pages/cookbooks
```

## Quick Start Patterns

### Tool Use Agent (from Basic Multi-Step)
```python
import cohere
import json

co = cohere.ClientV2()

tools = [...]  # Your tools
functions_map = {...}  # Tool implementations

def run_agent(query: str, max_steps: int = 10):
    messages = [{"role": "user", "content": query}]

    for _ in range(max_steps):
        response = co.chat(model="command-a-03-2025", messages=messages, tools=tools)

        if not response.message.tool_calls:
            return response.message.content[0].text

        messages.append({
            "role": "assistant",
            "tool_plan": response.message.tool_plan,
            "tool_calls": response.message.tool_calls
        })

        for tc in response.message.tool_calls:
            result = functions_map[tc.function.name](**json.loads(tc.function.arguments))
            messages.append({
                "role": "tool",
                "tool_call_id": tc.id,
                "content": [{"type": "document", "document": {"data": json.dumps(result)}}]
            })

    return "Max steps reached"
```

### RAG Pattern (from Basic RAG)
```python
documents = [
    {"id": "doc1", "data": {"text": "Your document content..."}},
    {"id": "doc2", "data": {"text": "More content..."}}
]

response = co.chat(
    model="command-a-03-2025",
    messages=[{"role": "user", "content": "Your question"}],
    documents=documents
)

# Citations automatically included
for citation in response.message.citations or []:
    print(f"'{citation.text}' from {citation.sources}")
```

### Two-Stage Retrieval (from Rerank Demo)
```python
# Stage 1: Fast retrieval
candidates = vectorstore.similarity_search(query, k=100)

# Stage 2: Precise reranking
reranked = co.rerank(
    model="rerank-v4.0-pro",
    query=query,
    documents=[doc.page_content for doc in candidates],
    top_n=10
)

final_docs = [candidates[r.index] for r in reranked.results]
```
