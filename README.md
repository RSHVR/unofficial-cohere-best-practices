# Unofficial Cohere Best Practices

A Claude Code skill providing best practices for Cohere's AI APIs across **Python**, **TypeScript**, **Java**, and **Go**.

## What's Included

### Features
- **Chat/Text Generation** - Command A, Command R+, Command R models
- **Reasoning Model** - Command A Reasoning with `budget_tokens` control
- **Vision Model** - Command A Vision for images + text
- **Embeddings** - Embed v4 (multimodal, Matryoshka), v3 models
- **Reranking** - Two-stage retrieval patterns with Rerank v4/v3.5
- **Streaming** - All event types and patterns
- **Structured Outputs** - JSON mode, JSON Schema, strict_tools
- **RAG** - Document grounding with citations
- **Tool Use** - Function calling patterns
- **Agents** - LangGraph with create_cohere_react_agent

### Languages & Frameworks
- **Python** - Native SDK (`cohere.ClientV2`) + LangChain/LangGraph
- **TypeScript** - Native SDK (`cohere-ai`)
- **Java** - Native SDK (`cohere-java`)
- **Go** - Native SDK (`cohere-go`)

## Installation

**One-liner (recommended):**
```bash
npx skills add RSHVR/unofficial-cohere-best-practices
```

**Or via Claude Code directly:**
```bash
/install RSHVR/unofficial-cohere-best-practices
```

Works with Claude Code, Cursor, OpenCode, Codex, and [36+ other agents](https://github.com/vercel-labs/skills).

## Prerequisites

Get your Cohere API key at https://dashboard.cohere.com

Add to `~/.claude/settings.json`:
```json
{
  "env": {
    "CO_API_KEY": "your-api-key",
    "COHERE_API_KEY": "your-api-key"
  }
}
```

## Usage

Once installed, the skill activates when you mention Cohere, Command models, embeddings, reranking, or related concepts. It provides:

- Current model recommendations (2025)
- Code patterns for common tasks
- Critical gotchas (like embedding input types)
- Troubleshooting guides

## Reference Files

### By Language
| File | Language | Topics |
|------|----------|--------|
| `python-sdk.md` | Python | Chat, streaming, tool use, structured outputs, RAG |
| `typescript-sdk.md` | TypeScript | Chat, streaming, embeddings, rerank, tool use |
| `java-sdk.md` | Java | Chat, streaming, embeddings, rerank, tool use |
| `go-sdk.md` | Go | Chat, streaming, embeddings, rerank, tool use |

### By Feature
| File | Topics |
|------|--------|
| `embeddings.md` | Embed v4/v3, input types, batch processing |
| `rerank.md` | Reranking models, two-stage retrieval |
| `streaming.md` | Event types, async patterns |
| `structured-outputs.md` | JSON mode, schemas, strict_tools |
| `langchain.md` | ChatCohere, CohereEmbeddings, CohereRerank |
| `langgraph.md` | ReAct agents, memory, human-in-the-loop |
| `cookbooks.md` | Curated official cookbook links |

## Official Resources

- [Cohere Documentation](https://docs.cohere.com)
- [Cohere API Reference](https://docs.cohere.com/reference/about)
- [Cohere Cookbooks](https://docs.cohere.com/page/cookbooks)
- [Cohere Python SDK](https://github.com/cohere-ai/cohere-python)
- [LangChain-Cohere](https://python.langchain.com/docs/integrations/providers/cohere/)

## License

MIT
