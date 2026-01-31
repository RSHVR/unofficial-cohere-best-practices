# Cohere Go SDK Reference

## Table of Contents
1. [Installation](#installation)
2. [Client Setup](#client-setup)
3. [Chat API](#chat-api)
4. [Streaming](#streaming)
5. [Embeddings](#embeddings)
6. [Reranking](#reranking)
7. [Tool Use](#tool-use)
8. [Error Handling](#error-handling)

## Installation

```bash
go get github.com/cohere-ai/cohere-go/v2
```

## Client Setup

```go
package main

import (
    "context"
    "os"

    cohere "github.com/cohere-ai/cohere-go/v2"
    client "github.com/cohere-ai/cohere-go/v2/client"
)

func main() {
    // Option 1: Auto-read from CO_API_KEY env var
    co := client.NewClient()

    // Option 2: Explicit API key
    co := client.NewClient(
        client.WithToken("your-api-key"),
    )

    // Option 3: Custom endpoint
    co := client.NewClient(
        client.WithToken("your-api-key"),
        client.WithBaseURL("https://your-deployment.com"),
    )
}
```

## Chat API

### Basic Chat
```go
import (
    cohere "github.com/cohere-ai/cohere-go/v2"
)

ctx := context.Background()

response, err := co.V2.Chat(ctx, &cohere.V2ChatRequest{
    Model: "command-a-03-2025",
    Messages: []*cohere.ChatMessageV2{
        {
            Role: "user",
            User: &cohere.UserMessage{
                Content: &cohere.UserMessageContent{
                    String: cohere.String("What is machine learning?"),
                },
            },
        },
    },
})
if err != nil {
    log.Fatal(err)
}

fmt.Println(response.Message.Content[0].Text)
```

### With System Message
```go
response, err := co.V2.Chat(ctx, &cohere.V2ChatRequest{
    Model: "command-a-03-2025",
    Messages: []*cohere.ChatMessageV2{
        {
            Role: "system",
            System: &cohere.SystemMessage{
                Content: cohere.String("You are a helpful coding assistant."),
            },
        },
        {
            Role: "user",
            User: &cohere.UserMessage{
                Content: &cohere.UserMessageContent{
                    String: cohere.String("Write a Go hello world"),
                },
            },
        },
    },
})
```

### Parameters
```go
temperature := 0.7
maxTokens := 500

response, err := co.V2.Chat(ctx, &cohere.V2ChatRequest{
    Model:       "command-a-03-2025",
    Messages:    messages,
    Temperature: &temperature,
    MaxTokens:   &maxTokens,
    P:           cohere.Float64(0.9),
    K:           cohere.Int(50),
    Seed:        cohere.Int(42),
    StopSequences: []string{"END"},
})
```

## Streaming

```go
stream, err := co.V2.ChatStream(ctx, &cohere.V2ChatStreamRequest{
    Model: "command-a-03-2025",
    Messages: []*cohere.ChatMessageV2{
        {
            Role: "user",
            User: &cohere.UserMessage{
                Content: &cohere.UserMessageContent{
                    String: cohere.String("Write a poem"),
                },
            },
        },
    },
})
if err != nil {
    log.Fatal(err)
}
defer stream.Close()

for {
    event, err := stream.Recv()
    if err != nil {
        break
    }

    switch event.Type {
    case "content-delta":
        if event.Delta != nil && event.Delta.Message != nil {
            fmt.Print(event.Delta.Message.Content.Text)
        }
    case "message-end":
        fmt.Println("\nGeneration complete")
    }
}
```

## Embeddings

### Basic Embedding
```go
response, err := co.V2.Embed(ctx, &cohere.V2EmbedRequest{
    Model:          "embed-v4.0",
    Texts:          []string{"Hello world", "Machine learning"},
    InputType:      cohere.EmbedInputTypeSearchDocument.Ptr(),
    EmbeddingTypes: []cohere.EmbeddingType{cohere.EmbeddingTypeFloat},
})
if err != nil {
    log.Fatal(err)
}

embeddings := response.Embeddings.Float
fmt.Printf("Dimensions: %d\n", len(embeddings[0]))
```

### Input Types (CRITICAL)
```go
// For STORING documents
docResponse, err := co.V2.Embed(ctx, &cohere.V2EmbedRequest{
    Model:          "embed-v4.0",
    Texts:          documents,
    InputType:      cohere.EmbedInputTypeSearchDocument.Ptr(),  // MUST use
    EmbeddingTypes: []cohere.EmbeddingType{cohere.EmbeddingTypeFloat},
})

// For QUERYING
queryResponse, err := co.V2.Embed(ctx, &cohere.V2EmbedRequest{
    Model:          "embed-v4.0",
    Texts:          []string{userQuery},
    InputType:      cohere.EmbedInputTypeSearchQuery.Ptr(),  // MUST use
    EmbeddingTypes: []cohere.EmbeddingType{cohere.EmbeddingTypeFloat},
})
```

### Batch Processing
```go
func embedBatched(ctx context.Context, co *client.Client, texts []string, batchSize int) ([][]float64, error) {
    var allEmbeddings [][]float64

    for i := 0; i < len(texts); i += batchSize {
        end := i + batchSize
        if end > len(texts) {
            end = len(texts)
        }
        batch := texts[i:end]

        response, err := co.V2.Embed(ctx, &cohere.V2EmbedRequest{
            Model:          "embed-v4.0",
            Texts:          batch,
            InputType:      cohere.EmbedInputTypeSearchDocument.Ptr(),
            EmbeddingTypes: []cohere.EmbeddingType{cohere.EmbeddingTypeFloat},
        })
        if err != nil {
            return nil, err
        }

        allEmbeddings = append(allEmbeddings, response.Embeddings.Float...)
    }

    return allEmbeddings, nil
}

// Usage: embedBatched(ctx, co, largeTextSlice, 96)
```

## Reranking

### Basic Reranking
```go
topN := 3

response, err := co.V2.Rerank(ctx, &cohere.V2RerankRequest{
    Model: "rerank-v4.0-pro",
    Query: "What is machine learning?",
    Documents: []*cohere.RerankRequestDocumentsItem{
        {String: cohere.String("Machine learning is a subset of AI...")},
        {String: cohere.String("The weather is sunny today...")},
        {String: cohere.String("Deep learning uses neural networks...")},
    },
    TopN: &topN,
})
if err != nil {
    log.Fatal(err)
}

for _, result := range response.Results {
    fmt.Printf("Index: %d, Score: %.4f\n", result.Index, result.RelevanceScore)
}
```

### Two-Stage Retrieval
```go
// Stage 1: Fast retrieval with embeddings
candidates, _ := vectorStore.SimilaritySearch(query, 30)

// Stage 2: Precise reranking
docs := make([]*cohere.RerankRequestDocumentsItem, len(candidates))
for i, c := range candidates {
    docs[i] = &cohere.RerankRequestDocumentsItem{
        String: cohere.String(c.Content),
    }
}

topN := 10
reranked, err := co.V2.Rerank(ctx, &cohere.V2RerankRequest{
    Model:     "rerank-v4.0-pro",
    Query:     query,
    Documents: docs,
    TopN:      &topN,
})

finalDocs := make([]Document, len(reranked.Results))
for i, r := range reranked.Results {
    finalDocs[i] = candidates[r.Index]
}
```

## Tool Use

### Define Tools
```go
tools := []*cohere.ToolV2{
    {
        Type: cohere.String("function"),
        Function: &cohere.ToolV2Function{
            Name:        cohere.String("get_weather"),
            Description: cohere.String("Get current weather for a location"),
            Parameters: map[string]interface{}{
                "type": "object",
                "properties": map[string]interface{}{
                    "location": map[string]interface{}{
                        "type":        "string",
                        "description": "City name",
                    },
                },
                "required": []string{"location"},
            },
        },
    },
}

functionsMap := map[string]func(map[string]interface{}) interface{}{
    "get_weather": func(args map[string]interface{}) interface{} {
        return map[string]string{
            "temperature": "20°C",
            "condition":   "sunny",
            "location":    args["location"].(string),
        }
    },
}
```

### Execute Tool Calls
```go
func runWithTools(ctx context.Context, co *client.Client, userMessage string) (string, error) {
    messages := []*cohere.ChatMessageV2{
        {
            Role: "user",
            User: &cohere.UserMessage{
                Content: &cohere.UserMessageContent{
                    String: cohere.String(userMessage),
                },
            },
        },
    }

    response, err := co.V2.Chat(ctx, &cohere.V2ChatRequest{
        Model:    "command-a-03-2025",
        Messages: messages,
        Tools:    tools,
    })
    if err != nil {
        return "", err
    }

    for len(response.Message.ToolCalls) > 0 {
        // Add assistant message
        messages = append(messages, &cohere.ChatMessageV2{
            Role: "assistant",
            Assistant: &cohere.AssistantMessage{
                ToolPlan:  response.Message.ToolPlan,
                ToolCalls: response.Message.ToolCalls,
            },
        })

        // Execute tools
        for _, tc := range response.Message.ToolCalls {
            var args map[string]interface{}
            json.Unmarshal([]byte(*tc.Function.Arguments), &args)

            result := functionsMap[*tc.Function.Name](args)
            resultJSON, _ := json.Marshal(result)

            messages = append(messages, &cohere.ChatMessageV2{
                Role: "tool",
                Tool: &cohere.ToolMessage{
                    ToolCallId: tc.Id,
                    Content: []*cohere.ToolContent{
                        {
                            Type: cohere.String("document"),
                            Document: &cohere.Document{
                                Data: map[string]string{"result": string(resultJSON)},
                            },
                        },
                    },
                },
            })
        }

        // Get next response
        response, err = co.V2.Chat(ctx, &cohere.V2ChatRequest{
            Model:    "command-a-03-2025",
            Messages: messages,
            Tools:    tools,
        })
        if err != nil {
            return "", err
        }
    }

    return *response.Message.Content[0].Text, nil
}
```

## Error Handling

```go
import (
    "errors"
    core "github.com/cohere-ai/cohere-go/v2/core"
)

response, err := co.V2.Chat(ctx, request)
if err != nil {
    var apiErr *core.APIError
    if errors.As(err, &apiErr) {
        fmt.Printf("API Error: %d - %s\n", apiErr.StatusCode, apiErr.Message)
    } else {
        fmt.Printf("Error: %v\n", err)
    }
    return
}
```

## Environment Variables

```bash
export CO_API_KEY="your-api-key"
```
