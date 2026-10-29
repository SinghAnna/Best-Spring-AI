# Part 20: Spring AI — Cost Management (Token Budget + Caching + Model Routing)

> Cost management ka core — **token budget (per-turn / per-conversation / per-day) + semantic cache (vector similarity, 30-50% hit rate) + model routing (simple → cheap, complex → smart) + prompt caching (cache read = 10-20% price)** — `Usage` interface (promptTokens + completionTokens + cacheRead/Write), advisor se analytics persist. [2399][2400][2402][2410]

---

## 1. Cost Management Mental Model

```text
Problem:
LLM APIs = token-based pricing
= Input + output tokens
= Cost blow (uncontrolled)

Solution:
4 Levers:
1. Token budget (caps)
2. Semantic cache (avoid LLM)
3. Model routing (cheap model)
4. Prompt caching (cache read)
```

### Cost Formula

```text
Cost = (uncached_input × input_price) 
     + (cached_input × cache_price) 
     + (output × output_price)
```

Cost formula — uncached input + cached input + output. [2402][2408]

### Cost Optimization Impact

```text
Model routing:     40-60% reduction
Prompt caching:    70-90% (on hits)
Prompt compression: 20-40%
Batch processing:  50% discount
Semantic cache:    30-50% hit rate
= Combined: 60-80% reduction
```

Cost optimization — routing (40-60%), caching (70-90%), compression (20-40%), batch (50%). [2402][2405][2411]

---

## 2. Token Budget (3 Levels)

```text
Budget = Cap (per-turn / per-conversation / per-day)
```

### Budget Levels

| Level | Cap | Trigger |
|---|---|---|
| Per-turn | maxTokens (output) | Single completion |
| Per-conversation | 6,000 tokens | Long conversation → cheap model |
| Per-day (global) | 3× 7-day avg | All traffic → cheap model |

Budget — per-turn (maxTokens), per-conversation (6K), per-day (3× avg). [2410]

### Usage Tracking (Spring AI)

```java
ChatResponse response = chatClient.prompt(prompt).call().chatResponse();
Usage usage = response.getMetadata().getUsage();

long promptTokens = usage.getPromptTokens();
long completionTokens = usage.getCompletionTokens();
long totalTokens = usage.getTotalTokens();
```

`Usage` — promptTokens + completionTokens + totalTokens. [2399][2409]

### Cache Metrics (Unified)

```java
Usage usage = response.getMetadata().getUsage();
Long cacheRead = usage.getCacheReadInputTokens();
Long cacheWrite = usage.getCacheWriteInputTokens();

if (cacheRead != null && cacheRead > 0) {
    log.info("Cache hit: {} tokens read from cache", cacheRead);
}
```

Cache metrics — `getCacheReadInputTokens()` + `getCacheWriteInputTokens()` (unified). [2399]

### Provider Cache Support

| Provider | Cache Read | Cache Write |
|---|---|---|
| Anthropic | Yes | Yes |
| OpenAI | Yes (`cachedTokens`) | No |
| Google Gemini | Yes | No |
| DeepSeek / Mistral / Ollama | No | No |

Cache support — Anthropic (read+write), OpenAI (read), Gemini (read), Ollama (none). [2399]

### Budget Manager (Redis)

```java
@Service
public class TokenBudgetManager {

    private final StringRedisTemplate redis;

    public TokenBudgetManager(StringRedisTemplate redis) {
        this.redis = redis;
    }

    public boolean canAfford(String userId, long estimatedTokens) {
        String key = "budget:" + userId + ":" + LocalDate.now();
        Long used = redis.opsForValue().increment(key, estimatedTokens);
        if (used != null && used == estimatedTokens) {
            redis.expire(key, Duration.ofDays(1));
        }
        return used == null || used <= 100_000;  // 100K daily cap
    }
}
```

Budget manager — Redis counter + daily expiry + cap check. [2402][2410]

---

## 3. Semantic Cache (Vector Similarity)

```text
Semantic cache = Query + Response (vector store)
= Similar query -> Cached response (LLM skip)
```

### Semantic Cache Flow

```text
1. Query -> Embedding
2. Vector store -> Similarity search (threshold)
3. Match? -> Cached response (LLM skip)
4. No match? -> LLM call -> Cache (query + response)
```

Semantic cache — embed → search → match → cached / LLM → cache. [2400][2412]

### Semantic Cache Implementation

```java
@Service
public class SemanticCacheService {

    private final VectorStore vectorStore;
    private final ChatClient chatClient;

    private static final double SIMILARITY_THRESHOLD = 0.92;

    public SemanticCacheService(VectorStore vectorStore, ChatClient chatClient) {
        this.vectorStore = vectorStore;
        this.chatClient = chatClient;
    }

    public String getOrCompute(String question) {
        // 1. Cache check
        List<Document> cached = vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(question)
                .topK(1)
                .similarityThreshold(SIMILARITY_THRESHOLD)
                .build());

        if (!cached.isEmpty()) {
            return cached.get(0).getMetadata().get("response").toString();
        }

        // 2. LLM call
        String response = chatClient.prompt()
            .user(question)
            .call()
            .content();

        // 3. Cache
        vectorStore.add(List.of(new Document(
            question,
            Map.of("response", response))));

        return response;
    }
}
```

Semantic cache — threshold (0.92) + vector store (query → response). [2400][2412]

### Threshold Tuning

```text
0.95+ = Very strict (few hits, high precision)
0.90-0.95 = Balanced (recommended)
0.85-0.90 = Loose (more hits, risk wrong answer)
```

Threshold — 0.92 recommended (balanced). [2400][2403]

### Cache Hit Rate

```text
FAQ-style: 30-50% hit rate
= 30-50% LLM calls skip
= 30-50% cost save
```

Cache hit rate — FAQ 30-50%. [2402][2411]

---

## 4. Model Routing (Cost Optimization)

```text
Routing = Classify (complexity) -> Route (model)
= Simple -> cheap, Complex -> smart
```

### Routing Flow

```text
Query -> Classifier (complexity)
   |
   +--> Simple (FAQ, status) -> GPT-4o-mini ($)
   +--> Medium (analysis)     -> GPT-4o ($$)
   +--> Complex (reasoning)   -> Claude ($$$)
```

Routing — classify → route (cheap → smart). [2402][2411]

### Routing Implementation

```java
@Service
public class ModelRoutingService {

    private final ChatClient fastClient;   // GPT-4o-mini
    private final ChatClient smartClient;  // Claude

    public ModelRoutingService(
        @Qualifier("fastClient") ChatClient fastClient,
        @Qualifier("smartClient") ChatClient smartClient) {
        this.fastClient = fastClient;
        this.smartClient = smartClient;
    }

    public String chat(String message) {
        boolean complex = isComplex(message);
        ChatClient client = complex ? smartClient : fastClient;
        return client.prompt().user(message).call().content();
    }

    private boolean isComplex(String message) {
        // Heuristic: length, keywords, question type
        return message.length() > 200 
            || message.contains("analyze")
            || message.contains("compare");
    }
}
```

Routing — heuristic (length + keywords) → model select. [2402][2411]

### Routing Impact

```text
40-60% requests = Simple (cheap model)
= 40-60% cost save
```

Routing impact — 40-60% simple → cheap. [2402][2411]

---

## 5. Prompt Caching (Provider-Native)

```text
Prompt caching = Stable prefix cache
= Cache read = 10-20% price
```

### Cacheable Prefix

```text
Cache:
1. System prompt (stable)
2. Tool definitions (stable)
3. Static reference content

Don't cache:
- Retrieved docs (per-query)
- Conversation history (accumulates)
```

Cacheable — system + tools + static; non-cacheable — docs + history. [2408]

### Cache Constraint

```text
Prefix = Byte-for-byte identical
= Any change = cache invalid
```

Constraint — byte-for-byte identical prefix. [2408]

### Cache Hit Rate Metric

```text
Cache hit rate = cached_tokens / total_input_tokens
= High = good (cost save)
```

Cache hit rate — cached / total input. [2405][2408]

---

## 6. Full Working Example

### CostTrackingAdvisor

```java
package com.example.cost.advisor;

import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.ai.chat.client.ChatClientRequest;
import org.springframework.ai.chat.client.ChatClientResponse;
import org.springframework.ai.chat.client.advisor.api.CallAdvisor;
import org.springframework.ai.chat.client.advisor.api.CallAdvisorChain;
import org.springframework.ai.chat.metadata.Usage;

public class CostTrackingAdvisor implements CallAdvisor {

    private final MeterRegistry registry;

    public CostTrackingAdvisor(MeterRegistry registry) {
        this.registry = registry;
    }

    @Override
    public ChatClientResponse adviseCall(ChatClientRequest request, CallAdvisorChain chain) {
        ChatClientResponse response = chain.nextCall(request);
        Usage usage = response.response().getMetadata().getUsage();

        registry.counter("ai.tokens.total",
            "model", response.response().getMetadata().getModel())
            .increment(usage.getTotalTokens());

        registry.counter("ai.tokens.prompt",
            "model", response.response().getMetadata().getModel())
            .increment(usage.getPromptTokens());

        registry.counter("ai.tokens.completion",
            "model", response.response().getMetadata().getModel())
            .increment(usage.getCompletionTokens());

        return response;
    }

    @Override
    public String getName() {
        return "CostTrackingAdvisor";
    }

    @Override
    public int getOrder() {
        return 300;
    }
}
```

### CostConfig

```java
package com.example.cost.config;

import com.example.cost.advisor.CostTrackingAdvisor;
import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.model.ChatModel;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class CostConfig {

    @Bean
    public ChatClient chatClient(ChatModel chatModel, MeterRegistry registry) {
        return ChatClient.builder(chatModel)
            .defaultAdvisors(new CostTrackingAdvisor(registry))
            .build();
    }
}
```

### CostController

```java
package com.example.cost.controller;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/chat")
public class CostController {

    private final ChatClient chatClient;

    public CostController(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    public record ChatRequest(String message) {}
    public record ChatResponse(String reply) {}

    @PostMapping
    public ChatResponse chat(@RequestBody ChatRequest req) {
        String reply = chatClient.prompt()
            .user(req.message())
            .call()
            .content();
        return new ChatResponse(reply);
    }
}
```

---

## 7. Run + Test

```bash
export OPENAI_API_KEY=sk-...
mvn spring-boot:run

# Chat (cost tracked)
curl -X POST http://localhost:8080/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"What is Java?"}'

# Metrics (token usage)
curl http://localhost:8080/actuator/prometheus | grep ai_tokens
```

---

## 8. Common Gotchas

### Gotcha 1 — No Budget Cap

```text
Problem:
Cost blow

Fix:
Per-turn + per-conversation + per-day
```

### Gotcha 2 — Cache Threshold Too Low

```text
Problem:
Wrong cached answer

Fix:
Threshold 0.92+
```

### Gotcha 3 — Cache Volatile Content

```text
Problem:
Docs / history cache = invalid

Fix:
Stable prefix only (system + tools)
```

### Gotcha 4 — No Cost Metrics

```text
Problem:
Blind spend

Fix:
CostTrackingAdvisor + Grafana
```

---

## 9. Interview Questions

| Question | Answer |
|---|---|
| Cost management kya? | Token budget + caching + routing. |
| Token budget levels? | Per-turn, per-conversation, per-day. |
| Semantic cache? | Vector similarity (query → response). |
| Model routing? | Simple → cheap, complex → smart. |
| Prompt caching? | Stable prefix (cache read = 10-20%). |
| Usage interface? | promptTokens + completionTokens + cache metrics. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Cost blow | Budget caps |
| Repeated queries | Semantic cache |
| Cost optimize | Model routing |
| Long conversations | Prompt caching |
| Cost tracking | Advisor + metrics |

---

## 10. Key Takeaways

```text
1. Cost = token-based (input + output).

2. Budget = per-turn + per-conversation + per-day.

3. Semantic cache = vector similarity (0.92).

4. Model routing = simple → cheap.

5. Prompt caching = stable prefix.

6. Usage = prompt + completion + cache.

7. Cache read = 10-20% price.

8. Advisor = cost tracking.

9. Combined = 60-80% reduction.

10. Metrics = cost visibility.
```

---

## 11. Next Part Preview

**Part 21: Resilience (Circuit Breaker + Fallback + Retry)**

Isme hum cover karenge:

```text
- Resilience deep dive
- Circuit breaker
- Fallback chain
- Retry (transient errors)
- Rate limit handling
```