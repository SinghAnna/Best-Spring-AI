# Part 17: Spring AI — Persistent Memory (JDBC/Redis Deep Dive)

> Persistent memory ka core — **`JdbcChatMemoryRepository` (PostgreSQL/MySQL, `initialize-schema: always`, dialect auto-detect) + Redis (Redisson, JSON doc + index, TTL per-entry) + multi-instance (shared memory, rolling deployment safe)** — Redis = sub-ms latency + TTL, JDBC = durable + SQL queries + backups. [2369][2371][2373][2375]

---

## 1. Persistent Memory Mental Model

```text
Problem:
In-memory memory
= Restart / rolling deploy = conversation loss
= Multi-instance = memory split (user hits different pod)

Solution:
Persistent memory
= JDBC (PostgreSQL/MySQL) ya Redis
= Shared across instances
= Restart-safe
```

### In-Memory vs Persistent

| Aspect | In-Memory | JDBC / Redis |
|---|---|---|
| Restart | Loss | Safe |
| Multi-instance | Split | Shared |
| Latency | Fastest | Low (Redis) / Medium (JDBC) |
| Durability | No | Yes |
| Use case | Dev / test | Prod |

In-memory (dev) vs JDBC/Redis (prod) — restart-safe + shared. [2372][2375]

### Rolling Deployment Problem

```text
Rolling deploy:
Pod A (memory) -> Terminate
Pod B (new)    -> Start
= Pod A conversations lost

Persistent:
Pod A -> DB
Pod B -> DB (same)
= Conversations safe
```

Rolling deploy — in-memory loss, persistent safe. [2375]

---

## 2. JDBC Chat Memory (PostgreSQL/MySQL)

```text
JDBC = SQL database
= Durable + queryable + backup-able
```

### Dependency

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-chat-memory-repository-jdbc</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

JDBC starter — `JdbcChatMemoryRepository` auto-config (DataSource present). [2375]

### application.yml (PostgreSQL)

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/chatdb
    username: postgres
    password: postgres
  ai:
    chat:
      memory:
        repository:
          jdbc:
            initialize-schema: always  # always | never | embedded
```

`initialize-schema: always` — `CREATE TABLE IF NOT EXISTS` (safe). [2369][2375]

### Schema (Auto-Generated)

```sql
CREATE TABLE IF NOT EXISTS SPRING_AI_CHAT_MEMORY (
    conversation_id VARCHAR(256) NOT NULL,
    content TEXT,
    type VARCHAR(10),
    sequence_id BIGINT,
    created_at TIMESTAMP
);
```

Schema — conversation_id + content + type + sequence_id (order). [2375]

### MySQL Config

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/chatdb?createDatabaseIfNotExist=true&serverTimezone=UTC
    username: root
    password: password
    driver-class-name: com.mysql.cj.jdbc.Driver
  ai:
    chat:
      memory:
        repository:
          jdbc:
            initialize-schema: always
```

MySQL — `createDatabaseIfNotExist=true` (local dev), prod me remove. [2375]

### Dialect Auto-Detect

```text
DataSource -> Metadata -> Dialect detect
= PostgreSQL / MySQL / Oracle / HSQLDB / H2
= DDL script auto-select
```

Dialect auto-detect — DataSource metadata se. [2369][2375]

---

## 3. Redis Chat Memory (Redisson)

```text
Redis = In-memory data store
= Sub-ms latency + TTL + distributed
```

### Dependency

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-chat-memory-repository-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.45.0</version>
</dependency>
```

Redis starter — `RedisChatMemoryRepository` (Redisson). [2371][2373]

### application.yml (Redis)

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
  ai:
    chat:
      memory:
        repository:
          redis:
            ttl: 86400  # 24 hours (seconds)
```

Redis — TTL (per-entry expiry, write pe set, read pe extend nahi). [2371]

### Redis Storage Model

```text
Each message = Redis JSON document
Indexed by Redis Query Engine
= Conversation read (in order)
= Search by content / type / timestamp / metadata
```

Redis — JSON doc + index (content/type/timestamp search). [2373]

### TTL Behavior

```text
TTL = Per-entry (message-level)
Set on write, read pe extend nahi
= Conversation expire (last message + TTL)
```

TTL — per-entry, write pe set, read pe extend nahi. [2371]

### Redis vs JDBC

| Aspect | Redis | JDBC |
|---|---|---|
| Latency | Sub-ms | Medium |
| TTL | Yes (native) | Manual (cron) |
| Query | Limited (index) | Full SQL |
| Backup | RDB/AOF | Full backup |
| Use case | High-frequency chat | Audit / compliance |

Redis (sub-ms + TTL) vs JDBC (SQL + backup + audit). [2373][2376]

---

## 4. Multi-Instance (Shared Memory)

```text
Multi-instance = All pods same DB/Redis
= Conversation ID -> Same memory
```

### Multi-Instance Flow

```text
User -> Load Balancer
   |
   +--> Pod A -> DB/Redis
   +--> Pod B -> DB/Redis
   +--> Pod C -> DB/Redis
= Same conversation, any pod
```

Multi-instance — shared DB/Redis (any pod). [2372][2375]

### Conversation ID (Sticky vs Shared)

```text
Sticky session:
User -> Same pod (session affinity)

Shared memory:
User -> Any pod (DB/Redis shared)
= Better (no sticky needed)
```

Shared memory — no sticky needed (any pod). [2375]

---

## 5. Full Working Example

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
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

### docker-compose.yml

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: chatdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
```

### application.yml

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/chatdb
    username: postgres
    password: postgres
  ai:
    chat:
      memory:
        repository:
          jdbc:
            initialize-schema: always
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
import org.springframework.ai.chat.memory.ChatMemoryRepository;
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

## 6. Run + Test

```bash
# 1. Start PostgreSQL
docker-compose up -d

# 2. API key
export OPENAI_API_KEY=sk-...

# 3. Run (2 instances, shared memory)
mvn spring-boot:run -Dspring-boot.run.arguments=--server.port=8080 &
mvn spring-boot:run -Dspring-boot.run.arguments=--server.port=8081 &

# 4. Turn 1 (instance 1)
curl -X POST http://localhost:8080/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"My name is John"}'

# 5. Turn 2 (instance 2, same conversation)
curl -X POST http://localhost:8081/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"What is my name?","conversationId":"<from-turn-1>"}'

# Response:
{"reply":"Your name is John","conversationId":"..."}
```

---

## 7. Common Gotchas

### Gotcha 1 — initialize-schema in Prod

```text
Problem:
always = startup overhead

Fix:
never (schema pre-created)
```

### Gotcha 2 — No Connection Pool

```text
Problem:
Connection exhaustion

Fix:
HikariCP (default, tune pool size)
```

### Gotcha 3 — Redis TTL Too Short

```text
Problem:
Conversation expire early

Fix:
TTL tune (24h+)
```

### Gotcha 4 — No Index on conversation_id

```text
Problem:
Slow queries

Fix:
Index (auto, but verify)
```

---

## 8. Interview Questions

| Question | Answer |
|---|---|
| Persistent memory kya? | Restart-safe + multi-instance shared. |
| JDBC repository? | PostgreSQL/MySQL/Oracle (dialect auto). |
| initialize-schema? | always / never / embedded. |
| Redis repository? | Redisson (JSON doc + index + TTL). |
| Multi-instance? | Shared DB/Redis (any pod). |
| TTL behavior? | Per-entry, write pe set, read pe extend nahi. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Rolling deploy | JDBC / Redis |
| Multi-instance | Shared DB/Redis |
| High-frequency chat | Redis (sub-ms) |
| Audit / compliance | JDBC (SQL + backup) |
| Session expiry | Redis TTL |

---

## 9. Key Takeaways

```text
1. Persistent = restart-safe + multi-instance.

2. JDBC = PostgreSQL/MySQL/Oracle (dialect auto).

3. initialize-schema = always / never / embedded.

4. Redis = Redisson (JSON doc + index + TTL).

5. TTL = per-entry (write pe set).

6. Multi-instance = shared DB/Redis.

7. Redis = sub-ms latency.

8. JDBC = SQL + backup + audit.

9. Sticky session = not needed (shared).

10. Prod = JDBC ya Redis (in-memory nahi).
```

---

## 10. Next Part Preview

**Part 18: Long-Term Memory (VectorStoreChatMemoryAdvisor)**

Isme hum cover karenge:

```text
- Long-term memory deep dive
- VectorStoreChatMemoryAdvisor
- Semantic recall
- Hybrid memory (short + long)
- Real-world patterns
```