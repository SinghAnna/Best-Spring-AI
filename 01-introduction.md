# Part 1: Spring AI — Introduction & Setup

> Spring AI ka core — **ChatClient (fluent API, WebClient/RestClient jaisa) + BOM (version management) + starter (`spring-ai-starter-model-openai`) + auto-config (`ChatClient.Builder` bean)** — 5-step quick start: project → dependency → API key → ChatClient → run. [2166][2167][2170][2174]

---

## 1. Spring AI Mental Model

```text
Spring Boot = Web apps ka framework
Spring AI   = AI apps ka framework
             (same patterns: auto-config, starters, abstractions)

Traditional:
OpenAI SDK + custom HTTP + provider-specific code
= Vendor lock-in

Spring AI:
ChatClient (unified) -> ChatModel (provider abstraction)
= Provider swap (OpenAI <-> Ollama <-> Anthropic)
```

### Spring AI = Spring Ecosystem Extension

```text
Spring Boot patterns -> Spring AI patterns
- Starters        -> spring-ai-starter-model-openai
- Auto-config     -> ChatClient.Builder bean
- Properties      -> spring.ai.openai.api-key
- Profiles        -> dev (Ollama) / prod (OpenAI)
- Observability   -> Micrometer + OTel
```

Spring AI — Spring idioms (starters, auto-config, properties) AI ke liye. [2170][2175]

---

## 2. Core Concepts

| Concept | What | Analogy |
|---|---|---|
| ChatClient | Fluent API for LLM calls | WebClient (HTTP) |
| ChatModel | Provider abstraction | RestClient (HTTP) |
| Prompt | LLM input | Request body |
| Advisor | Middleware around call | Filter/Interceptor |
| EmbeddingModel | Text → vector | Encoder |
| VectorStore | Vector storage + search | Database |

ChatClient — fluent API, ChatModel — provider abstraction, Advisor — middleware. [2154][2173][2174]

### Architecture

```text
Your Code
   |
   v
ChatClient (fluent API)
   |
   v
ChatModel (interface)
   |
   +--> OpenAiChatModel
   +--> AnthropicChatModel
   +--> OllamaChatModel
   +--> BedrockChatModel
```

ChatClient → ChatModel → provider-specific impl. [2154][2167]

---

## 3. Project Setup (Spring Initializr)

```text
[https://start.spring.io](https://start.spring.io)

Project: Maven
Language: Java
Spring Boot: 3.4+
Group: com.example
Artifact: hello-ai
Packaging: Jar
Java: 17+

Dependencies:
- Spring Web
- Spring Boot Actuator
- Spring AI: OpenAI
```

Spring Initializr — AI Models + Vector Stores select kar sakte ho. [2170][2173]

---

## 4. Full pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         [https://maven.apache.org/xsd/maven-4.0.0.xsd](https://maven.apache.org/xsd/maven-4.0.0.xsd)">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.4.0</version>
        <relativePath/>
    </parent>

    <groupId>com.example</groupId>
    <artifactId>hello-ai</artifactId>
    <version>1.0.0</version>
    <name>hello-ai</name>

    <properties>
        <java.version>17</java.version>
        <spring-ai.version>1.0.0</spring-ai.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.ai</groupId>
                <artifactId>spring-ai-bom</artifactId>
                <version>${spring-ai.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <!-- Web -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Actuator (health + metrics) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <!-- Spring AI OpenAI starter -->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-starter-model-openai</artifactId>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

BOM — version management (no per-dependency version), starter — auto-config. [2166][2167][2171]

---

## 5. application.yml

```yaml
spring:
  application:
    name: hello-ai
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o-mini
          temperature: 0.7

server:
  port: 8080

management:
  endpoints:
    web:
      exposure:
        include: health,metrics
```

### Environment Variable (Secure)

```bash
# .env (gitignore me)
export OPENAI_API_KEY=sk-...

# Run
source .env && mvn spring-boot:run
```

API key — env var (never hardcode), `${OPENAI_API_KEY}` SpEL reference. [2167][2174]

### application.yml (Local Ollama — Free)

```yaml
spring:
  ai:
    ollama:
      base-url: http://localhost:11434
      chat:
        options:
          model: llama3.2
```

```bash
# Ollama install + run
ollama pull llama3.2
ollama serve
```

Ollama — local, free, no API key (dev ke liye). [2173][2175]

---

## 6. Main Application

```java
package com.example.helloai;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class HelloAiApplication {

    public static void main(String[] args) {
        SpringApplication.run(HelloAiApplication.class, args);
    }
}
```

---

## 7. ChatController (First LLM Call)

```java
package com.example.helloai.controller;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api")
public class ChatController {

    private final ChatClient chatClient;

    public ChatController(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    public record ChatRequest(String message) {}
    public record ChatResponse(String reply) {}

    @PostMapping(path = "/chat", consumes = MediaType.APPLICATION_JSON_VALUE)
    public ChatResponse chat(@RequestBody ChatRequest request) {
        String reply = chatClient
            .prompt()
            .user(request.message())
            .call()
            .content();
        return new ChatResponse(reply);
    }

    @GetMapping("/chat")
    public ChatResponse chatGet(
        @RequestParam(defaultValue = "Tell me a joke") String message) {
        return chat(new ChatRequest(message));
    }
}
```

ChatClient.Builder — auto-configured bean, `.prompt().user().call().content()` — fluent chain. [2170][2174]

### Code Walkthrough

```text
chatClient.prompt()      -> New prompt
       .user(msg)        -> User message set
       .call()           -> Sync call (blocking)
       .content()        -> Response text extract
```

---

## 8. Run + Test

```bash
# 1. API key set
export OPENAI_API_KEY=sk-...

# 2. Run
mvn spring-boot:run

# 3. Test
curl "http://localhost:8080/api/chat?message=Tell%20me%20a%20joke"

# Response:
{"reply":"Why don't scientists trust atoms? Because they make up everything!"}
```

---

## 9. Production-Ready Structure

```text
hello-ai/
├── src/main/java/com/example/helloai/
│   ├── HelloAiApplication.java
│   ├── controller/
│   │   └── ChatController.java
│   ├── service/
│   │   └── ChatService.java
│   └── config/
│       └── AiConfig.java
├── src/main/resources/
│   ├── application.yml
│   └── application-local.yml
├── src/test/java/
├── .env (gitignore)
├── .gitignore
├── Dockerfile
└── pom.xml
```

### ChatService (Service Layer)

```java
package com.example.helloai.service;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.stereotype.Service;

@Service
public class ChatService {

    private final ChatClient chatClient;

    public ChatService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    public String chat(String message) {
        return chatClient
            .prompt()
            .user(message)
            .call()
            .content();
    }
}
```

### ChatController (Thin)

```java
@RestController
@RequestMapping("/api")
public class ChatController {

    private final ChatService chatService;

    public ChatController(ChatService chatService) {
        this.chatService = chatService;
    }

    @GetMapping("/chat")
    public ChatResponse chat(@RequestParam String message) {
        return new ChatResponse(chatService.chat(message));
    }
}
```

Service layer — business logic separate, controller thin. [2174]

---

## 10. Dockerfile

```dockerfile
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

```bash
mvn clean package
docker build -t hello-ai:latest .
docker run -e OPENAI_API_KEY=$OPENAI_API_KEY -p 8080:8080 hello-ai:latest
```

---

## 11. Common Gotchas

### Gotcha 1 — API Key Hardcoded

```text
Problem:
application.yml me key commit

Fix:
Env var + .gitignore
```

### Gotcha 2 — No BOM

```text
Problem:
Version mismatch

Fix:
spring-ai-bom import
```

### Gotcha 3 — Model Name Wrong

```text
Problem:
gpt-4 (deprecated)

Fix:
gpt-4o-mini (current)
```

### Gotcha 4 — No Health Check

```text
Problem:
Prod me blind

Fix:
Actuator + health endpoint
```

---

## 12. Interview Questions

| Question | Answer |
|---|---|
| Spring AI kya? | Spring ecosystem ka AI framework. |
| ChatClient kya? | Fluent API for LLM calls. |
| ChatModel kya? | Provider abstraction. |
| BOM kya? | Version management. |
| API key kahan? | Env var (never hardcode). |
| Ollama kya? | Local LLM runtime (free). |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Provider swap | ChatModel abstraction |
| Local dev | Ollama |
| Prod | OpenAI / Bedrock |
| Secrets | Env var + Vault |

---

## 13. Key Takeaways

```text
1. Spring AI = Spring idioms + AI.

2. ChatClient = fluent API.

3. ChatModel = provider abstraction.

4. BOM = version management.

5. Starter = auto-config.

6. API key = env var.

7. Ollama = local dev.

8. Service layer = business logic.

9. Actuator = health + metrics.

10. Docker = deployment.
```

---

## 14. Next Part Preview

**Part 2: ChatClient Deep Dive**

Isme hum cover karenge:

```text
- ChatClient fluent API deep dive
- System vs user messages
- call() vs stream()
- ChatOptions (temperature, maxTokens)
- Multiple ChatClient beans
```