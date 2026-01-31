# AGENTS.md

This file provides guidance to AI coding agents (Claude Code, Cursor, Copilot, etc.) when working with code in this repository.

## Project overview

Agent Skills is a collection of skills for working with Cohere's AI APIs. Skills follow the [Agent Skills specification](https://agentskills.io/specification).

## Directory structure

```
agent-skills/
├── README.md                    # Project documentation
├── CLAUDE.md                    # Claude Code guidance
├── AGENTS.md                    # This file
├── LICENSE                      # MIT License
└── skills/
    ├── cohere-python-sdk/       # Python SDK patterns
    │   └── SKILL.md
    ├── cohere-typescript-sdk/   # TypeScript SDK patterns
    │   └── SKILL.md
    ├── cohere-java-sdk/         # Java SDK patterns
    │   └── SKILL.md
    ├── cohere-go-sdk/           # Go SDK patterns
    │   └── SKILL.md
    ├── cohere-embeddings/       # Embedding best practices
    │   └── SKILL.md
    ├── cohere-rerank/           # Reranking patterns
    │   └── SKILL.md
    ├── cohere-streaming/        # Streaming implementation
    │   └── SKILL.md
    ├── cohere-structured-outputs/ # JSON mode & schemas
    │   └── SKILL.md
    ├── cohere-langchain/        # LangChain integration
    │   └── SKILL.md
    ├── cohere-langgraph/        # LangGraph agents
    │   └── SKILL.md
    ├── cohere-cookbooks/        # Official cookbook links
    │   └── SKILL.md
    └── cohere-best-practices/   # Production guidelines
        └── SKILL.md
```

## Skill format

### SKILL.md structure

```markdown
---
name: skill-name
description: One sentence describing when to use this skill. Include trigger phrases.
---

# Skill Title

## Official Resources

- **Docs & Cookbooks**: https://github.com/cohere-ai/cohere-developer-experience
- **API Reference**: https://docs.cohere.com/reference/about

## Content sections...
```

### Naming conventions

- **Skill directory:** kebab-case (e.g., `cohere-python-sdk`, `cohere-embeddings`)
- **SKILL.md:** Always uppercase, exact filename
- **Scripts:** kebab-case.sh or kebab-case.py
- **References:** UPPERCASE.md or descriptive-name.md

### Content guidelines

1. **Link to official docs** — Always include Official Resources section
2. **Copy-paste ready** — Code examples should work without modification
3. **Show imports** — Always include import statements in examples
4. **Highlight gotchas** — Call out common mistakes (e.g., input_type mismatch)
5. **Use tables** — For model comparisons, parameters, etc.

## Writing guidelines

### For code examples

Always show working code with imports:

```python
import cohere

co = cohere.ClientV2()

response = co.chat(
    model="command-a-03-2025",
    messages=[{"role": "user", "content": "Hello"}]
)
print(response.message.content[0].text)
```

### For critical gotchas

Use blockquotes for important warnings:

```markdown
> **Critical**: Mismatched `input_type` between indexing and querying will degrade search quality significantly.
```

### For model/parameter tables

```markdown
| Model | Use Case | Notes |
|-------|----------|-------|
| `command-a-03-2025` | General chat | Latest Command A |
| `embed-english-v3.0` | English embeddings | Best for English-only |
```

## Cohere-specific patterns

### Always use ClientV2

```python
# Correct
co = cohere.ClientV2()

# Deprecated - avoid
co = cohere.Client()
```

### Asymmetric embeddings

```python
# Indexing documents
co.embed(texts=docs, input_type="search_document", ...)

# Querying
co.embed(texts=[query], input_type="search_query", ...)
```

### Two-stage retrieval

```python
# Stage 1: Broad retrieval
candidates = vectorstore.similarity_search(query, k=30)

# Stage 2: Precise reranking
reranked = co.rerank(query=query, documents=candidates, top_n=5)
```

## Priority rules

When helping users with Cohere:

1. **Use official patterns** — Follow Cohere's recommended approaches
2. **Avoid deprecated APIs** — Use ClientV2, not Client
3. **Include error handling** — Show ApiError handling patterns
4. **Link to docs** — Reference official documentation for deep dives

## Updating skills

When updating existing skills:

1. Verify code examples still work with latest API
2. Check for deprecated patterns
3. Update model names if newer versions available
4. Keep Official Resources links current
