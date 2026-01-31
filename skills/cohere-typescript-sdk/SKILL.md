---
name: cohere-typescript-sdk
description: Cohere TypeScript/JavaScript SDK reference for chat, streaming, embeddings, reranking, and tool use. Use when building Node.js or browser applications with Cohere APIs.
---

# Cohere TypeScript SDK Reference

## Official Resources

- **Docs & Cookbooks**: https://github.com/cohere-ai/cohere-developer-experience
- **API Reference**: https://docs.cohere.com/reference/about

## Installation

```bash
npm install cohere-ai
# or yarn add cohere-ai
# or pnpm add cohere-ai
```

## Client Setup

```typescript
import { CohereClientV2 } from "cohere-ai";

// Option 1: Auto-read from CO_API_KEY env var
const cohere = new CohereClientV2({});

// Option 2: Explicit API key
const cohere = new CohereClientV2({
  token: "your-api-key",
});

// Option 3: Custom endpoint (private deployment)
const cohere = new CohereClientV2({
  token: "your-api-key",
  environment: "https://your-deployment.com",
});
```

## Chat API

### Basic Chat
```typescript
const response = await cohere.chat({
  model: "command-a-03-2025",
  messages: [
    { role: "user", content: "What is machine learning?" }
  ],
});

console.log(response.message.content[0].text);
```

### With System Message
```typescript
const response = await cohere.chat({
  model: "command-a-03-2025",
  messages: [
    { role: "system", content: "You are a helpful coding assistant." },
    { role: "user", content: "Write a TypeScript hello world" }
  ],
});
```

### Parameters
```typescript
const response = await cohere.chat({
  model: "command-a-03-2025",
  messages: [{ role: "user", content: "Write a story" }],
  temperature: 0.7,
  maxTokens: 500,
  p: 0.9,
  k: 50,
  seed: 42,
  stopSequences: ["END"],
});
```

## Streaming

```typescript
const stream = await cohere.chatStream({
  model: "command-a-03-2025",
  messages: [{ role: "user", content: "Write a poem about AI" }],
});

for await (const event of stream) {
  if (event.type === "content-delta") {
    process.stdout.write(event.delta?.message?.content?.text ?? "");
  }
}
```

## Embeddings

### Basic Embedding
```typescript
const response = await cohere.embed({
  model: "embed-v4.0",
  texts: ["Hello world", "Machine learning is cool"],
  inputType: "search_document",
  embeddingTypes: ["float"],
});

const embeddings = response.embeddings.float;
```

### Input Types (CRITICAL)
```typescript
// For STORING documents in vector DB
const docEmbeddings = await cohere.embed({
  model: "embed-v4.0",
  texts: documents,
  inputType: "search_document",  // MUST use for docs
  embeddingTypes: ["float"],
});

// For USER QUERIES searching against docs
const queryEmbedding = await cohere.embed({
  model: "embed-v4.0",
  texts: [userQuery],
  inputType: "search_query",  // MUST use for queries
  embeddingTypes: ["float"],
});
```

### Batch Processing (96 item limit)
```typescript
async function embedBatched(texts: string[], batchSize = 96): Promise<number[][]> {
  const allEmbeddings: number[][] = [];

  for (let i = 0; i < texts.length; i += batchSize) {
    const batch = texts.slice(i, i + batchSize);
    const response = await cohere.embed({
      model: "embed-v4.0",
      texts: batch,
      inputType: "search_document",
      embeddingTypes: ["float"],
    });
    allEmbeddings.push(...(response.embeddings.float ?? []));
  }

  return allEmbeddings;
}
```

## Reranking

```typescript
const response = await cohere.rerank({
  model: "rerank-v4.0-pro",
  query: "What is machine learning?",
  documents: [
    "Machine learning is a subset of AI...",
    "The weather today is sunny...",
    "Deep learning uses neural networks...",
  ],
  topN: 3,
});

for (const result of response.results) {
  console.log(`Index: ${result.index}, Score: ${result.relevanceScore}`);
}
```

## Tool Use

### Define Tools
```typescript
const tools = [
  {
    type: "function" as const,
    function: {
      name: "get_weather",
      description: "Get current weather for a location",
      parameters: {
        type: "object",
        properties: {
          location: {
            type: "string",
            description: "City name, e.g., 'Toronto'",
          },
        },
        required: ["location"],
      },
    },
  },
];

const functionsMap: Record<string, (args: any) => any> = {
  get_weather: ({ location }) => ({
    temperature: "20°C",
    condition: "sunny",
    location,
  }),
};
```

### Execute Tool Calls
```typescript
import { ChatMessageV2 } from "cohere-ai/api";

async function runWithTools(userMessage: string) {
  const messages: ChatMessageV2[] = [
    { role: "user", content: userMessage }
  ];

  let response = await cohere.chat({
    model: "command-a-03-2025",
    messages,
    tools,
  });

  while (response.message.toolCalls && response.message.toolCalls.length > 0) {
    messages.push({
      role: "assistant",
      toolPlan: response.message.toolPlan,
      toolCalls: response.message.toolCalls,
    });

    for (const tc of response.message.toolCalls) {
      const args = JSON.parse(tc.function.arguments ?? "{}");
      const result = functionsMap[tc.function.name](args);

      messages.push({
        role: "tool",
        toolCallId: tc.id,
        content: [
          {
            type: "document",
            document: { data: JSON.stringify(result) },
          },
        ],
      });
    }

    response = await cohere.chat({
      model: "command-a-03-2025",
      messages,
      tools,
    });
  }

  return response.message.content[0].text;
}
```

## Structured Outputs

### JSON Schema Mode
```typescript
const response = await cohere.chat({
  model: "command-a-03-2025",
  messages: [
    { role: "user", content: "Extract: John is 30 years old" }
  ],
  responseFormat: {
    type: "json_object",
    jsonSchema: {
      type: "object",
      properties: {
        name: { type: "string" },
        age: { type: "integer" },
      },
      required: ["name", "age"],
    },
  },
});
```

## Error Handling

```typescript
import { CohereError, CohereTimeoutError } from "cohere-ai";

try {
  const response = await cohere.chat({
    model: "command-a-03-2025",
    messages: [{ role: "user", content: "Hello" }],
  });
} catch (err) {
  if (err instanceof CohereTimeoutError) {
    console.error("Request timed out");
  } else if (err instanceof CohereError) {
    console.error(`API Error: ${err.statusCode} - ${err.message}`);
  } else {
    throw err;
  }
}
```

## Vercel AI SDK Integration

```typescript
import { cohere } from "@ai-sdk/cohere";
import { generateText, embed } from "ai";

const { text } = await generateText({
  model: cohere("command-a-03-2025"),
  prompt: "Hello!",
});

const { embedding } = await embed({
  model: cohere.embedding("embed-v4.0"),
  value: "Your text here",
});
```
