---
name: cohere-java-sdk
description: Cohere Java SDK reference for chat, streaming, embeddings, reranking, and tool use. Use when building Java/Kotlin applications with Cohere APIs.
---

# Cohere Java SDK Reference

## Official Resources

- **Docs & Cookbooks**: https://github.com/cohere-ai/cohere-developer-experience
- **API Reference**: https://docs.cohere.com/reference/about

## Installation

### Maven
```xml
<dependency>
  <groupId>com.cohere</groupId>
  <artifactId>cohere-java</artifactId>
  <version>1.x.x</version>
</dependency>
```

### Gradle
```groovy
implementation 'com.cohere:cohere-java:1.x.x'
```

## Client Setup

```java
import com.cohere.api.Cohere;

// Option 1: Auto-read from CO_API_KEY env var
Cohere cohere = Cohere.builder().build();

// Option 2: Explicit API key
Cohere cohere = Cohere.builder()
    .token("your-api-key")
    .build();

// Option 3: Custom endpoint
Cohere cohere = Cohere.builder()
    .token("your-api-key")
    .environment(Environment.custom("https://your-deployment.com"))
    .build();
```

## Chat API

### Basic Chat
```java
import com.cohere.api.types.*;
import com.cohere.api.requests.*;
import java.util.List;

ChatResponse response = cohere.v2().chat(
    V2ChatRequest.builder()
        .model("command-a-03-2025")
        .messages(List.of(
            ChatMessageV2.user(
                UserMessage.builder()
                    .content(UserMessageContent.of("What is machine learning?"))
                    .build()
            )
        ))
        .build()
);

System.out.println(response.getMessage().getContent().get(0).getText());
```

### With System Message
```java
ChatResponse response = cohere.v2().chat(
    V2ChatRequest.builder()
        .model("command-a-03-2025")
        .messages(List.of(
            ChatMessageV2.system(
                SystemMessage.builder()
                    .content("You are a helpful coding assistant.")
                    .build()
            ),
            ChatMessageV2.user(
                UserMessage.builder()
                    .content(UserMessageContent.of("Write a Java hello world"))
                    .build()
            )
        ))
        .build()
);
```

### Parameters
```java
ChatResponse response = cohere.v2().chat(
    V2ChatRequest.builder()
        .model("command-a-03-2025")
        .messages(messages)
        .temperature(0.7)
        .maxTokens(500)
        .p(0.9)
        .k(50)
        .seed(42)
        .stopSequences(List.of("END"))
        .build()
);
```

## Streaming

```java
import com.cohere.api.types.StreamedChatResponseV2;

Stream<StreamedChatResponseV2> stream = cohere.v2().chatStream(
    V2ChatStreamRequest.builder()
        .model("command-a-03-2025")
        .messages(List.of(
            ChatMessageV2.user(
                UserMessage.builder()
                    .content(UserMessageContent.of("Write a poem"))
                    .build()
            )
        ))
        .build()
);

stream.forEach(event -> {
    if (event.isContentDelta()) {
        event.getContentDelta().ifPresent(delta -> {
            System.out.print(delta.getDelta().getMessage().getContent().getText());
        });
    }
});
```

## Embeddings

### Basic Embedding
```java
EmbedResponse response = cohere.v2().embed(
    V2EmbedRequest.builder()
        .model("embed-v4.0")
        .texts(List.of("Hello world", "Machine learning"))
        .inputType(EmbedInputType.SEARCH_DOCUMENT)
        .embeddingTypes(List.of(EmbeddingType.FLOAT))
        .build()
);

List<List<Double>> embeddings = response.getEmbeddings().getFloat_();
System.out.println("Dimensions: " + embeddings.get(0).size());
```

### Input Types (CRITICAL)
```java
// For STORING documents
EmbedResponse docResponse = cohere.v2().embed(
    V2EmbedRequest.builder()
        .model("embed-v4.0")
        .texts(documents)
        .inputType(EmbedInputType.SEARCH_DOCUMENT)  // MUST use for docs
        .embeddingTypes(List.of(EmbeddingType.FLOAT))
        .build()
);

// For QUERYING
EmbedResponse queryResponse = cohere.v2().embed(
    V2EmbedRequest.builder()
        .model("embed-v4.0")
        .texts(List.of(userQuery))
        .inputType(EmbedInputType.SEARCH_QUERY)  // MUST use for queries
        .embeddingTypes(List.of(EmbeddingType.FLOAT))
        .build()
);
```

## Reranking

```java
RerankResponse response = cohere.v2().rerank(
    V2RerankRequest.builder()
        .model("rerank-v4.0-pro")
        .query("What is machine learning?")
        .documents(List.of(
            RerankRequestDocumentsItem.of("Machine learning is a subset of AI..."),
            RerankRequestDocumentsItem.of("The weather is sunny today..."),
            RerankRequestDocumentsItem.of("Deep learning uses neural networks...")
        ))
        .topN(3)
        .build()
);

for (RerankResponseResultsItem result : response.getResults()) {
    System.out.printf("Index: %d, Score: %.4f%n",
        result.getIndex(), result.getRelevanceScore());
}
```

## Tool Use

### Define Tools
```java
List<ToolV2> tools = List.of(
    ToolV2.builder()
        .type("function")
        .function(ToolV2Function.builder()
            .name("get_weather")
            .description("Get current weather for a location")
            .parameters(Map.of(
                "type", "object",
                "properties", Map.of(
                    "location", Map.of(
                        "type", "string",
                        "description", "City name"
                    )
                ),
                "required", List.of("location")
            ))
            .build())
        .build()
);
```

### Execute Tool Calls
```java
public String runWithTools(String userMessage) {
    List<ChatMessageV2> messages = new ArrayList<>();
    messages.add(ChatMessageV2.user(
        UserMessage.builder()
            .content(UserMessageContent.of(userMessage))
            .build()
    ));

    ChatResponse response = cohere.v2().chat(
        V2ChatRequest.builder()
            .model("command-a-03-2025")
            .messages(messages)
            .tools(tools)
            .build()
    );

    while (response.getMessage().getToolCalls() != null
           && !response.getMessage().getToolCalls().isEmpty()) {

        messages.add(ChatMessageV2.assistant(
            AssistantMessage.builder()
                .toolPlan(response.getMessage().getToolPlan())
                .toolCalls(response.getMessage().getToolCalls())
                .build()
        ));

        for (ToolCallV2 tc : response.getMessage().getToolCalls()) {
            Map<String, Object> args = parseJson(tc.getFunction().getArguments());
            Object result = executeFunction(tc.getFunction().getName(), args);

            messages.add(ChatMessageV2.tool(
                ToolMessage.builder()
                    .toolCallId(tc.getId())
                    .content(List.of(
                        ToolContent.document(
                            Document.builder()
                                .data(Map.of("result", result))
                                .build()
                        )
                    ))
                    .build()
            ));
        }

        response = cohere.v2().chat(
            V2ChatRequest.builder()
                .model("command-a-03-2025")
                .messages(messages)
                .tools(tools)
                .build()
        );
    }

    return response.getMessage().getContent().get(0).getText();
}
```

## Error Handling

```java
import com.cohere.api.core.CohereApiException;

try {
    ChatResponse response = cohere.v2().chat(request);
} catch (CohereApiException e) {
    System.err.printf("API Error: %d - %s%n",
        e.statusCode(), e.getMessage());
} catch (Exception e) {
    System.err.println("Error: " + e.getMessage());
}
```
