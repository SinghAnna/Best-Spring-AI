# Part 24: Spring AI — Observability (Micrometer + Grafana + Distributed Tracing)

> Observability ka core — **3 signals (metrics + traces + logs) + Micrometer (`Observation` → metrics + span) + Spring AI built-in instrumentation (`spring.ai.chat.client`, `gen_ai.client.operation`, `gen_ai.client.token.usage`, `db.vector.client.operation`) + Actuator (`/actuator/prometheus`) + Prometheus + Grafana + distributed tracing (Micrometer Tracing + OTel bridge + OTLP → Tempo/Jaeger) + GenAI semantic conventions (`gen_ai.*` attributes)** — prompt/completion logging (`log-prompt=true`, `log-completion=true`), span hierarchy (HTTP → ChatClient → advisor → LLM → tool). [2152][2459][2460][2464]

---

## 1. Observability Mental Model

```text
Problem:
AI app = Black box
= LLM call slow? Why?
= Token usage? Cost?
= Vector search slow?
= Tool call fail?

Solution:
3 Signals:
1. Metrics (aggregate)
2. Traces (single request)
3. Logs (detailed)
```

### 3 Signals

| Signal | Question | Example |
|---|---|---|
| Metrics | What happened across requests? | Total tokens, calls, avg latency |
| Traces | Where did one request spend time? | Doc search → tool → LLM |
| Logs | What exactly happened? | Prompt + response content |

3 signals — metrics (aggregate), traces (single request), logs (detailed). [2464][2467][2470]

### Observability vs Monitoring

```text
Monitoring = Known failure (alert)
Observability = Unknown failure (debug)

Monitoring = CPU high → alert
Observability = Why slow? → trace
```

Observability — unknown failure debug (trace). [2459][2467]

---

## 2. Micrometer (Observation → Metrics + Span)

```text
Micrometer = Facade (metrics + tracing)
= Observation → metrics + span
```

### Observation Concept

```text
Observation = Recording of something happening
= Metrics (timer, counter)
+ Span (trace)
```

`Observation` — metrics + span (dual signal). [2459][2463]

### Micrometer Tracing

```text
Micrometer Tracing = Facade (tracer libraries)
= Brave / OpenTelemetry bridge
= Span create / start / stop / report
```

Micrometer Tracing — Brave / OTel bridge (span lifecycle). [2463]

### Span Hierarchy (AI App)

```text
POST /api/chat/stream ← HTTP servlet span
└── spring_ai chat_client ← ChatClient.stream()
    └── message_chat_memory ← Memory advisor
        └── chat claude-haiku-4-5 ← LLM call (tools)
            └── tool_call displayMerchImages ← Tool
            └── chat claude-haiku-4-5 ← LLM call (format)
```

Span hierarchy — HTTP → ChatClient → advisor → LLM → tool. [2464]

---

## 3. Spring AI Built-in Instrumentation

```text
Built-in = ChatClient + ChatModel + Embedding + VectorStore
= Auto-instrumented (Micrometer)
```

### Instrumented Components

| Component | Observation | Metrics |
|---|---|---|
| ChatClient | `spring.ai.chat.client` | Timer + counter |
| Advisor | `spring.ai.advisor` | Timer + counter |
| ChatModel | `gen_ai.client.operation` | Timer + counter |
| Token usage | `gen_ai.client.token.usage` | Counter |
| EmbeddingModel | `gen_ai.client.operation` | Timer + counter |
| VectorStore | `db.vector.client.operation` | Timer + counter |

Instrumented — ChatClient + advisor + model + token + embedding + vector. [2152][2469][2470]

### Metric Names (Prometheus)

```text
Base: gen_ai.client.operation
Prometheus: gen_ai_client_operation_seconds_sum
            gen_ai_client_operation_seconds_count
            gen_ai_client_operation_seconds_max
            gen_ai_client_operation_active_count

Dots → underscores + suffix
```

Metric names — dots → underscores + suffix (`_seconds_sum`, `_count`, `_max`, `_active_count`). [2152][2469]

### Key Metrics Table

| Metric | Type | Unit | Description |
|---|---|---|---|
| `gen_ai_chat_client_operation_seconds_sum` | Timer | seconds | Total ChatClient time |
| `gen_ai_chat_client_operation_seconds_count` | Counter | count | Completed operations |
| `gen_ai_chat_client_operation_seconds_max` | Gauge | seconds | Max duration |
| `gen_ai_chat_client_operation_active_count` | Gauge | count | In-flight calls |

Key metrics — timer + counter + gauge (active vs completed). [2152]

### Active vs Completed

```text
active_count = In-flight (currently running)
_seconds series = Completed only

Example:
active_count = 5 (5 running)
_seconds_count = 100 (100 done)
```

Active vs completed — in-flight vs done. [2152]

---

## 4. GenAI Semantic Conventions

```text
GenAI conventions = Standard attributes
= gen_ai.* (model, tokens, finish reason)
```

### GenAI Attributes

```text
gen_ai.request.model        = Requested model
gen_ai.response.model       = Actual model
gen_ai.usage.input_tokens   = Input tokens
gen_ai.usage.output_tokens  = Output tokens
gen_ai.response.finish_reasons = Stop reason
```

GenAI attributes — model + tokens + finish reason. [2464]

### Token Usage Metric

```text
gen_ai.client.token.usage
= Input + output + total tokens
= Cost tracking
```

Token usage — input + output + total (cost). [2465][2470]

---

## 5. Prompt / Completion Logging

```text
Prompt + completion logging = Detailed debug
= Opt-in (default off)
```

### Logging Config

```yaml
spring:
  ai:
    chat:
      observations:
        log-prompt: true       # Log prompt
        log-completion: true   # Log completion
    tools:
      observations:
        include-content: true  # Log tool content
```

Logging — `log-prompt` + `log-completion` + `include-content`. [2464][2468]

### Logging Risk

```text
Prompt / completion = PII / secrets
= Log carefully (mask / redact)

Best:
- Dev / staging: on
- Production: off (or mask)
```

Logging risk — PII / secrets (mask in prod). [2464][2468]

### TracingAwareLoggingObservationHandler

```text
Logging handler = Trace-aware
= Log + trace ID (correlate)
```

Trace-aware logging — log + trace ID (correlate). [2468]

---

## 6. Distributed Tracing (OTel + Grafana)

```text
Distributed tracing = End-to-end request
= Trace ID + span ID
= Cross-service propagation
```

### Distributed Tracing Flow

```text
1. Request → Trace ID + span ID
2. HTTP header (traceparent) propagate
3. Each service = Span (parent-child)
4. Export → Tempo / Jaeger
5. Grafana → Visualize
```

Distributed tracing — trace ID + span ID + propagate + export + visualize. [2458][2467]

### OTel Bridge

```text
Micrometer Tracing → OTel bridge → OTLP → Backend
= micrometer-tracing-bridge-otel
= OtlpHttpSpanExporter
```

OTel bridge — Micrometer → OTel → OTLP → backend. [2460][2463]

### OTLP Config

```yaml
management:
  opentelemetry:
    tracing:
      export:
        otlp:
          endpoint: http://localhost:4318/v1/traces
```

OTLP — endpoint (HTTP / gRPC). [2460][2464]

---

## 7. Full Working Example

### pom.xml (Dependencies)

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

### application.yml

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
    chat:
      observations:
        log-prompt: true
        log-completion: true
    tools:
      observations:
        include-content: true

management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  tracing:
    sampling:
      probability: 1.0
  opentelemetry:
    tracing:
      export:
        otlp:
          endpoint: http://localhost:4318/v1/traces
```

### ChatController

```java
package com.example.observability.controller;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/chat")
public class ChatController {

    private final ChatClient chatClient;

    public ChatController(ChatClient chatClient) {
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

### ObservabilityConfig

```java
package com.example.observability.config;

import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.model.ChatModel;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ObservabilityConfig {

    @Bean
    public ChatClient chatClient(ChatModel chatModel, MeterRegistry registry) {
        return ChatClient.builder(chatModel)
            .build();
    }
}
```

---

## 8. Run + Test

```bash
# 1. Start Prometheus + Grafana + Tempo (docker-compose)
docker-compose up -d

# 2. API key
export OPENAI_API_KEY=sk-...

# 3. Run
mvn spring-boot:run

# 4. Chat (metrics + trace)
curl -X POST http://localhost:8080/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"What is Java?"}'

# 5. Metrics (Prometheus)
curl http://localhost:8080/actuator/prometheus | grep gen_ai

# 6. Grafana
# http://localhost:3000
# Dashboard: Spring AI Observability
```

### docker-compose.yml

```yaml
version: '3.8'
services:
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"

  tempo:
    image: grafana/tempo
    ports:
      - "4318:4318"
```

### prometheus.yml

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'springai'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['host.docker.internal:8080']
```

---

## 9. Grafana Dashboard (Queries)

```text
1. Token usage:
   sum(gen_ai_client_token_usage_total)

2. Latency (p95):
   histogram_quantile(0.95, gen_ai_client_operation_seconds_bucket)

3. Call count:
   sum(gen_ai_client_operation_seconds_count)

4. Active calls:
   gen_ai_chat_client_operation_active_count

5. Vector search latency:
   histogram_quantile(0.95, db_vector_client_operation_seconds_bucket)
```

Grafana queries — token + latency + count + active + vector. [2152][2461][2470]

---

## 10. Common Gotchas

### Gotcha 1 — No Actuator

```text
Problem:
Metrics not exposed

Fix:
spring-boot-starter-actuator + /actuator/prometheus
```

### Gotcha 2 — No Tracing Bridge

```text
Problem:
Traces not exported

Fix:
micrometer-tracing-bridge-otel + OTLP
```

### Gotcha 3 — Prompt Logging in Prod

```text
Problem:
PII / secrets logged

Fix:
Prod: off (or mask)
```

### Gotcha 4 — Sampling 0

```text
Problem:
No traces

Fix:
sampling.probability: 1.0 (dev)
```

---

## 11. Interview Questions

| Question | Answer |
|---|---|
| Observability kya? | Metrics + traces + logs. |
| Micrometer? | Facade (Observation → metrics + span). |
| Spring AI instrumentation? | ChatClient + model + vector (auto). |
| GenAI conventions? | `gen_ai.*` (model, tokens, finish). |
| Distributed tracing? | Trace ID + span ID + propagate. |
| OTel bridge? | Micrometer → OTel → OTLP. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Latency debug | Trace (span hierarchy) |
| Cost tracking | Token usage metric |
| Vector slow | `db.vector.client.operation` |
| Cross-service | Distributed tracing (OTel) |
| Detailed debug | Prompt / completion logging |

---

## 12. Key Takeaways

```text
1. Observability = metrics + traces + logs.

2. Micrometer = Observation → metrics + span.

3. Spring AI = auto-instrumented (ChatClient + model + vector).

4. GenAI conventions = gen_ai.* attributes.

5. Token usage = cost tracking.

6. Actuator = /actuator/prometheus.

7. Prometheus = metrics scrape.

8. Grafana = dashboards.

9. OTel bridge = Micrometer → OTLP.

10. Span hierarchy = HTTP → ChatClient → LLM → tool.
```

---

## 13. Next Part Preview

**Part 25: Advanced RAG (HyDE + Reranking + Query Decomposition)**

Isme hum cover karenge:

```text
- Advanced RAG deep dive
- HyDE (Hypothetical Document Embeddings)
- Reranking (cross-encoder)
- Query decomposition
- Multi-query retrieval
```