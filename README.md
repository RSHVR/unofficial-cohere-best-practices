# Unofficial Cohere Best Practices

An (unofficial) collection of Agent Skills for building with Cohere's AI APIs. Covers chat, embeddings, reranking, streaming, structured outputs, and agent workflows.

Works with **Python**, **TypeScript**, **Java**, and **Go** SDKs, plus **LangChain** and **LangGraph** integrations.

## Installation

```bash
npx skills add RSHVR/unofficial-cohere-best-practices
```

Or install a specific skill:

```bash
npx skills add RSHVR/unofficial-cohere-best-practices --skill cohere-python-sdk
```

Manual installation:

```bash
cp -r skills/* ~/.claude/skills/
```

## Available Skills

| Skill | Description |
|-------|-------------|
| `cohere-python-sdk` | Python SDK patterns for chat, embeddings, reranking, and tool use |
| `cohere-typescript-sdk` | TypeScript SDK with Vercel AI SDK integration |
| `cohere-java-sdk` | Java SDK with builder patterns and Maven/Gradle setup |
| `cohere-go-sdk` | Go SDK with idiomatic patterns |
| `cohere-embeddings` | Asymmetric embeddings, input types, and vector search |
| `cohere-rerank` | Two-stage retrieval and semantic reranking |
| `cohere-streaming` | Real-time streaming with all event types |
| `cohere-structured-outputs` | JSON mode, schemas, and strict tool parameters |
| `cohere-langchain` | ChatCohere, CohereEmbeddings, CohereRerank, RAG chains |
| `cohere-langgraph` | ReAct agents, memory, human-in-the-loop patterns |
| `cohere-cookbooks` | Links to official Cohere cookbooks and tutorials |
| `cohere-best-practices` | Production patterns and configuration guidelines |

## Resources

- [Cohere API Reference](https://docs.cohere.com/reference/about)
- [Cohere Developer Experience](https://github.com/cohere-ai/cohere-developer-experience)
- [Cohere Dashboard](https://dashboard.cohere.com)

## License

MIT License - see [LICENSE](LICENSE)
