# Part 18: Spring AI — Long-Term Memory (VectorStoreChatMemoryAdvisor)

> Long-term memory ka core — **`VectorStoreChatMemoryAdvisor` (semantic recall, `LONG_TERM_MEMORY` system section) + hybrid memory (short-term window + long-term semantic) + memory types (episodic / semantic / procedural) + session-scoped filtering** — short-term = "recent turns", long-term = "what's worth remembering from ALL of them". [2377][2378][2380][2382]

---

## 1. Long-Term Memory Mental Model

```text
Short-term (Part 17):
- Conversation window (last 20)
- Full history (recent turns)
- New chat = empty

Long-term:
- Vector store (semantic)
- Relevant memories (ALL conversations)
- New chat = still recall
```

### Short-Term vs Long-Term

| Aspect | Short-Term | Long-Term |
|---|---|---|
| Storage | Message store | Vector store |
| Recall | Full history (window) | Semantic (relevant) |
| Scope | Conversation | Cross-conversation |
| New chat | Empty | Still recall |
| Use case | Tool agents, chat | Personalization, knowledge |

Short-term (window, conversation) vs Long-term (semantic, cross-conversation). [2378][2380][2382]

### Complementary (Not Replacement)

```text
Repository (short-term) = Recent turns (in order)
Vector store (long-term) = What's worth remembering (from all)
= Both needed
```

Complementary — repository (recent) + vector (worth remembering). [2378]

---

## 2. VectorStoreChatMemoryAdvisor

```text
VectorStoreChatMemoryAdvisor = Semantic recall
= Memories -> System prompt (LONG_TERM_MEMORY section)
```

### How It Works

```text
1. User message -> Embedding
2. VectorStore -> Similarity search (top-K)
3. Memories -> System prompt (LONG_TERM_MEMORY)
4. LLM -> Answer (grounded in memory)
5. New exchange -> VectorStore save
```

Advisor — embed → search → inject (system) → save. [2377]

### Security Note

```text
Memory content = XML-escaped + <memory-entry> wrapped
= Role-boundary injection neutralize
= "Historical data only, not instructions"
```

Security — XML-escape + `<memory-entry>` + "historical data only". [2377]

### Limitation

```text
Text-only exchanges
= Multi-modal (image/audio) not supported
```

Limitation — text-only (multi-modal not supported). [2377]

---

## 3. Memory Types

```text
3 Types:
1. Episodic  = Personal experiences / preferences
2. Semantic  = General facts / domain knowledge
3. Procedural = How-to / workflows
```

### Memory Types

| Type | What | Example |
|---|---|---|
| Episodic | Personal experiences | "User prefers Delta airlines" |
| Semantic | General facts | "Singapore requires passport" |
| Procedural | How-to | "To book, follow these steps" |

Memory types — episodic (personal), semantic (facts), procedural (how-to). [2378][2381][2382]

### Memory Extraction

```text
LLM = Extractor (atomic facts)
= "Only clear, factual information"
= No assumptions / inference
```

Extraction — LLM se atomic facts (clear, factual). [2382]

---

## 4. Hybrid Memory (Short + Long)

```text
Hybrid = Short-term (window) + Long-term (semantic)
= Best of both
```

### Hybrid Flow

```text
User message
   |
   v
1. Short-term: Window se recent turns
2. Long-term: Vector se relevant memories
3. Merge -> Prompt
4. LLM -> Answer
5. Save:
   - Short-term: Message store
   - Long-term: Vector store (facts)
```

Hybrid — short-term (window) + long-term (semantic) + merge. [2378][2382]

### Hybrid Config

```java
@Bean
public ChatClient chatClient(
    ChatModel chatModel, 
    ChatMemory chatMemory,
    VectorStore vectorStore) {
    return ChatClient.builder(chatModel)
        .defaultAdvisors(
            MessageChatMemoryAdvisor.builder(chatMemory).build(),
            VectorStoreChatMemoryAdvisor.builder(vectorStore).build())
        .build();
}
```

Hybrid — MessageChatMemoryAdvisor + VectorStoreChatMemoryAdvisor. [2378][2382]

---

## 5. Session-Scoped Filtering

```text
Session-scoped = Memories per user / session
= filterExpression (sessionId)
```

### Session Filter

```java
List<Document> memories = vectorStore.similaritySearch(
    SearchRequest.builder()
        .query(currentInput)
        .topK(5)
        .similarityThreshold(0.6)
        .filterExpression("sessionId == '" + sessionId + "'")
        .build());
```

Session filter — `sessionId == 'X'` (scoped recall). [2378]

### Memory Metadata

```java
Document memory = new Document(
    memoryText,
    Map.of(
        "sessionId", sessionId,
        "type", type,  // conversation / fact / preference
        "timestamp", Instant.now().toString()
    ));
```

Memory metadata — sessionId + type + timestamp. [2378]

---

## 6. Full Working Example

### pom.xml (Dependencies)

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-vector-store-pgvector</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```

### docker-compose.yml

```yaml
services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: memorydb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
```

### application.yml

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/memorydb
    username: postgres
    password: postgres
  ai:
    vectorstore:
      pgvector:
        dimensions: 1536
        distance-type: cosine_distance
        index-type: hnsw
    openai:
      api-key: ${OPENAI_API_KEY}
```

### MemoryConfig

```java
package com.example.memory.config;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.client.advisor.MessageChatMemoryAdvisor;
import org.springframework.ai.chat.client.advisor.vectorstore.VectorStoreChatMemoryAdvisor;
import org.springframework.ai.chat.memory.ChatMemory;
import org.springframework.ai.chat.memory.MessageWindowChatMemory;
import org.springframework.ai.chat.model.ChatModel;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MemoryConfig {

    @Bean
    public ChatMemory chatMemory() {
        return MessageWindowChatMemory.builder()
            .maxMessages(20)
            .build();
    }

    @Bean
    public ChatClient chatClient(
        ChatModel chatModel, 
        ChatMemory chatMemory,
        VectorStore vectorStore) {
        return ChatClient.builder(chatModel)
            .defaultSystem("""
                You are a helpful assistant with long-term memory.
                Use LONG_TERM_MEMORY for past context.
                Remember user preferences and facts.
                """)
            .defaultAdvisors(
                MessageChatMemoryAdvisor.builder(chatMemory).build(),
                VectorStoreChatMemoryAdvisor.builder(vectorStore).build())
            .build();
    }
}
```

### MemoryService

```java
package com.example.memory.service;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.memory.ChatMemory;
import org.springframework.ai.document.Document;
import org.springframework.ai.vectorstore.SearchRequest;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.stereotype.Service;

import java.time.Instant;
import java.util.List;
import java.util.Map;

@Service
public class MemoryService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;

    public MemoryService(ChatClient chatClient, VectorStore vectorStore) {
        this.chatClient = chatClient;
        this.vectorStore = vectorStore;
    }

    // 1. Chat with hybrid memory
    public String chat(String sessionId, String message) {
        return chatClient.prompt()
            .user(message)
            .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, sessionId))
            .call()
            .content();
    }

    // 2. Remember (explicit fact)
    public void remember(String sessionId, String memoryText, String type) {
        vectorStore.add(List.of(new Document(
            memoryText,
            Map.of(
                "sessionId", sessionId,
                "type", type,
                "timestamp", Instant.now().toString()
            ))));
    }

    // 3. Recall (semantic)
    public List<Document> recall(String sessionId, String query, int topK) {
        return vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(query)
                .topK(topK)
                .similarityThreshold(0.6)
                .filterExpression("sessionId == '" + sessionId + "'")
                .build());
    }
}
```

### MemoryController

```java
package com.example.memory.controller;

import com.example.memory.service.MemoryService;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.UUID;

@RestController
@RequestMapping("/api/memory")
public class MemoryController {

    private final MemoryService memoryService;

    public MemoryController(MemoryService memoryService) {
        this.memoryService = memoryService;
    }

    public record ChatRequest(String message, String sessionId) {}
    public record ChatResponse(String reply, String sessionId) {}
    public record RememberRequest(String sessionId, String memory, String type) {}
    public record RecallRequest(String sessionId, String query, int topK) {}

    @PostMapping("/chat")
    public ChatResponse chat(@RequestBody ChatRequest req) {
        String sessionId = req.sessionId() != null
            ? req.sessionId()
            : UUID.randomUUID().toString();
        String reply = memoryService.chat(sessionId, req.message());
        return new ChatResponse(reply, sessionId);
    }

    @PostMapping("/remember")
    public String remember(@RequestBody RememberRequest req) {
        memoryService.remember(req.sessionId(), req.memory(), req.type());
        return "Remembered";
    }

    @PostMapping("/recall")
    public List<Document> recall(@RequestBody RecallRequest req) {
        return memoryService.recall(req.sessionId(), req.query(), req.topK());
    }
}
```

---

## 7. Run + Test

```bash
# 1. Start pgvector
docker-compose up -d

# 2. API key
export OPENAI_API_KEY=sk-...

# 3. Run
mvn spring-boot:run

# 4. Chat (new session)
curl -X POST http://localhost:8080/api/memory/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"My name is John, I prefer Delta airlines"}'

# 5. Remember (explicit fact)
curl -X POST http://localhost:8080/api/memory/remember \
  -H "Content-Type: application/json" \
  -d '{"sessionId":"<from-chat>","memory":"User prefers window seats","type":"preference"}'

# 6. Recall (semantic)
curl -X POST http://localhost:8080/api/memory/recall \
  -H "Content-Type: application/json" \
  -d '{"sessionId":"<from-chat>","query":"What airline does the user prefer?","topK":3}'
```

---

## 8. Common Gotchas

### Gotcha 1 — No Session Filter

```text
Problem:
Cross-user memory leak

Fix:
filterExpression (sessionId)
```

### Gotcha 2 — Vector Only (No Short-Term)

```text
Problem:
Recent turns miss

Fix:
Hybrid (Message + Vector)
```

### Gotcha 3 — Multi-Modal

```text
Problem:
VectorStoreChatMemoryAdvisor text-only

Fix:
MessageChatMemoryAdvisor (multi-modal)
```

### Gotcha 4 — Memory Pollution

```text
Problem:
Irrelevant memories

Fix:
Similarity threshold (0.6+) + dedup
```

---

## 9. Interview Questions

| Question | Answer |
|---|---|
| Long-term memory kya? | Semantic recall (cross-conversation). |
| VectorStoreChatMemoryAdvisor? | Memories → system prompt (LONG_TERM_MEMORY). |
| Memory types? | Episodic, semantic, procedural. |
| Hybrid memory? | Short-term (window) + long-term (semantic). |
| Session filter? | filterExpression (sessionId). |
| Security? | XML-escape + `<memory-entry>` + "historical only". |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Personalization | Long-term (episodic) |
| Domain knowledge | Long-term (semantic) |
| Multi-turn chat | Short-term (window) |
| Both | Hybrid (Message + Vector) |
| Cross-user leak | Session filter |

---

## 10. Key Takeaways

```text
1. Long-term = semantic recall (cross-conversation).

2. VectorStoreChatMemoryAdvisor = memories → system.

3. Memory types = episodic + semantic + procedural.

4. Hybrid = short-term + long-term.

5. Session filter = sessionId (scoped).

6. Security = XML-escape + historical-only.

7. Text-only = multi-modal not supported.

8. Complementary = repository + vector.

9. Extraction = LLM (atomic facts).

10. Threshold = 0.6+ (relevance).
```

---

## 11. Next Part Preview

**Part 19: Observability (Micrometer + OpenTelemetry)**

Isme hum cover karenge:

```text
- Observability deep dive
- Micrometer + OTel
- Traces + metrics
- Token usage
- Grafana dashboards
```