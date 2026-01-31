# Agent Skills

A collection of agent skills for AI coding assistants. Each skill provides focused, actionable reference documentation for specific tools, frameworks, and APIs.

## Structure

```
agent-skills/
├── skills/
│   ├── cohere-python-sdk/
│   │   └── SKILL.md
│   ├── cohere-typescript-sdk/
│   │   └── SKILL.md
│   ├── cohere-java-sdk/
│   │   └── SKILL.md
│   ├── cohere-go-sdk/
│   │   └── SKILL.md
│   ├── cohere-embeddings/
│   │   └── SKILL.md
│   ├── cohere-rerank/
│   │   └── SKILL.md
│   ├── cohere-streaming/
│   │   └── SKILL.md
│   ├── cohere-structured-outputs/
│   │   └── SKILL.md
│   ├── cohere-langchain/
│   │   └── SKILL.md
│   ├── cohere-langgraph/
│   │   └── SKILL.md
│   ├── cohere-cookbooks/
│   │   └── SKILL.md
│   └── cohere-best-practices/
│       └── SKILL.md
├── README.md
├── LICENSE
└── .gitignore
```

## Available Skills

### Cohere SDK Skills
| Skill | Description |
|-------|-------------|
| `cohere-python-sdk` | Python SDK patterns for chat, embeddings, reranking, and tool use |
| `cohere-typescript-sdk` | TypeScript SDK with Vercel AI SDK integration |
| `cohere-java-sdk` | Java SDK with builder patterns and Maven/Gradle setup |
| `cohere-go-sdk` | Go SDK with idiomatic patterns |

### Cohere Feature Skills
| Skill | Description |
|-------|-------------|
| `cohere-embeddings` | Asymmetric embeddings, input types, and vector search |
| `cohere-rerank` | Two-stage retrieval and semantic reranking |
| `cohere-streaming` | Real-time streaming with all event types |
| `cohere-structured-outputs` | JSON mode, schemas, and strict tool parameters |

### Cohere Integration Skills
| Skill | Description |
|-------|-------------|
| `cohere-langchain` | ChatCohere, CohereEmbeddings, CohereRerank, RAG chains |
| `cohere-langgraph` | ReAct agents, memory, human-in-the-loop patterns |

### Reference Skills
| Skill | Description |
|-------|-------------|
| `cohere-cookbooks` | Links to official Cohere cookbooks and tutorials |
| `cohere-best-practices` | Production patterns and configuration guidelines |

## Usage

### With npx skills (Recommended)

```bash
npx skills add RSHVR/unofficial-cohere-best-practices
```

Install a specific skill:

```bash
npx skills add RSHVR/unofficial-cohere-best-practices --skill cohere-python-sdk
```

### Manual Installation

Add a skill to your project's `.claude/skills/` directory:

```bash
cp -r skills/cohere-python-sdk ~/.claude/skills/
```

## License

MIT License - see [LICENSE](LICENSE)
