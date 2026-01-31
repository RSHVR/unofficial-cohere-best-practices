---
name: cohere-structured-outputs
description: Cohere structured outputs reference for JSON mode, JSON Schema validation, and strict tool parameters. Guarantees valid output structure for data extraction, classification, and function calling.
---

# Cohere Structured Outputs Reference

## Official Resources

- **Docs & Cookbooks**: https://github.com/cohere-ai/cohere-developer-experience
- **API Reference**: https://docs.cohere.com/reference/about

## Overview

Structured Outputs forces the LLM's response to strictly follow a user-specified schema, guaranteeing valid output 100% of the time.

| Approach | Use Case | Parameter |
|----------|----------|-----------|
| **JSON Mode** | Simple JSON output | `response_format={"type": "json_object"}` |
| **JSON Schema Mode** | Guaranteed schema compliance | `response_format` with `schema` |
| **Strict Tools** | Function call schema compliance | `strict_tools=True` |

## JSON Mode

```python
import cohere
import json

co = cohere.ClientV2()

response = co.chat(
    model="command-a-03-2025",
    messages=[{
        "role": "user",
        "content": "Generate a JSON describing a person with 'name' and 'age' fields"
    }],
    response_format={"type": "json_object"}
)

data = json.loads(response.message.content[0].text)
```

> **Important**: Your message MUST explicitly instruct the model to generate JSON. Otherwise, the model may generate infinite characters.

## JSON Schema Mode

```python
response = co.chat(
    model="command-a-03-2025",
    messages=[{
        "role": "user",
        "content": "Extract book info from: The Great Gatsby by F. Scott Fitzgerald, published 1925"
    }],
    response_format={
        "type": "json_object",
        "schema": {
            "type": "object",
            "required": ["title", "author", "publication_year"],
            "properties": {
                "title": {"type": "string"},
                "author": {"type": "string"},
                "publication_year": {"type": "integer"}
            }
        }
    }
)
# Guaranteed: {"title": "The Great Gatsby", "author": "F. Scott Fitzgerald", "publication_year": 1925}
```

### Nested Objects and Arrays

```python
response = co.chat(
    model="command-a-03-2025",
    messages=[{"role": "user", "content": "List 3 programming languages with their paradigms"}],
    response_format={
        "type": "json_object",
        "schema": {
            "type": "object",
            "required": ["languages"],
            "properties": {
                "languages": {
                    "type": "array",
                    "items": {
                        "type": "object",
                        "required": ["name", "paradigm"],
                        "properties": {
                            "name": {"type": "string"},
                            "paradigm": {
                                "type": "string",
                                "enum": ["functional", "object-oriented", "procedural", "multi-paradigm"]
                            }
                        }
                    }
                }
            }
        }
    }
)
```

## Strict Tool Parameters

Eliminates parameter hallucinations in function calling:

```python
tools = [{
    "type": "function",
    "function": {
        "name": "search_products",
        "description": "Search the product catalog",
        "parameters": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "Search terms"},
                "category": {
                    "type": "string",
                    "enum": ["electronics", "clothing", "books", "home"]
                },
                "max_price": {"type": "number"}
            },
            "required": ["query"]  # At least one required field needed
        }
    }
}]

response = co.chat(
    model="command-a-03-2025",
    messages=[{"role": "user", "content": "Find cheap electronics under $50"}],
    tools=tools,
    strict_tools=True  # Guarantees tool calls match schema exactly
)
```

### strict_tools Requirements
- Each tool must have **at least one required parameter**
- Maximum **200 fields** across all tools in a single call
- Only supported in Chat API v2 (`ClientV2`)

## Schema Constraints

### Unsupported JSON Schema Features

| Feature | Status |
|---------|--------|
| `anyOf`, `allOf`, `oneOf`, `not` | Not supported |
| `minimum`, `maximum` (numeric ranges) | Not supported |
| `minItems`, `maxItems` (array length) | Not supported |
| `minLength`, `maxLength` (string length) | Not supported |

### Supported String Formats
```python
{"type": "string", "format": "date-time"}  # ISO 8601 datetime
{"type": "string", "format": "uuid"}        # UUID format
{"type": "string", "format": "date"}        # ISO 8601 date
{"type": "string", "format": "time"}        # ISO 8601 time
```

## Common Patterns

### Data Extraction
```python
def extract_entities(text: str) -> dict:
    response = co.chat(
        model="command-a-03-2025",
        messages=[{"role": "user", "content": f"Extract entities from: {text}"}],
        response_format={
            "type": "json_object",
            "schema": {
                "type": "object",
                "required": ["entities"],
                "properties": {
                    "entities": {
                        "type": "array",
                        "items": {
                            "type": "object",
                            "required": ["text", "type"],
                            "properties": {
                                "text": {"type": "string"},
                                "type": {"type": "string", "enum": ["person", "organization", "location", "date"]}
                            }
                        }
                    }
                }
            }
        }
    )
    return json.loads(response.message.content[0].text)
```

### Classification
```python
def classify_sentiment(text: str) -> dict:
    response = co.chat(
        model="command-a-03-2025",
        messages=[{"role": "user", "content": f"Analyze the sentiment of: {text}"}],
        response_format={
            "type": "json_object",
            "schema": {
                "type": "object",
                "required": ["sentiment", "confidence"],
                "properties": {
                    "sentiment": {"type": "string", "enum": ["positive", "negative", "neutral"]},
                    "confidence": {"type": "number"},
                    "reasoning": {"type": "string"}
                }
            }
        }
    )
    return json.loads(response.message.content[0].text)
```

## Performance Note

First requests with a new schema incur additional latency for schema processing. Subsequent requests with the same schema are faster due to caching.
