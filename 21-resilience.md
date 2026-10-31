# Part 21: Spring AI — Resilience (Circuit Breaker + Fallback + Retry)

> Resilience ka core — **Spring AI built-in retry (`spring.ai.retry.*`, `TransientAiException` vs `NonTransientAiException`) + Resilience4j (CircuitBreaker + RateLimiter + Retry) + fallback chain (primary → secondary → degraded) + exponential backoff with jitter** — 429/503 = retryable, 400/401 = non-retryable, order = `Retry(RateLimiter(call))`. [2417][2419][2420][2424]

---

## 1. Resilience Mental Model

```text
Problem:
LLM APIs fail differently
= 429 (rate limit)
= 503 (overloaded)
= 401 (invalid key)
= Timeout
= Context overflow

Solution:
Layered resilience:
1. Retry (transient)
2. Circuit breaker (cascade prevent)
3. Rate limiter (quota respect)
4. Fallback (graceful degradation)
```

### Error Classification

| Error | HTTP | Retry? | Why |
|---|---|---|---|
| Rate limit | 429 | Yes (backoff) | Transient |
| Overloaded | 503 | Yes (backoff) | Transient |
| Server error | 5xx | Yes (bounded) | Transient |
| Invalid key | 401 | No | Permanent |
| Bad request | 400 | No | Permanent |
| Not found | 404 | No | Permanent |
| Insufficient quota | 429/402 | No | Billing issue |

Error classification — 429/503/5xx = retryable, 400/401/404 = non-retryable. [2419][2420][2424]

### Spring AI Exceptions

```java
TransientAiException     = Retryable (429, 5xx, timeout)
NonTransientAiException  = Permanent (401, 400, 404)
```

Spring AI — `TransientAiException` (retryable) vs `NonTransientAiException` (permanent). [2417][2424]

---

## 2. Spring AI Built-in Retry

```text
Built-in = spring.ai.retry.* properties
= Auto-configured (starter on classpath)
```

### Built-in Retry Config

```yaml
spring:
  ai:
    retry:
      max-attempts: 3              # Default: 10 (tune down)
      on-client-errors: false      # Retry 4xx? (default false)
      on-http-codes: 429,503       # Retry these
      exclude-on-http-codes: 401,403
      backoff:
        initial-interval: 2s       # First delay
        multiplier: 5              # Exponential growth
        max-interval: 3m           # Max delay
```

Built-in retry — `max-attempts` (default 10, tune to 3), `on-http-codes` (429, 503), backoff (2s → 10s → 50s). [2424][2426]

### Built-in Retry Defaults

| Property | Default | Recommended |
|---|---|---|
| max-attempts | 10 | 3 |
| initial-interval | 2s | 2s |
| multiplier | 5 | 5 |
| max-interval | 3m | 3m |
| on-client-errors | false | false |

Defaults — max-attempts 10 (tune 3), initial 2s, multiplier 5. [2424][2426]

### Priority Order

```text
1. onHttpCodes (highest) → Retry
2. excludeOnHttpCodes → No retry
3. onClientErrors → Retry 4xx if true
4. Default → Retry 5xx, no retry 4xx
```

Priority — onHttpCodes → exclude → onClientErrors → default. [2426]

---

## 3. Resilience4j (Circuit Breaker + Rate Limiter)

```text
Resilience4j = Circuit breaker + rate limiter + bulkhead
= Beyond built-in retry
```

### Dependency

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
    <version>2.2.0</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

Resilience4j — circuit breaker + rate limiter + bulkhead + retry. [2417][2419]

### Circuit Breaker Config

```yaml
resilience4j:
  circuitbreaker:
    instances:
      ai-client:
        failure-rate-threshold: 50        # Open at 50% failure
        slow-call-rate-threshold: 80      # Open at 80% slow
        slow-call-duration-threshold: 10s # Slow = >10s
        sliding-window-size: 10           # Last 10 calls
        minimum-number-of-calls: 5        # Min before evaluate
        wait-duration-in-open-state: 30s  # Open → half-open
        permitted-number-of-calls-in-half-open-state: 3
```

Circuit breaker — failure rate 50%, slow call 80%, window 10, wait 30s. [2417][2419]

### Circuit Breaker States

```text
CLOSED -> Normal (calls pass)
OPEN   -> Fail fast (no calls)
HALF-OPEN -> Test (3 calls)

CLOSED --(50% fail)--> OPEN --(30s)--> HALF-OPEN --(success)--> CLOSED
```

States — CLOSED → OPEN → HALF-OPEN → CLOSED. [2417][2419]

### Rate Limiter Config

```yaml
resilience4j:
  ratelimiter:
    instances:
      llm:
        limit-for-period: 200        # 200 calls
        limit-refresh-period: 100ms  # Per 100ms
        timeout-duration: 60s        # Wait max
```

Rate limiter — 200 calls / 100ms, timeout 60s. [2423]

### Order (Critical)

```text
Retry(RateLimiter(call))
= Har attempt limiter se permit le
= Retry burst quota exceed nahi kare

WRONG: RateLimiter(Retry(call))
= Retry burst = quota exceed
```

Order — `Retry(RateLimiter(call))` (har attempt permit). [2423]

---

## 4. Fallback Chain

```text
Fallback = Graceful degradation
= Primary fail → Secondary → Degraded
```

### Fallback Chain

```text
1. Primary (GPT-4o) fail
2. Secondary (Claude) fail
3. Degraded (cached / static message)
```

Fallback chain — primary → secondary → degraded. [2418][2420][2428]

### Fallback Implementation

```java
@Service
public class ResilientChatService {

    private final ChatClient primary;   // GPT-4o
    private final ChatClient secondary; // Claude

    public ResilientChatService(
        @Qualifier("primaryClient") ChatClient primary,
        @Qualifier("secondaryClient") ChatClient secondary) {
        this.primary = primary;
        this.secondary = secondary;
    }

    @CircuitBreaker(name = "ai-client", fallbackMethod = "fallbackToSecondary")
    @Retry(name = "ai-client")
    public String chat(String message) {
        return primary.prompt().user(message).call().content();
    }

    public String fallbackToSecondary(String message, Throwable t) {
        try {
            return secondary.prompt().user(message).call().content();
        } catch (Exception e) {
            return degradedResponse(message);
        }
    }

    private String degradedResponse(String message) {
        return "AI service temporarily unavailable. Please try again shortly.";
    }
}
```

Fallback — `@CircuitBreaker(fallbackMethod)` + `@Retry` + degraded. [2417][2418][2420]

### Degraded Flag (UI)

```java
public record SummaryResult(String text, boolean degraded) {}

// UI: degraded = true -> "Availability notice" show
```

Degraded flag — UI availability notice (fallback distinguish). [2418]

---

## 5. Exponential Backoff + Jitter

```text
Backoff = 1s → 2s → 4s → 8s
Jitter = Random (±20%)
= Stampede prevent
```

### Jitter Why

```text
Without jitter:
All requests 429 at same time
= Retry at same time
= Stampede (re-trigger 429)

With jitter:
Random (±20%)
= Spread (no stampede)
```

Jitter — stampede prevent (random spread). [2420][2428]

### Jitter Implementation

```java
long baseDelay = 2000;  // 2s
long jitter = (long) (baseDelay * 0.2 * (Math.random() * 2 - 1));
long delay = baseDelay + jitter;
Thread.sleep(delay);
```

Jitter — base + random (±20%). [2420]

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
    <artifactId>spring-ai-starter-model-ollama</artifactId>
</dependency>
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
    <version>2.2.0</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

### application.yml

```yaml
spring:
  ai:
    retry:
      max-attempts: 3
      on-http-codes: 429,503
      exclude-on-http-codes: 401,403
      backoff:
        initial-interval: 2s
        multiplier: 5
        max-interval: 3m
    openai:
      api-key: ${OPENAI_API_KEY}
    ollama:
      base-url: http://localhost:11434

resilience4j:
  circuitbreaker:
    instances:
      ai-client:
        failure-rate-threshold: 50
        slow-call-rate-threshold: 80
        slow-call-duration-threshold: 10s
        sliding-window-size: 10
        minimum-number-of-calls: 5
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 3
  ratelimiter:
    instances:
      llm:
        limit-for-period: 200
        limit-refresh-period: 100ms
        timeout-duration: 60s
```

### ResilientChatService

```java
package com.example.resilience.service;

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.ratelimiter.annotation.RateLimiter;
import io.github.resilience4j.retry.annotation.Retry;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;

@Service
public class ResilientChatService {

    private final ChatClient primary;   // OpenAI
    private final ChatClient secondary; // Ollama

    public ResilientChatService(
        @Qualifier("openAiClient") ChatClient primary,
        @Qualifier("ollamaClient") ChatClient secondary) {
        this.primary = primary;
        this.secondary = secondary;
    }

    @CircuitBreaker(name = "ai-client", fallbackMethod = "fallbackToSecondary")
    @Retry(name = "ai-client")
    @RateLimiter(name = "llm")
    public String chat(String message) {
        return primary.prompt().user(message).call().content();
    }

    public String fallbackToSecondary(String message, Throwable t) {
        try {
            return secondary.prompt().user(message).call().content();
        } catch (Exception e) {
            return degradedResponse();
        }
    }

    private String degradedResponse() {
        return "AI service temporarily unavailable. Please try again shortly.";
    }
}
```

### ResilienceConfig

```java
package com.example.resilience.config;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.openai.OpenAiChatModel;
import org.springframework.ai.ollama.OllamaChatModel;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

@Configuration
public class ResilienceConfig {

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

### ChatController

```java
package com.example.resilience.controller;

import com.example.resilience.service.ResilientChatService;
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

## 7. Run + Test

```bash
# 1. Start Ollama (fallback)
ollama serve

# 2. API key
export OPENAI_API_KEY=sk-...

# 3. Run
mvn spring-boot:run

# 4. Test (primary → fallback → degraded)
curl -X POST http://localhost:8080/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"What is Java?"}'
```

---

## 8. Common Gotchas

### Gotcha 1 — Double Retry

```text
Problem:
Outer retry + built-in retry = double

Fix:
Built-in retry configure (outer nahi)
```

### Gotcha 2 — Retry 400

```text
Problem:
Blind retry = money burn

Fix:
on-client-errors: false
```

### Gotcha 3 — No Jitter

```text
Problem:
Stampede (429 re-trigger)

Fix:
Jitter (±20%)
```

### Gotcha 4 — Wrong Order

```text
Problem:
Retry burst = quota exceed

Fix:
Retry(RateLimiter(call))
```

---

## 9. Interview Questions

| Question | Answer |
|---|---|
| Resilience kya? | Retry + circuit breaker + rate limiter + fallback. |
| Transient vs NonTransient? | Retryable vs permanent. |
| Circuit breaker states? | CLOSED → OPEN → HALF-OPEN. |
| Fallback chain? | Primary → secondary → degraded. |
| Jitter kya? | Random spread (stampede prevent). |
| Order? | Retry(RateLimiter(call)). |

### Scenario-Based

| Scenario | Solution |
|---|---|
| 429 / 503 | Retry + backoff |
| Cascade prevent | Circuit breaker |
| Quota respect | Rate limiter |
| Graceful degradation | Fallback chain |
| Stampede | Jitter |

---

## 10. Key Takeaways

```text
1. Resilience = retry + CB + rate limiter + fallback.

2. Transient = retryable (429, 5xx).

3. NonTransient = permanent (401, 400).

4. Circuit breaker = 50% fail → OPEN.

5. Rate limiter = quota respect.

6. Fallback = primary → secondary → degraded.

7. Jitter = stampede prevent.

8. Order = Retry(RateLimiter(call)).

9. Built-in retry = spring.ai.retry.*.

10. Resilience4j = CB + rate limiter + bulkhead.
```

---

## 11. Next Part Preview

**Part 22: Security (PII Scrubbing + Prompt Injection Guard + Output Validation)**

Isme hum cover karenge:

```text
- Security deep dive
- PII scrubbing (advisor)
- Prompt injection guard
- Output validation
- OWASP LLM Top 10
```