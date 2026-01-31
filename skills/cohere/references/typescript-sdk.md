# Cohere TypeScript SDK Reference

## Table of Contents
1. [Installation](#installation)
2. [Client Setup](#client-setup)
3. [Chat API](#chat-api)
4. [Streaming](#streaming)
5. [Embeddings](#embeddings)
6. [Reranking](#reranking)
7. [Tool Use](#tool-use)
8. [Structured Outputs](#structured-outputs)
9. [Error Handling](#error-handling)

## Installation

```bash
npm install cohere-ai
# or
yarn add cohere-ai
# or
pnpm add cohere-ai
```

## Client Setup

### Basic Setup
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

### Multi-turn Conversation
```typescript
const messages = [
  { role: "user" as const, content: "My name is Veer" },
  { role: "assistant" as const, content: "Nice to meet you, Veer!" },
  { role: "user" as const, content: "What's my name?" }
];

const response = await cohere.chat({
  model: "command-a-03-2025",
  messages,
});
```

### Parameters
```typescript
const response = await cohere.chat({
  model: "command-a-03-2025",
  messages: [{ role: "user", content: "Write a story" }],
  temperature: 0.7,        // 0.0-1.0, higher = more creative
  maxTokens: 500,          // Max response length
  p: 0.9,                  // Top-p sampling
  k: 50,                   // Top-k sampling
  seed: 42,                // For reproducibility
  stopSequences: ["END"],  // Stop generation at these
});
```

## Streaming

### Basic Streaming
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

### Streaming Event Types
```typescript
const stream = await cohere.chatStream({
  model: "command-a-03-2025",
  messages,
});

for await (const event of stream) {
  switch (event.type) {
    case "message-start":
      console.log("Generation started");
      break;
    case "content-start":
      console.log("Content block started");
      break;
    case "content-delta":
      process.stdout.write(event.delta?.message?.content?.text ?? "");
      break;
    case "content-end":
      console.log("\nContent block ended");
      break;
    case "message-end":
      console.log("Generation complete");
      // Access final response: event.response
      break;
    case "tool-plan-delta":
      console.log(`Tool plan: ${event.delta?.message?.toolPlan}`);
      break;
    case "tool-call-start":
      console.log("Tool call started");
      break;
    case "tool-call-delta":
      console.log("Tool args delta");
      break;
    case "tool-call-end":
      console.log("Tool call complete");
      break;
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
console.log(`Embedding dimensions: ${embeddings[0].length}`);
```

### With Matryoshka Dimensions (Embed v4)
```typescript
const response = await cohere.embed({
  model: "embed-v4.0",
  texts: ["Your text here"],
  inputType: "search_document",
  embeddingTypes: ["float"],
  outputDimension: 512,  // Options: 256, 512, 1024, 1536
});
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

### Basic Reranking
```typescript
const response = await cohere.rerank({
  model: "rerank-v4.0-pro",
  query: "What is machine learning?",
  documents: [
    "Machine learning is a subset of AI that enables systems to learn from data.",
    "The weather today is sunny with clear skies.",
    "Deep learning uses neural networks with many layers.",
  ],
  topN: 3,
});

for (const result of response.results) {
  console.log(`Index: ${result.index}, Score: ${result.relevanceScore}`);
}
```

### Two-Stage Retrieval Pattern
```typescript
// Stage 1: Fast retrieval with embeddings
const candidates = await vectorStore.similaritySearch(query, 30);

// Stage 2: Precise reranking
const reranked = await cohere.rerank({
  model: "rerank-v4.0-pro",
  query,
  documents: candidates.map(doc => doc.content),
  topN: 10,
});

const finalDocs = reranked.results.map(r => candidates[r.index]);
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

// Tool implementation
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
    // Add assistant message with tool calls
    messages.push({
      role: "assistant",
      toolPlan: response.message.toolPlan,
      toolCalls: response.message.toolCalls,
    });

    // Execute tools and add results
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

    // Get next response
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

### JSON Mode
```typescript
const response = await cohere.chat({
  model: "command-a-03-2025",
  messages: [
    { role: "user", content: "Generate a JSON with name and age for a person" }
  ],
  responseFormat: { type: "json_object" },
});

const data = JSON.parse(response.message.content[0].text);
```

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

### Strict Tools
```typescript
const response = await cohere.chat({
  model: "command-a-03-2025",
  messages,
  tools,
  strictTools: true,  // Guarantees tool calls match schema exactly
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

## Environment Variables

```bash
export CO_API_KEY="your-api-key"
```

## Vercel AI SDK Integration

If using Vercel AI SDK, use `@ai-sdk/cohere`:

```typescript
import { cohere } from "@ai-sdk/cohere";
import { generateText, embed } from "ai";

// Chat
const { text } = await generateText({
  model: cohere("command-a-03-2025"),
  prompt: "Hello!",
});

// Embeddings
const { embedding } = await embed({
  model: cohere.embedding("embed-v4.0"),
  value: "Your text here",
});
```
