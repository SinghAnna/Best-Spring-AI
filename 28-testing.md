# Part 28: Spring AI — Testing (Unit + Integration + Mock LLM)

> Testing ka core — **3 levels (unit fake ChatModel + integration Testcontainers Ollama + evaluation LLM-as-judge) + `ChatModel` interface (lambda / Mockito mock) + `ChatClient.create(fakeModel)` (no Spring) + `@MockBean ChatModel` + `ArgumentCaptor<Prompt>` (prompt verify) + Testcontainers (`@ServiceConnection OllamaContainer`) + semantic assertion (keywords, not exact) + CI strategy (unit every build, integration every PR, evaluation nightly)** — fake model = deterministic + fast (<1s), real model = integration (Docker), evaluator = quality (nightly). [2503][2508][2511]

---

## 1. Testing Mental Model

```text
Problem:
LLM = Non-deterministic
= Same prompt → different answer
= External API (slow + cost + flaky)

Solution:
3-Level Testing:
1. Unit (fake ChatModel) — fast, deterministic
2. Integration (Testcontainers Ollama) — real, offline
3. Evaluation (LLM-as-judge) — quality
```

### 3-Level Testing

| Level | Model | Speed | Cost | When |
|---|---|---|---|---|
| Unit | Fake (lambda / mock) | <1s | Free | Every build |
| Integration | Real (Ollama container) | ~16s | Free (local) | Every PR |
| Evaluation | Real + judge | Minutes | API cost | Nightly |

3-level testing — fake + real + judge (speed + cost + when). [2508][2511]

### Testing Pyramid

```text
        /  Evaluation  \      (Few, nightly)
       / Integration    \     (Some, every PR)
      /   Unit Tests     \    (Many, every build)
```

Testing pyramid — unit (many) + integration (some) + evaluation (few). [2511]

---

## 2. Unit Testing (Fake ChatModel)

```text
Fake ChatModel = Lambda / Mockito
= Deterministic + fast
```

### Fake ChatModel (Lambda)

```java
ChatModel fakeModel = prompt -> new ChatResponse(
    List.of(new Generation(new AssistantMessage("POSITIVE"))));

ChatClient fakeClient = ChatClient.create(fakeModel);
```

Fake ChatModel — lambda (fixed answer, no Spring). [2511]

### Mockito Mock

```java
ChatModel mockModel = mock(ChatModel.class);
when(mockModel.call(any(Prompt.class)))
    .thenReturn(new ChatResponse(
        List.of(new Generation(new AssistantMessage("POSITIVE")))));
```

Mockito — `mock(ChatModel.class)` + `when().thenReturn()`. [2508][2511]

### @MockBean (Spring Context)

```java
@SpringBootTest
class SentimentServiceTests {

    @MockBean
    private ChatModel chatModel;

    @Autowired
    private SentimentService sentimentService;

    @Test
    void testSentiment() {
        given(chatModel.call(any(Prompt.class)))
            .willReturn(new ChatResponse(
                List.of(new Generation(new AssistantMessage("POSITIVE")))));

        String result = sentimentService.analyze("I love Spring Boot!");
        assertThat(result).isEqualTo("POSITIVE");
    }
}
```

`@MockBean` — Spring context + mock bean (inject). [2508]

### Prompt Verification (ArgumentCaptor)

```java
ArgumentCaptor<Prompt> captor = ArgumentCaptor.forClass(Prompt.class);
verify(chatModel).call(captor.capture());

String prompt = captor.getValue().getContents();
assertThat(prompt).contains("Classify sentiment");
assertThat(prompt).contains("I love Spring Boot!");
```

`ArgumentCaptor<Prompt>` — prompt verify (system + user). [2508]

---

## 3. Integration Testing (Testcontainers)

```text
Testcontainers = Real model in Docker
= Ollama container (offline, zero-cost)
```

### Testcontainers + Ollama

```java
@Testcontainers
@SpringBootTest
class IntegrationTests {

    @Container
    @ServiceConnection
    static OllamaContainer ollama = 
        new OllamaContainer("ollama/ollama:latest")
            .withPullModel("gemma:2b");  // Small model (speed)

    @Autowired
    private ChatClient chatClient;

    @Test
    void testRealInference() {
        String result = chatClient.prompt()
            .user("Is this positive or negative? 'This pizza is terrible'")
            .call()
            .content();

        assertThat(result.toUpperCase())
            .containsAnyOf("NEGATIVE", "BAD", "POOR");
    }
}
```

Testcontainers — `@Container` + `@ServiceConnection` + Ollama (small model). [2503][2508][2511]

### Semantic Assertion

```text
Semantic = Keywords (not exact match)
= LLM non-deterministic
= Exact match fail
```

Semantic assertion — keywords (not exact). [2508][2511]

### Testcontainers + Vector Store (RAG)

```java
@Container
@ServiceConnection
static PostgreSQLContainer<?> postgres = 
    new PostgreSQLContainer<>("pgvector/pgvector:pg16");

// Vector store + embeddings + retrieval test
```

Testcontainers — Postgres + pgvector (RAG integration). [2503][2508]

---

## 4. Evaluation Testing (LLM-as-Judge)

```text
Evaluation = LLM-as-judge (quality)
= RelevancyEvaluator + FactCheckingEvaluator
```

### Evaluation Test

```java
@SpringBootTest
class EvaluationTests {

    @Autowired
    private ChatClient chatClient;

    @Test
    void testRelevance() {
        String question = "What is Spring AI?";
        String answer = chatClient.prompt().user(question).call().content();

        RelevancyEvaluator evaluator = 
            new RelevancyEvaluator(ChatClient.builder(chatModel));

        EvaluationRequest request = new EvaluationRequest(
            question, List.of(), answer);

        assertThat(evaluator.evaluate(request).isPass()).isTrue();
    }
}
```

Evaluation — `RelevancyEvaluator` (LLM-as-judge). [2508][2511]

### Evaluation CI (Nightly)

```text
Evaluation = Nightly (not every build)
= API cost + slow
= Separate job (CI)
```

Evaluation CI — nightly (cost + slow). [2511]

---

## 5. Tool Calling Testing

```text
Tool calling test = Mock tool + verify call
= Record / replay (VCR)
```

### Tool Mock (Record / Replay)

```text
VCR = Record real exchange → Replay deterministic
= ToolCallingManager intercept (above network)
```

VCR — record / replay (ToolCallingManager intercept). [2510]

### Tool Test (Execute Real)

```java
@Test
@VcrTool(mode = VcrToolMode.EXECUTE_REAL)
void testRealToolCall() {
    // Real tool runs (assert invocation)
    String result = chatClient.prompt()
        .user("What is the weather?")
        .call()
        .content();

    verify(weatherService).getWeather(any());
}
```

Tool test — `EXECUTE_REAL` (real tool, assert invocation). [2510]

---

## 6. Full Working Example

### pom.xml (Dependencies)

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
```

### SentimentService

```java
package com.example.testing.service;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.stereotype.Service;

@Service
public class SentimentService {

    private final ChatClient chatClient;

    public SentimentService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    public String analyze(String text) {
        return chatClient.prompt()
            .system("Classify sentiment as POSITIVE or NEGATIVE")
            .user(text)
            .call()
            .content();
    }
}
```

### SentimentServiceUnitTest

```java
package com.example.testing;

import com.example.testing.service.SentimentService;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.ai.chat.model.ChatModel;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.chat.ChatResponse;
import org.springframework.ai.chat.messages.AssistantMessage;
import org.springframework.ai.chat.Generation;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.BDDMockito.given;
import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
class SentimentServiceUnitTest {

    @MockBean
    private ChatModel chatModel;

    @Autowired
    private SentimentService sentimentService;

    @Test
    void testPositive() {
        given(chatModel.call(any(Prompt.class)))
            .willReturn(new ChatResponse(
                List.of(new Generation(new AssistantMessage("POSITIVE")))));

        String result = sentimentService.analyze("I love Spring Boot!");
        assertThat(result).isEqualTo("POSITIVE");
    }
}
```

### SentimentServiceIntegrationTest

```java
package com.example.testing;

import com.example.testing.service.SentimentService;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.testcontainers.containers.OllamaContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import static org.assertj.core.api.Assertions.assertThat;

@Testcontainers
@SpringBootTest
class SentimentServiceIntegrationTest {

    @Container
    @ServiceConnection
    static OllamaContainer ollama = 
        new OllamaContainer("ollama/ollama:latest")
            .withPullModel("gemma:2b");

    @Autowired
    private SentimentService sentimentService;

    @Test
    void testRealInference() {
        String result = sentimentService.analyze("This pizza is terrible");
        assertThat(result.toUpperCase())
            .containsAnyOf("NEGATIVE", "BAD", "POOR");
    }
}
```

---

## 7. Run + Test

```bash
# Unit tests (fast, no Docker)
mvn test -Dtest=SentimentServiceUnitTest

# Integration tests (Docker required)
mvn test -Dtest=SentimentServiceIntegrationTest

# All tests
mvn test
```

---

## 8. CI Strategy

```text
CI = 3 jobs
1. Unit (every build, <1s)
2. Integration (every PR, ~16s)
3. Evaluation (nightly, minutes)
```

CI strategy — unit (every build) + integration (every PR) + evaluation (nightly). [2511]

### CI Config (GitHub Actions)

```yaml
jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
      - run: mvn test -Dtest='*UnitTest'

  integration:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
      - run: mvn test -Dtest='*IntegrationTest'

  evaluation:
    runs-on: ubuntu-latest
    if: github.event_name == 'schedule'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
      - run: mvn test -Dtest='*EvaluationTest'
```

CI — 3 jobs (unit + integration + evaluation). [2511]

---

## 9. Common Gotchas

### Gotcha 1 — Exact Match

```text
Problem:
LLM non-deterministic (exact match fail)

Fix:
Semantic assertion (keywords)
```

### Gotcha 2 — Real API in Unit Test

```text
Problem:
Slow + cost + flaky

Fix:
Fake ChatModel (lambda / mock)
```

### Gotcha 3 — No Docker in CI

```text
Problem:
Integration test fail

Fix:
Docker in CI (Testcontainers)
```

### Gotcha 4 — Evaluation Every Build

```text
Problem:
Cost + slow

Fix:
Nightly (separate job)
```

---

## 10. Interview Questions

| Question | Answer |
|---|---|
| Testing levels? | Unit + integration + evaluation. |
| Fake ChatModel? | Lambda / mock (deterministic). |
| @MockBean? | Spring context + mock bean. |
| ArgumentCaptor? | Prompt verify (system + user). |
| Testcontainers? | Real model in Docker (Ollama). |
| Semantic assertion? | Keywords (not exact). |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Fast test | Fake ChatModel (unit) |
| Real model | Testcontainers (Ollama) |
| Prompt verify | ArgumentCaptor<Prompt> |
| RAG test | Postgres + pgvector container |
| Quality test | LLM-as-judge (evaluation) |

---

## 11. Key Takeaways

```text
1. Testing = 3 levels (unit + integration + evaluation).

2. Fake ChatModel = lambda / mock (fast).

3. @MockBean = Spring context + mock.

4. ArgumentCaptor = prompt verify.

5. Testcontainers = real model (Ollama).

6. Semantic assertion = keywords (not exact).

7. CI = unit (build) + integration (PR) + evaluation (nightly).

8. VCR = record / replay (tool calling).

9. Pyramid = unit (many) + integration (some) + evaluation (few).

10. Trade-off = speed vs realism vs cost.
```

---

## 12. Next Part Preview

**Part 29: Production Deployment (Docker + K8s + Config)**

Isme hum cover karenge:

```text
- Production deployment deep dive
- Docker (multi-stage build)
- Kubernetes (Deployment + Service + ConfigMap)
- Config management (env vars + secrets)
- Health checks + readiness
```