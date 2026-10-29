# Part 16: Spring AI — Chat Memory (Conversation History)

> Chat memory ka core — **`ChatMemory` (retention strategy) + `ChatMemoryRepository` (storage) + `MessageWindowChatMemory` (sliding window, default 20) + `MessageChatMemoryAdvisor` (retrieve → inject → save)** — conversation ID mandatory (nahi to `IllegalArgumentException`), in-memory (default, restart = loss) → JDBC/Redis (persistent). [2357][2359][2361][2364]

---

## 1. Chat Memory Mental Model

```text
Problem:
LLM stateless
= Previous turns yaad nahi
= "What did I ask?" -> "I don't know"

Solution:
Chat Memory = Conversation history
= Retrieve -> Inject -> Save
= Multi-turn context
```

### Memory Flow

```text
1. User message -> ChatClient
2. Advisor -> Memory se history retrieve
3. History + new message -> Prompt
4. LLM -> Response
5. Advisor -> User + Assistant messages save
6. Next turn -> Repeat
```

Memory flow — retrieve → inject → save (advisor auto). [2357][2360][2361]

### Memory vs No Memory

```text
No Memory:
User: "My name is John"
AI: "Nice to meet you!"
User: "What's my name?"
AI: "I don't know"  ❌

With Memory:
User: "My name is John"
AI: "Nice to meet you, John!"
User: "What's my name?"
AI: "Your name is John"  ✅
```

Memory — multi-turn context (name recall). [2360][2361]

---

## 2. Core Components

```text
ChatMemory           = Retention strategy (window)
ChatMemoryRepository = Storage (in-memory / JDBC / Redis)
MessageChatMemoryAdvisor = Glue (retrieve + inject + save)
```

### Components

| Component | Role | Default |
|---|---|---|
| `ChatMemory` | Retention strategy | `MessageWindowChatMemory` |
| `ChatMemoryRepository` | Storage | `InMemoryChatMemoryRepository` |
| `MessageChatMemoryAdvisor` | Glue (advisor) | — |

ChatMemory (retention) + Repository (storage) + Advisor (glue). [2359][2361][2364]

### Separation of Concerns

```text
ChatMemory           = "Kitne messages rakhne hain" (window)
ChatMemoryRepository = "Kahan store karna hai" (in-memory / DB)
MessageChatMemoryAdvisor = "Kab retrieve / save" (advisor)
```

Separation — retention (window) vs storage (repo) vs glue (advisor). [2361][2364]

---

## 3. MessageWindowChatMemory

```text
MessageWindowChatMemory = Sliding window
= Last N messages (default 20)
= Older evict
```

### Window Behavior

```text
maxMessages = 20

Messages 1-20 -> In memory
Message 21    -> Message 1 evict
Message 22    -> Message 2 evict
...
```

Window — last N, older evict (FIFO). [2359][2361]

### System Message Special

```text
New SystemMessage -> Old SystemMessages remove
Window overflow -> SystemMessages preserve (others evict)
```

System message — special (preserve on evict). [2359]

### Code Example

```java
ChatMemory chatMemory = MessageWindowChatMemory.builder()
    .chatMemoryRepository(chatMemoryRepository)
    .maxMessages(10)  // Last 10 messages
    .build();
```

`maxMessages(10)` — last 10 messages (window). [2359][2364]

---

## 4. ChatMemoryRepository (Storage)

```text
Repository = "Kahan store karna hai"
```

### Repository Options

| Repository | Storage | Persistence | Use Case |
|---|---|---|---|
| `InMemoryChatMemoryRepository` | ConcurrentHashMap | No (restart = loss) | Dev / test |
| `JdbcChatMemoryRepository` | PostgreSQL / MySQL / Oracle | Yes | Prod (SQL) |
| Redis (Redisson) | Redis | Yes + TTL | Low-latency + expiry |
| Cassandra / Neo4j / MongoDB | NoSQL | Yes | Specific use cases |
| VectorStore | Vector DB | Yes | Long-term (semantic) |

Repositories — InMemory (dev), JDBC (prod SQL), Redis (low-latency), VectorStore (long-term). [2357][2361][2363][2364]

### InMemoryChatMemoryRepository (Default)

```text
Default = In-memory (ConcurrentHashMap)
= Key: conversation ID
= Value: List<Message>
= Restart = loss
```

Default — InMemory (ConcurrentHashMap), restart = loss. [2361][2364]

### JdbcChatMemoryRepository (Persistent)

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-chat-memory-repository-jdbc</artifactId>
</dependency>
```

```java
@Bean
ChatMemoryRepository chatMemoryRepository(DataSource dataSource) {
    return JdbcChatMemoryRepository.builder()
        .dataSource(dataSource)
        .build();
}
```

JDBC — PostgreSQL / MySQL / SQL Server / HSQLDB / Oracle (auto-detect dialect). [2357][2361][2364]

### JDBC Schema (Auto)

```sql
CREATE TABLE spring_ai_chat_memory (
    conversation_id VARCHAR(36) NOT NULL,
    content TEXT,
    type VARCHAR(10),
    sequence_id BIGINT,
    created_at TIMESTAMP
);
```

JDBC schema — conversation_id + content + type + sequence_id (order). [2361][2368]

### Redis (Redisson)

```java
ChatMemory chatMemory = MessageWindowChatMemory.builder()
    .chatMemoryRepository(chatMemoryRepository)  // Redis-backed
    .maxMessages(20)
    .build();
```

Redis — Redisson `ChatMemoryRepository` (JSON doc + index). [2363]

---

## 5. MessageChatMemoryAdvisor (Glue)

```text
Advisor = Retrieve + Inject + Save (auto)
```

### Advisor Flow

```text
Before call:
1. Conversation ID -> Repository
2. Messages retrieve
3. Prompt me inject

After call:
4. User + Assistant messages save
```

Advisor — before (retrieve + inject) + after (save). [2357][2360][2361]

### Advisor Registration

```java
@Bean
public ChatClient chatClient(ChatModel chatModel, ChatMemory chatMemory) {
    return ChatClient.builder(chatModel)
        .defaultAdvisors(
            MessageChatMemoryAdvisor.builder(chatMemory).build())
        .build();
}
```

`MessageChatMemoryAdvisor.builder(chatMemory)` — default advisor. [2357][2360][2364]

### Conversation ID (Mandatory)

```java
String answer = chatClient.prompt()
    .user(userMessage)
    .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, conversationId))
    .call()
    .content();
```

`ChatMemory.CONVERSATION_ID` — mandatory (nahi to `IllegalArgumentException`). [2357][2361][2364]

### Conversation ID Best Practices

```text
WRONG:  Client se raw ID accept (poisoning)
RIGHT:  Server-side session / auth context
BEST:   userId + chatSessionId (composite)
```

Conversation ID — server-side (poisoning prevent), userId + sessionId (best). [2361][2364]

---

## 6. Long-Term Memory (VectorStoreChatMemoryAdvisor)

```text
Long-term = Semantic recall (old conversations)
= VectorStore-based
```

### VectorStoreChatMemoryAdvisor

```java
@Bean
public ChatClient chatClient(ChatModel chatModel, VectorStore vectorStore) {
    return ChatClient.builder(chatModel)
        .defaultAdvisors(
            VectorStoreChatMemoryAdvisor.builder(vectorStore).build())
        .build();
}
```

`VectorStoreChatMemoryAdvisor` — semantic recall (old conversations). [2357][2358]

### Message vs Vector Memory

| Advisor | Storage | Recall | Use Case |
|---|---|---|---|
| MessageChatMemoryAdvisor | Message store | Full history (window) | Tool agents, chat |
| VectorStoreChatMemoryAdvisor | Vector store | Semantic (relevant) | Long-term, knowledge |

Message (full history) vs Vector (semantic recall). [2357][2358]

### Hybrid (Both)

```text
Short-term: MessageChatMemoryAdvisor (window)
Long-term:  VectorStoreChatMemoryAdvisor (semantic)
= Best of both
```

Hybrid — short-term (window) + long-term (semantic). [2321][2326]

---

## 7. Full Working Example

### pom.xml (Dependencies)

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-chat-memory-repository-jdbc</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

### application.yml

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/chatdb
    username: postgres
    password: postgres
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o-mini
```

### MemoryConfig

```java
package com.example.memory.config;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.memory.ChatMemory;
import org.springframework.ai.chat.memory.MessageWindowChatMemory;
import org.springframework.ai.chat.memory.jdbc.JdbcChatMemoryRepository;
import org.springframework.ai.chat.model.ChatModel;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.sql.DataSource;

@Configuration
public class MemoryConfig {

    @Bean
    public ChatMemoryRepository chatMemoryRepository(DataSource dataSource) {
        return JdbcChatMemoryRepository.builder()
            .dataSource(dataSource)
            .build();
    }

    @Bean
    public ChatMemory chatMemory(ChatMemoryRepository repository) {
        return MessageWindowChatMemory.builder()
            .chatMemoryRepository(repository)
            .maxMessages(20)
            .build();
    }

    @Bean
    public ChatClient chatClient(ChatModel chatModel, ChatMemory chatMemory) {
        return ChatClient.builder(chatModel)
            .defaultSystem("""
                You are a helpful assistant.
                Remember conversation context.
                """)
            .defaultAdvisors(
                MessageChatMemoryAdvisor.builder(chatMemory).build())
            .build();
    }
}
```

### ChatController

```java
package com.example.memory.controller;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.memory.ChatMemory;
import org.springframework.web.bind.annotation.*;

import java.util.UUID;

@RestController
@RequestMapping("/api/chat")
public class ChatController {

    private final ChatClient chatClient;

    public ChatController(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    public record ChatRequest(String message, String conversationId) {}
    public record ChatResponse(String reply, String conversationId) {}

    @PostMapping
    public ChatResponse chat(@RequestBody ChatRequest req) {
        String conversationId = req.conversationId() != null
            ? req.conversationId()
            : UUID.randomUUID().toString();

        String reply = chatClient.prompt()
            .user(req.message())
            .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, conversationId))
            .call()
            .content();

        return new ChatResponse(reply, conversationId);
    }
}
```

---

## 8. Run + Test

```bash
# 1. Start PostgreSQL
docker run -d -p 5432:5432 -e POSTGRES_PASSWORD=postgres postgres:16

# 2. API key
export OPENAI_API_KEY=sk-...

# 3. Run
mvn spring-boot:run

# Turn 1
curl -X POST http://localhost:8080/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"My name is John"}'

# Turn 2 (same conversation)
curl -X POST http://localhost:8080/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"What is my name?","conversationId":"<from-turn-1>"}'

# Response:
{"reply":"Your name is John","conversationId":"..."}
```

---

## 9. Common Gotchas

### Gotcha 1 — No Conversation ID

```text
Problem:
IllegalArgumentException

Fix:
ChatMemory.CONVERSATION_ID param
```

### Gotcha 2 — Client-Side ID

```text
Problem:
Session poisoning

Fix:
Server-side session / auth context
```

### Gotcha 3 — In-Memory in Prod

```text
Problem:
Restart = memory loss

Fix:
JDBC / Redis repository
```

### Gotcha 4 — Window Too Small

```text
Problem:
Context loss

Fix:
maxMessages (20-50)
```

---

## 10. Interview Questions

| Question | Answer |
|---|---|
| Chat memory kya? | Conversation history (multi-turn). |
| ChatMemory vs Repository? | Retention vs storage. |
| MessageWindowChatMemory? | Sliding window (default 20). |
| MessageChatMemoryAdvisor? | Retrieve + inject + save. |
| Conversation ID? | Mandatory (session scoping). |
| Long-term memory? | VectorStoreChatMemoryAdvisor. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Multi-turn chat | MessageChatMemoryAdvisor |
| Persistent | JDBC repository |
| Low-latency + TTL | Redis (Redisson) |
| Long-term recall | VectorStoreChatMemoryAdvisor |
| Session poisoning | Server-side ID |

---

## 11. Key Takeaways

```text
1. Chat memory = multi-turn context.

2. ChatMemory = retention (window).

3. Repository = storage (in-memory / JDBC / Redis).

4. MessageWindowChatMemory = sliding window (20).

5. MessageChatMemoryAdvisor = glue.

6. Conversation ID = mandatory.

7. Server-side ID = security.

8. JDBC = persistent (prod).

9. VectorStore = long-term (semantic).

10. Hybrid = short-term + long-term.
```

---

## 12. Next Part Preview

**Part 17: Persistent Memory (JDBC/Redis Deep Dive)**

Isme hum cover karenge:

```text
- Persistent memory deep dive
- JDBC repository (PostgreSQL/MySQL)
- Redis repository (Redisson)
- Multi-instance (shared memory)
- TTL + expiry
```