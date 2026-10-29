# Part 19: Spring AI — Observability (Micrometer + OpenTelemetry)

> Observability ka core — **3 pillars (logs + metrics + traces) + Micrometer Observation (1 instrumentation = timer + span dono) + Spring AI built-in observations (`ChatClient`, `ChatModel`, `EmbeddingModel`, `VectorStore`, `Advisor`, `ToolCallingManager`) + token usage (`gen_ai_client_token_usage_total`) + OTLP export** — low-cardinality tags (model, provider, operation) + high-cardinality (conversation ID, trace ID). [2385][2388][2389][2394]

---

## 1. Observability Mental Model

```text
3 Pillars:
1. Logs     = "Kya hua" (events)
2. Metrics  = "Kitna hua" (aggregates)
3. Traces   = "Kahan hua" (journey)

Spring AI:
Micrometer Observation = 1 instrumentation = timer + span
= Metrics + traces dono (single API)
```

### Observability Flow

```text
App -> Micrometer Observation
   |
   +--> Metrics (Prometheus / OTLP)
   +--> Traces (Zipkin / Tempo / OTLP)
   +--> Logs (Loki / OTLP)
```

Observability — logs + metrics + traces (3 pillars). [2385][2388]

### Micrometer Observation

```text
Observation = 1 unit of work
= Timer (metrics) + Span (traces)
= Single API, dual output
```

Micrometer Observation — 1 instrumentation = timer + span. [2385][2393]

---

## 2. Spring AI Built-in Observations

```text
Spring AI = Auto-instrumented
= ChatClient, ChatModel, EmbeddingModel, VectorStore, Advisor, ToolCallingManager
```

### Observation Points

| Component | Observation | Metrics |
|---|---|---|
| ChatClient | `spring.ai.chat.client` | `spring_ai_chat_client_operation_seconds` |
| ChatModel | `gen_ai.client` | `gen_ai_client_operation_seconds` |
| EmbeddingModel | `gen_ai.embeddings` | Token usage |
| VectorStore | `db.vector.client` | `db_vector_client_operation_seconds` |
| Advisor | Advisor-specific | — |
| ToolCallingManager | `spring.ai.tool` | Tool call metrics |

Spring AI observations — ChatClient, ChatModel, EmbeddingModel, VectorStore, Advisor, ToolCallingManager. [2386][2389][2394][2396]

### Key Metrics

| Metric | Type | Description |
|---|---|---|
| `spring_ai_chat_client_operation_seconds` | Timer | ChatClient op time |
| `gen_ai_client_operation_seconds` | Timer | AI generation time |
| `gen_ai_client_token_usage_total` | Counter | Total tokens |
| `db_vector_client_operation_seconds` | Timer | Vector store op time |

Key metrics — chat client, gen AI, token usage, vector store. [2389][2394]

### Token Usage (Cost Tracking)

```text
gen_ai_client_token_usage_total
= Prompt tokens + Completion tokens
= Cost calculation base
```

Token usage — cost tracking base. [2389][2394]

---

## 3. Tags (Low vs High Cardinality)

```text
Low-cardinality = Few values (model, provider, operation)
High-cardinality = Many values (conversation ID, trace ID)
```

### Tag Design

| Tag | Cardinality | Example |
|---|---|---|
| `gen_ai_system` | Low | openai, anthropic |
| `gen_ai_request_model` | Low | gpt-4o-mini |
| `gen_ai_operation` | Low | chat, embedding |
| `conversation_id` | High | UUID |
| `trace_id` | High | Trace ID |

Tags — low (model, provider, op) + high (conversation, trace). [2389][2395]

### Cardinality Warning

```text
High-cardinality tags = Metric explosion
= Prometheus memory blow

Rule:
Metrics = Low-cardinality only
Traces = High-cardinality OK
```

Cardinality — metrics low-only, traces high OK. [2389][2395]

---

## 4. Setup (Dependencies)

```xml
<!-- Actuator (metrics + prometheus endpoint) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<!-- Micrometer Prometheus registry -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>

<!-- Micrometer Tracing (OTel bridge) -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>

<!-- OTLP trace export -->
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

Dependencies — actuator + prometheus registry + OTel bridge + OTLP exporter. [2385][2388][2394]

### application.yml

```yaml
spring:
  application:
    name: spring-ai-observability

management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  tracing:
    sampling:
      probability: 1.0  # Dev (100%), prod = 0.1
  otlp:
    tracing:
      endpoint: http://localhost:4318/v1/traces
  observations:
    annotations:
      enabled: true

spring.ai.chat.client.observations.log-prompt: false  # PII protect
```

Config — sampling (dev 100%, prod 10%), OTLP endpoint, log-prompt false (PII). [2385][2388][2394]

### Docker Compose (Full Stack)

```yaml
services:
  app:
    build: .
    ports: ["8080:8080"]
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=http://otel-collector:4318/v1/traces

  prometheus:
    image: prom/prometheus
    ports: ["9090:9090"]
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    ports: ["3000:3000"]

  zipkin:
    image: openzipkin/zipkin
    ports: ["9411:9411"]

  otel-collector:
    image: otel/opentelemetry-collector
    ports: ["4317:4317", "4318:4318"]
```

Docker compose — app + prometheus + grafana + zipkin + otel-collector. [2384][2387][2394]

### prometheus.yml

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'spring-ai-app'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['app:8080']
```

Prometheus — scrape `/actuator/prometheus` (15s). [2392][2394][2398]

---

## 5. Custom Observations (@Observed)

```text
Custom = Business-specific (endpoint, tenant)
```

### @Observed Annotation

```java
@Service
public class ChatService {

    @Observed(name = "chat.request", contextualName = "chat-request")
    public String chat(String message, String tenantId) {
        return chatClient.prompt()
            .user(message)
            .call()
            .content();
    }
}
```

`@Observed` — custom observation (timer + span). [2393]

### ObservationRegistry (Programmatic)

```java
public String chat(String message, String tenantId) {
    return Observation.createNotStarted("chat.request", observationRegistry)
        .lowCardinalityKeyValue("tenant", tenantId)
        .highCardinalityKeyValue("conversation_id", conversationId)
        .observe(() -> 
            chatClient.prompt().user(message).call().content());
}
```

`Observation.createNotStarted()` — programmatic (low + high cardinality). [2385][2393]

### Custom Convention (Tags)

```java
public class ChatObservationConvention 
    implements ChatClientObservationConvention {

    @Override
    public KeyValues getLowCardinalityKeyValues(ChatClientObservationContext context) {
        return KeyValues.of(
            "tenant", context.getTenantId(),
            "model", context.getRequest().getOptions().getModel());
    }
}
```

Custom convention — low-cardinality tags (tenant, model). [2389][2391]

---

## 6. Grafana Dashboard

```text
Dashboard = Metrics visualize
```

### Dashboard Panels

```text
1. Token usage (rate)     -> gen_ai_client_token_usage_total
2. Latency (p95)          -> gen_ai_client_operation_seconds
3. Error rate             -> gen_ai_client_operation_errors
4. Vector store latency   -> db_vector_client_operation_seconds
5. Active operations      -> gen_ai_client_operation_active_seconds
```

Dashboard — token usage, latency, error rate, vector store, active. [2394]

### PromQL Queries

```promql
# Token usage rate (per minute)
sum(rate(gen_ai_client_token_usage_total[5m])) * 60

# Latency p95
histogram_quantile(0.95, rate(gen_ai_client_operation_seconds_bucket[5m]))

# Error rate
sum(rate(gen_ai_client_operation_seconds_count{error="true"}[5m]))

# Vector store latency
rate(db_vector_client_operation_seconds_sum[5m]) / rate(db_vector_client_operation_seconds_count[5m])
```

PromQL — token rate, latency p95, error rate, vector latency. [2392][2394]

---

## 7. Full Working Example

### ChatService (Observed)

```java
package com.example.observability.service;

import io.micrometer.observation.Observation;
import io.micrometer.observation.ObservationRegistry;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.stereotype.Service;

@Service
public class ChatService {

    private final ChatClient chatClient;
    private final ObservationRegistry observationRegistry;

    public ChatService(ChatClient chatClient, ObservationRegistry observationRegistry) {
        this.chatClient = chatClient;
        this.observationRegistry = observationRegistry;
    }

    public String chat(String message, String tenantId, String conversationId) {
        return Observation.createNotStarted("chat.request", observationRegistry)
            .lowCardinalityKeyValue("tenant", tenantId)
            .highCardinalityKeyValue("conversation_id", conversationId)
            .observe(() -> 
                chatClient.prompt()
                    .user(message)
                    .call()
                    .content());
    }
}
```

### ChatController

```java
package com.example.observability.controller;

import com.example.observability.service.ChatService;
import org.springframework.web.bind.annotation.*;

import java.util.UUID;

@RestController
@RequestMapping("/api/chat")
public class ChatController {

    private final ChatService chatService;

    public ChatController(ChatService chatService) {
        this.chatService = chatService;
    }

    public record ChatRequest(String message) {}
    public record ChatResponse(String reply) {}

    @PostMapping
    public ChatResponse chat(
        @RequestBody ChatRequest req,
        @RequestHeader(value = "X-Tenant-Id", defaultValue = "default") String tenantId) {
        String conversationId = UUID.randomUUID().toString();
        String reply = chatService.chat(req.message(), tenantId, conversationId);
        return new ChatResponse(reply);
    }
}
```

---

## 8. Run + Test

```bash
# 1. Start full stack
docker-compose up -d

# 2. API key
export OPENAI_API_KEY=sk-...

# 3. Run app
mvn spring-boot:run

# 4. Test
curl -X POST http://localhost:8080/api/chat \
  -H "Content-Type: application/json" \
  -H "X-Tenant-Id: acme" \
  -d '{"message":"What is Java?"}'

# 5. View
# Prometheus: http://localhost:9090
# Grafana:    http://localhost:3000 (admin/admin)
# Zipkin:     http://localhost:9411
```

---

## 9. Common Gotchas

### Gotcha 1 — High-Cardinality in Metrics

```text
Problem:
Metric explosion

Fix:
Low-cardinality tags only (metrics)
```

### Gotcha 2 — Prompt Logging

```text
Problem:
PII leak

Fix:
log-prompt: false
```

### Gotcha 3 — 100% Sampling in Prod

```text
Problem:
Trace volume blow

Fix:
Sampling 0.1 (10%)
```

### Gotcha 4 — No Trace Correlation

```text
Problem:
Logs / metrics / traces disconnected

Fix:
Trace ID propagate (MDC + headers)
```

---

## 10. Interview Questions

| Question | Answer |
|---|---|
| Observability 3 pillars? | Logs, metrics, traces. |
| Micrometer Observation? | 1 instrumentation = timer + span. |
| Spring AI observations? | ChatClient, ChatModel, EmbeddingModel, VectorStore, Advisor, Tools. |
| Token usage metric? | gen_ai_client_token_usage_total. |
| Cardinality? | Low (metrics) vs high (traces). |
| OTLP? | OpenTelemetry Protocol (vendor-neutral). |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Cost tracking | Token usage metric |
| Latency | Timer (p95) |
| Trace | OTel + Zipkin/Tempo |
| Cardinality | Low tags (metrics) |
| PII | log-prompt false |

---

## 11. Key Takeaways

```text
1. Observability = logs + metrics + traces.

2. Micrometer Observation = 1 → timer + span.

3. Spring AI = auto-instrumented (6 points).

4. Token usage = cost tracking.

5. Cardinality = low (metrics) / high (traces).

6. OTLP = vendor-neutral export.

7. Sampling = dev 100%, prod 10%.

8. log-prompt = false (PII).

9. Grafana = dashboards.

10. Trace ID = correlation.
```

---

## 12. Next Part Preview

**Part 20: Cost Management (Token Budget + Caching + Model Routing)**

Isme hum cover karenge:

```text
- Cost management deep dive
- Token budget
- Caching (semantic cache)
- Model routing
- Cost optimization
```