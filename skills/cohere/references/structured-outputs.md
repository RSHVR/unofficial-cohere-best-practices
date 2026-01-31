# Cohere Structured Outputs Reference

## Table of Contents
1. [Overview](#overview)
2. [JSON Mode](#json-mode)
3. [JSON Schema Mode](#json-schema-mode)
4. [Strict Tool Parameters](#strict-tool-parameters)
5. [Schema Constraints](#schema-constraints)
6. [When to Use What](#when-to-use-what)

## Overview

Structured Outputs forces the LLM's response to strictly follow a user-specified schema, guaranteeing valid output 100% of the time. This eliminates hallucinated fields and ensures reliable downstream processing.

**Compatible models:**
- Command A (`command-a-03-2025`)
- Command R+ 08 2024 (`command-r-plus-08-2024`)
- Command R 08 2024 (`command-r-08-2024`)
- Command R/R+

**Two approaches:**
| Approach | Use Case | Parameter |
|----------|----------|-----------|
| **Structured Outputs (JSON)** | Text generation, data extraction | `response_format` |
| **Structured Outputs (Tools)** | Function calling, agents | `strict_tools` |

## JSON Mode

Basic JSON mode guarantees valid JSON output without a specific schema:

```python
import cohere

co = cohere.ClientV2()

response = co.chat(
    model="command-a-03-2025",
    messages=[{
        "role": "user",
        "content": "Generate a JSON describing a person with 'name' and 'age' fields"
    }],
    response_format={"type": "json_object"}
)

import json
data = json.loads(response.message.content[0].text)
print(data)  # {"name": "Emma Johnson", "age": 32}
```

> **Important**: When using `{"type": "json_object"}`, your message MUST explicitly instruct the model to generate JSON (e.g., "Generate a JSON..."). Otherwise, the model may generate infinite characters and exhaust context.

### JSON Mode Limitations
- Nesting limited to **5 levels** without schema
- Not supported in RAG mode
- Model determines the exact structure

## JSON Schema Mode

For guaranteed schema compliance, provide a `schema` alongside `json_object`:

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

# Guaranteed output structure:
# {"title": "The Great Gatsby", "author": "F. Scott Fitzgerald", "publication_year": 1925}
```

### Nested Objects

JSON Schema mode supports unlimited nesting levels:

```python
response = co.chat(
    model="command-a-03-2025",
    messages=[{"role": "user", "content": "Generate a company profile"}],
    response_format={
        "type": "json_object",
        "schema": {
            "type": "object",
            "required": ["company", "employees"],
            "properties": {
                "company": {
                    "type": "object",
                    "required": ["name"],
                    "properties": {
                        "name": {"type": "string"},
                        "founded": {"type": "integer"},
                        "headquarters": {
                            "type": "object",
                            "required": ["city"],
                            "properties": {
                                "city": {"type": "string"},
                                "country": {"type": "string"}
                            }
                        }
                    }
                },
                "employees": {
                    "type": "array",
                    "items": {
                        "type": "object",
                        "required": ["name", "role"],
                        "properties": {
                            "name": {"type": "string"},
                            "role": {"type": "string"}
                        }
                    }
                }
            }
        }
    }
)
```

### Arrays and Enums

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
                            },
                            "year_created": {"type": "integer"}
                        }
                    }
                }
            }
        }
    }
)
```

## Strict Tool Parameters

For tool/function calling, use `strict_tools=True` to eliminate parameter hallucinations:

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
                    "enum": ["electronics", "clothing", "books", "home"],
                    "description": "Product category filter"
                },
                "max_price": {"type": "number", "description": "Maximum price"}
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

# Tool calls will NEVER include hallucinated parameters
```

### strict_tools Requirements
- Each tool must have **at least one required parameter**
- Maximum **200 fields** across all tools in a single call
- Only supported in Chat API v2 (`ClientV2`)

## Schema Constraints

### Must Follow
- Top-level `type` must be `object`
- Every object must have at least one `required` field

### Unsupported JSON Schema Features

| Feature | Status |
|---------|--------|
| `anyOf`, `allOf`, `oneOf`, `not` | Not supported |
| `minimum`, `maximum` (numeric ranges) | Not supported |
| `minItems`, `maxItems` (array length) | Not supported |
| `minLength`, `maxLength` (string length) | Not supported |
| `uniqueItems` | Not supported |
| `additionalProperties` | Not supported |
| Regex anchors (`^`, `$`, `?=`, `?!`) | Not supported |

### Supported String Formats
```python
# These formats ARE supported
{
    "type": "string",
    "format": "date-time"  # ISO 8601 datetime
}

{
    "type": "string",
    "format": "uuid"  # UUID format
}

{
    "type": "string",
    "format": "date"  # ISO 8601 date
}

{
    "type": "string",
    "format": "time"  # ISO 8601 time
}
```

## When to Use What

| Use Case | Approach | Parameter |
|----------|----------|-----------|
| Simple JSON output, model chooses structure | JSON mode | `response_format={"type": "json_object"}` |
| Guaranteed schema for text generation | JSON Schema mode | `response_format` with `schema` |
| Guaranteed schema for function calls | Strict Tools | `strict_tools=True` |
| Maximum flexibility with JSON output | JSON mode | `response_format={"type": "json_object"}` |
| API response parsing | JSON Schema mode | Define exact response structure |
| Agent tool calls | Strict Tools | Prevents hallucinated params |

## Common Patterns

### Data Extraction
```python
def extract_entities(text: str) -> dict:
    response = co.chat(
        model="command-a-03-2025",
        messages=[{
            "role": "user",
            "content": f"Extract entities from this text: {text}"
        }],
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
                                "type": {"type": "string", "enum": ["person", "organization", "location", "date"]},
                                "confidence": {"type": "number"}
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
        messages=[{
            "role": "user",
            "content": f"Analyze the sentiment of: {text}"
        }],
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

## Performance Notes

> **Latency**: First requests with a new schema incur additional latency for schema processing. Subsequent requests with the same schema are faster due to caching.

```python
# First call: schema processing overhead
response1 = co.chat(..., response_format={"type": "json_object", "schema": my_schema})

# Subsequent calls: schema is cached, faster
response2 = co.chat(..., response_format={"type": "json_object", "schema": my_schema})
```
