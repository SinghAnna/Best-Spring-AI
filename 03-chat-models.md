# Part 3: Spring AI — Chat Models & Providers

> ChatModel ka core — **provider abstraction (OpenAI/Anthropic/Ollama/Bedrock) + `ChatModel` interface (`call()` + `stream()`) + provider swap (code same) + fallback chain** — ek hi `ChatClient` API, multiple providers. [2183][2189][2190]

---

## 1. ChatModel Mental Model

```text
ChatModel = Provider abstraction
= "Kaunsa LLM" (OpenAI / Anthropic / Ollama)

ChatClient = "Kaise call karein" (fluent API)
ChatModel  = "Kaunsa provider" (implementation)
```

### Provider Abstraction

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
   +--> AzureOpenAiChatModel
```

ChatModel — provider abstraction, `call()` + `stream()` methods. [2183][2189]

### ChatModel Interface

```java
public interface ChatModel extends Model<Prompt, ChatResponse>, StreamingChatModel {
    default String call(String message) { ... }
    ChatResponse call(Prompt prompt);
}
```

`ChatModel` — `call(String)` shortcut + `call(Prompt)` full + `stream()`. [2183]

---

## 2. Supported Providers

| Provider | Starter | Model | Use Case |
|---|---|---|---|
| OpenAI | `spring-ai-starter-model-openai` | GPT-4o, GPT-4o-mini | General, smart |
| Anthropic | `spring-ai-starter-model-anthropic` | Claude 3.5 Sonnet | Long context, reasoning |
| Ollama | `spring-ai-starter-model-ollama` | Llama 3.2, Mistral | Local, free, private |
| Bedrock | `spring-ai-starter-model-bedrock` | Claude, Llama, Titan | AWS-native |
| Azure OpenAI | `spring-ai-starter-model-azure-openai` | GPT-4o | Azure-native |
| Vertex AI | `spring-ai-starter-model-vertex-ai-gemini` | Gemini | GCP-native |

Providers — OpenAI, Anthropic, Ollama, Bedrock, Azure, Vertex. [2183][2190]

### Provider Selection Guide

```text
Dev / Local:      Ollama (free, private)
Prod (general):   OpenAI (GPT-4o-mini, cost-effective)
Prod (reasoning): Anthropic (Claude, long context)
AWS shop:         Bedrock
Azure shop:       Azure OpenAI
GCP shop:         Vertex AI
```

Provider selection — dev (Ollama), prod (OpenAI/Anthropic), cloud-native (Bedrock/Azure/Vertex). [2190]

---

## 3. Provider Configuration

### OpenAI

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o-mini
          temperature: 0.7
```

### Anthropic

```yaml
spring:
  ai:
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        options:
          model: claude-3-5-sonnet-20241022
          temperature: 0.7
```

### Ollama (Local)

```yaml
spring:
  ai:
    ollama:
      base-url: http://localhost:11434
      chat:
        options:
          model: llama3.2
```

### Bedrock

```yaml
spring:
  ai:
    bedrock:
      region: us-east-1
      chat:
        options:
          model: anthropic.claude-3-5-sonnet-20241022-v2:0
```

Provider config — `spring.ai.<provider>.*` properties. [2183][2190]

---

## 4. Provider Switching (Profiles)

```text
Profile = Environment-based provider switch
```

### application.yml (Default — Ollama)

```yaml
spring:
  ai:
    ollama:
      base-url: http://localhost:11434
      chat:
        options:
          model: llama3.2
```

### application-prod.yml (Prod — OpenAI)

```yaml
spring:
  ai:
    ollama:
      enabled: false
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o-mini
```

### Run

```bash
# Dev (Ollama)
mvn spring-boot:run

# Prod (OpenAI)
mvn spring-boot:run -Dspring-boot.run.profiles=prod
```

Profiles — dev (Ollama) / prod (OpenAI), code same. [2190]

---

## 5. Fallback Chain

```text
Fallback = Primary fail -> Secondary
```

### Fallback Pattern

```java
@Service
public class ResilientChatService {

    private final ChatClient primary;   // OpenAI
    private final ChatClient fallback;  // Ollama

    public ResilientChatService(
        @Qualifier("openAiClient") ChatClient primary,
        @Qualifier("ollamaClient") ChatClient fallback) {
        this.primary = primary;
        this.fallback = fallback;
    }

    public String chat(String message) {
        try {
            return primary.prompt().user(message).call().content();
        } catch (Exception e) {
            return fallback.prompt().user(message).call().content();
        }
    }
}
```

Fallback — primary fail → secondary (resilience). [2190]

### Resilience4j Circuit Breaker

```java
@CircuitBreaker(name = "llm", fallbackMethod = "fallbackChat")
public String chat(String message) {
    return primary.prompt().user(message).call().content();
}

public String fallbackChat(String message, Throwable t) {
    return fallback.prompt().user(message).call().content();
}
```

Circuit breaker — fail threshold → fallback (avoid cascade). [2190]

---

## 6. Model Comparison

| Model | Strength | Weakness | Cost |
|---|---|---|---|
| GPT-4o | General, fast, multimodal | Cost | $$$ |
| GPT-4o-mini | Fast, cheap | Less capable | $ |
| Claude 3.5 Sonnet | Long context, reasoning | Cost | $$$ |
| Llama 3.2 (Ollama) | Free, private, local | Less capable | Free |
| Gemini 1.5 Pro | Long context, multimodal | Cost | $$$ |

Model comparison — strength, weakness, cost. [2190]

### Model Routing (Cost Optimization)

```java
public String chat(String message, boolean complex) {
    ChatClient client = complex ? smartClient : fastClient;
    return client.prompt().user(message).call().content();
}
```

Routing — simple → fast model, complex → smart model (cost optimize). [2190]

---

## 7. Full Working Example

### Multi-Model Config

```java
package com.example.chatmodels.config;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.openai.OpenAiChatModel;
import org.springframework.ai.ollama.OllamaChatModel;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

@Configuration
public class MultiModelConfig {

    @Bean
    @Primary
    public ChatClient openAiClient(OpenAiChatModel model) {
        return ChatClient.create(model);
    }

    @Bean
    public ChatClient ollamaClient(OllamaChatModel model) {
        return ChatClient.create(model);
    }
}
```

### Resilient ChatService

```java
package com.example.chatmodels.service;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;

@Service
public class ResilientChatService {

    private final ChatClient primary;
    private final ChatClient fallback;

    public ResilientChatService(
        @Qualifier("openAiClient") ChatClient primary,
        @Qualifier("ollamaClient") ChatClient fallback) {
        this.primary = primary;
        this.fallback = fallback;
    }

    public String chat(String message) {
        try {
            return primary.prompt().user(message).call().content();
        } catch (Exception e) {
            return fallback.prompt().user(message).call().content();
        }
    }
}
```

### ChatController

```java
package com.example.chatmodels.controller;

import com.example.chatmodels.service.ResilientChatService;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/chat")
public class ChatController {

    private final ResilientChatService chatService;

    public ChatController(ResilientChatService chatService) {
        this.chatService = chatService;
    }

    public record ChatRequest(String message) {}
    public record ChatResponse(String reply) {}

    @PostMapping
    public ChatResponse chat(@RequestBody ChatRequest req) {
        return new ChatResponse(chatService.chat(req.message()));
    }
}
```

---

## 8. Run + Test

```bash
# 1. Ollama start (local)
ollama serve

# 2. API keys
export OPENAI_API_KEY=sk-...

# 3. Run
mvn spring-boot:run

# 4. Test (OpenAI primary, Ollama fallback)
curl -X POST http://localhost:8080/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"What is Java?"}'
```

---

## 9. Common Gotchas

### Gotcha 1 — Multiple @Primary

```text
Problem:
Ambiguous injection

Fix:
Ek @Primary + @Qualifier for others
```

### Gotcha 2 — Provider-Specific Code

```text
Problem:
Vendor lock-in

Fix:
ChatClient abstraction (code same)
```

### Gotcha 3 — No Fallback

```text
Problem:
Provider outage = app down

Fix:
Fallback chain + circuit breaker
```

### Gotcha 4 — Wrong Model Name

```text
Problem:
Deprecated model (gpt-4)

Fix:
Current model (gpt-4o-mini)
```

---

## 10. Interview Questions

| Question | Answer |
|---|---|
| ChatModel kya? | Provider abstraction. |
| Providers? | OpenAI, Anthropic, Ollama, Bedrock, Azure, Vertex. |
| Provider switch? | Profiles + ChatModel abstraction. |
| Fallback? | Primary fail → secondary. |
| Model routing? | Simple → fast, complex → smart. |
| Ollama kya? | Local LLM runtime (free). |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Local dev | Ollama |
| Prod | OpenAI / Bedrock |
| Resilience | Fallback + circuit breaker |
| Cost optimize | Model routing |
| Multi-cloud | Bedrock / Azure / Vertex |

---

## 11. Key Takeaways

```text
1. ChatModel = provider abstraction.

2. Providers = OpenAI, Anthropic, Ollama, Bedrock.

3. Profiles = env-based switch.

4. Fallback = resilience.

5. Circuit breaker = cascade prevent.

6. Model routing = cost optimize.

7. Ollama = local, free.

8. ChatClient = unified API.

9. @Primary + @Qualifier = multi-model.

10. Provider swap = code same.
```

---

## 12. Next Part Preview

**Part 4: Prompts & Prompt Templates**

Isme hum cover karenge:

```text
- Prompt engineering deep dive
- PromptTemplate
- Variables + placeholders
- Few-shot prompting
- Chain-of-thought
```