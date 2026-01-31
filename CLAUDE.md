# CLAUDE.md

This file provides guidance to Claude Code when working with this repository.

## Project summary

Agent Skills is a collection of skills for AI coding assistants working with Cohere's AI APIs. The skills provide focused, actionable reference documentation for chat, embeddings, reranking, streaming, and agent workflows. Skills follow the [Agent Skills specification](https://agentskills.io/specification).

## Quick reference

### Available skills

| Skill | Location | Triggers |
|-------|----------|----------|
| cohere-python-sdk | `skills/cohere-python-sdk/` | "cohere python", "cohere sdk", "ClientV2" |
| cohere-typescript-sdk | `skills/cohere-typescript-sdk/` | "cohere typescript", "cohere js" |
| cohere-java-sdk | `skills/cohere-java-sdk/` | "cohere java", "cohere maven" |
| cohere-go-sdk | `skills/cohere-go-sdk/` | "cohere go", "cohere golang" |
| cohere-embeddings | `skills/cohere-embeddings/` | "embed", "embeddings", "vector search" |
| cohere-rerank | `skills/cohere-rerank/` | "rerank", "reranking", "two-stage retrieval" |
| cohere-streaming | `skills/cohere-streaming/` | "stream", "streaming", "real-time" |
| cohere-structured-outputs | `skills/cohere-structured-outputs/` | "json mode", "structured output", "schema" |
| cohere-langchain | `skills/cohere-langchain/` | "langchain", "ChatCohere", "CohereEmbeddings" |
| cohere-langgraph | `skills/cohere-langgraph/` | "langgraph", "react agent", "agent memory" |
| cohere-cookbooks | `skills/cohere-cookbooks/` | "cookbook", "tutorial", "example" |
| cohere-best-practices | `skills/cohere-best-practices/` | "best practices", "production", "optimization" |

### Key patterns

**API Client:**
- Always use `cohere.ClientV2()` (not the deprecated `Client()`)

**Embeddings:**
- Use `input_type="search_document"` when indexing
- Use `input_type="search_query"` when querying
- Mismatch degrades search quality significantly

**Two-Stage Retrieval:**
1. Embeddings for broad recall (k=30)
2. Reranking for precision (top_n=5-10)

**Agent Temperature:**
- Use `temperature=0.3` for reliable tool calling

## Common tasks

### Adding a new skill

1. Create directory: `skills/{skill-name}/`
2. Create `SKILL.md` with YAML frontmatter
3. Add optional `scripts/`, `references/`, `assets/`
4. Update `README.md` skills table

### Updating a skill

1. Edit relevant `SKILL.md` file
2. Keep focused and actionable
3. Link to official docs for detailed content
4. Include working code examples

## Code style

- Use kebab-case for directories
- YAML frontmatter required on all SKILL.md files
- Include "Official Resources" section linking to Cohere docs
- Code examples should be copy-paste ready
- Always show imports in examples

## External resources

- **Official Docs**: https://docs.cohere.com/reference/about
- **Cookbooks**: https://github.com/cohere-ai/cohere-developer-experience
- **Dashboard**: https://dashboard.cohere.com
