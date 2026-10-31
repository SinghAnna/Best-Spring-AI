# Part 23: Spring AI — Evaluation (RAGAS + Golden Dataset + Regression Testing)

> Evaluation ka core — **`Evaluator` interface (`EvaluationRequest` → `EvaluationResponse`) + `RelevancyEvaluator` (relevance) + `FactCheckingEvaluator` (grounded factuality) + RAGAS metrics (faithfulness, answer relevancy, context precision, context recall) + golden dataset + regression testing (CI gate)** — LLM-as-judge (second model judge first), pass rate threshold (90%+), JUnit 6 + CI. [2444][2445][2450][2451]

---

## 1. Evaluation Mental Model

```text
Problem:
LLM output = Non-deterministic
= Same prompt → different answer
= Manual testing = slow + subjective

Solution:
Automated evaluation:
1. Golden dataset (Q + expected)
2. Evaluator (LLM-as-judge)
3. Regression test (CI gate)
```

### Evaluation vs Testing

```text
Testing = Deterministic (unit / integration)
Evaluation = Non-deterministic (LLM output)

Testing = Exact match
Evaluation = Quality score (relevance / factuality)
```

Evaluation vs testing — deterministic vs non-deterministic. [2450][2452]

### LLM-as-Judge

```text
Judge = Second LLM (evaluate first)
= Relevance / factuality / coherence
= Same model or different
```

LLM-as-judge — second LLM evaluate first (relevance / factuality). [2444][2451]

---

## 2. Spring AI Evaluator Interface

```text
Evaluator = Functional interface
= evaluate(EvaluationRequest) -> EvaluationResponse
```

### Evaluator Interface

```java
public interface Evaluator {
    EvaluationResponse evaluate(EvaluationRequest evaluationRequest);
}
```

`Evaluator` — `evaluate(EvaluationRequest)` → `EvaluationResponse`. [2445]

### EvaluationRequest

```java
EvaluationRequest request = new EvaluationRequest(
    question,        // User question
    documents,       // Retrieved context (RAG)
    answer           // LLM response
);
```

`EvaluationRequest` — question + documents + answer (3 constructors). [2450][2451]

### EvaluationResponse

```java
EvaluationResponse response = evaluator.evaluate(request);

boolean pass = response.isPass();       // Pass / fail
float score = response.getScore();      // 0-1 (may be 0.0)
String feedback = response.getFeedback(); // Reasoning
```

`EvaluationResponse` — pass + score + feedback + metadata. [2450][2451]

---

## 3. Built-in Evaluators

```text
Built-in = RelevancyEvaluator + FactCheckingEvaluator
= LLM-as-judge (judge prompt)
```

### RelevancyEvaluator

```java
RelevancyEvaluator evaluator = 
    new RelevancyEvaluator(ChatClient.builder(chatModel));

EvaluationRequest request = new EvaluationRequest(
    question, documents, answer);

EvaluationResponse response = evaluator.evaluate(request);
assertThat(response.isPass()).isTrue();
```

`RelevancyEvaluator` — relevance (question + context + answer). [2450][2453]

### Judge Prompt (Relevancy)

```text
Your task is to evaluate if the response for the query
is in line with the context information provided.
You have two options to answer.
Either YES or NO.
Answer YES, if the response for the query
is in line with context information otherwise NO.
```

Relevancy judge prompt — YES / NO (context alignment). [2451]

### FactCheckingEvaluator

```java
FactCheckingEvaluator evaluator = 
    new FactCheckingEvaluator(ChatClient.builder(chatModel));

EvaluationRequest request = new EvaluationRequest(
    context, Collections.emptyList(), claim);

EvaluationResponse response = evaluator.evaluate(request);
assertThat(response.isPass()).isTrue();
```

`FactCheckingEvaluator` — grounded factuality (claim vs document). [2450][2453]

### Judge Prompt (FactChecking)

```text
Evaluate whether or not the following claim is supported 
by the provided document.
Respond with "yes" if the claim is supported, 
or "no" if it is not.
```

FactChecking judge prompt — yes / no (claim supported). [2451]

### Gotcha — FactChecking Score

```text
FactCheckingEvaluator:
pass=true → score=0.0 (default, no score)
= Don't assert score, assert pass only
```

FactChecking — pass=true → score=0.0 (default). [2451]

---

## 4. RAGAS Metrics (4 Core)

```text
RAGAS = RAG evaluation framework
= 4 metrics (generation + retrieval)
```

### RAGAS Metrics Table

| Metric | Measures | Needs Ground Truth? | Stage |
|---|---|---|---|
| Faithfulness | Claims grounded in context? | No | Generation |
| Answer Relevancy | Answer addresses question? | No | Generation |
| Context Precision | Relevant chunks ranked top? | Yes (or ref-free) | Retrieval |
| Context Recall | All needed context retrieved? | Yes | Retrieval |

RAGAS — faithfulness + answer relevancy (generation), context precision + recall (retrieval). [2443][2447][2456]

### Faithfulness

```text
Faithfulness = Supported claims / Total claims
= 0-1 (higher = better)

Example:
Answer: "France is in Europe and has 10M people"
Context: "France is in Europe" (only)
= 1 claim supported / 2 total = 0.5
```

Faithfulness — supported / total claims (0-1). [2449]

### Answer Relevancy

```text
Answer Relevancy = Mean cosine similarity
= Original question vs generated questions (from answer)
= 0-1 (higher = better)

Steps:
1. LLM generate N questions from answer
2. Cosine similarity (original vs generated)
3. Mean = relevancy
```

Answer relevancy — reverse-engineer questions → cosine similarity. [2446][2456]

### Context Precision

```text
Context Precision = Relevant chunks ranked top?
= Mean of precision@k
= 0-1 (higher = better)
```

Context precision — relevant chunks ranked top (mean precision@k). [2443]

### Context Recall

```text
Context Recall = All needed context retrieved?
= 0-1 (higher = better)
= Needs ground truth
```

Context recall — all needed context retrieved (needs ground truth). [2447][2456]

---

## 5. Golden Dataset

```text
Golden dataset = Curated Q + expected answer
= Regression baseline
```

### Golden Dataset Structure

```json
[
  {
    "question": "What is Java?",
    "expectedAnswer": "Java is a programming language",
    "expectedContext": ["Java is a programming language..."]
  },
  {
    "question": "What is Spring AI?",
    "expectedAnswer": "Spring AI is a framework for AI",
    "expectedContext": ["Spring AI is a framework..."]
  }
]
```

Golden dataset — Q + expected answer + expected context. [2450][2452]

### Golden Dataset Best Practices

```text
1. 20-50 cases (start)
2. Cover happy path + edge cases
3. Include adversarial (injection)
4. Version control (git)
5. Update on new features
```

Golden dataset — 20-50 cases, happy + edge + adversarial. [2450][2452]

---

## 6. Regression Testing (CI Gate)

```text
Regression test = Golden dataset + evaluator
= CI gate (pass rate threshold)
```

### Regression Test Flow

```text
1. Load golden dataset
2. Run each case (LLM call)
3. Evaluate (Relevancy + FactChecking)
4. Pass rate = passed / total
5. Gate: pass rate >= 90%
```

Regression test — golden dataset → evaluate → pass rate → gate. [2450][2451]

### Pass Rate Threshold

```text
90%+ = Good (production)
80-90% = Acceptable (staging)
<80% = Fail (block deploy)
```

Pass rate — 90%+ production, 80-90% staging, <80% fail. [2450][2451]

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
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

### application.yml

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
```

### GoldenDatasetLoader

```java
package com.example.evaluation.dataset;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.core.io.ClassPathResource;
import org.springframework.stereotype.Component;

import java.util.List;

@Component
public class GoldenDatasetLoader {

    private final ObjectMapper mapper = new ObjectMapper();

    public record GoldenCase(
        String question, 
        String expectedAnswer, 
        List<String> expectedContext) {}

    public List<GoldenCase> load() throws Exception {
        return mapper.readValue(
            new ClassPathResource("golden-dataset.json").getInputStream(),
            mapper.getTypeFactory().constructCollectionType(List.class, GoldenCase.class));
    }
}
```

### golden-dataset.json

```json
[
  {
    "question": "What is Java?",
    "expectedAnswer": "Java is a programming language",
    "expectedContext": ["Java is a programming language"]
  },
  {
    "question": "What is Spring AI?",
    "expectedAnswer": "Spring AI is a framework for AI",
    "expectedContext": ["Spring AI is a framework for AI"]
  }
]
```

### EvaluationService

```java
package com.example.evaluation.service;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.evaluation.RelevancyEvaluator;
import org.springframework.ai.chat.evaluation.FactCheckingEvaluator;
import org.springframework.ai.evaluation.EvaluationRequest;
import org.springframework.ai.evaluation.EvaluationResponse;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class EvaluationService {

    private final ChatClient chatClient;
    private final RelevancyEvaluator relevancyEvaluator;
    private final FactCheckingEvaluator factCheckingEvaluator;

    public EvaluationService(ChatClient chatClient, ChatClient.Builder builder) {
        this.chatClient = chatClient;
        this.relevancyEvaluator = new RelevancyEvaluator(builder);
        this.factCheckingEvaluator = new FactCheckingEvaluator(builder);
    }

    public record EvaluationResult(
        boolean relevant, 
        boolean factual, 
        String answer) {}

    public EvaluationResult evaluate(String question, List<String> context) {
        String answer = chatClient.prompt()
            .user(question)
            .call()
            .content();

        EvaluationRequest relevancyRequest = new EvaluationRequest(
            question, context, answer);
        EvaluationResponse relevancyResponse = 
            relevancyEvaluator.evaluate(relevancyRequest);

        EvaluationRequest factRequest = new EvaluationRequest(
            String.join(" ", context), List.of(), answer);
        EvaluationResponse factResponse = 
            factCheckingEvaluator.evaluate(factRequest);

        return new EvaluationResult(
            relevancyResponse.isPass(),
            factResponse.isPass(),
            answer);
    }
}
```

### EvaluationTest

```java
package com.example.evaluation;

import com.example.evaluation.dataset.GoldenDatasetLoader;
import com.example.evaluation.service.EvaluationService;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
class EvaluationTest {

    @Autowired
    private EvaluationService evaluationService;

    @Autowired
    private GoldenDatasetLoader loader;

    @Test
    void goldenDatasetPassRateAbove90() throws Exception {
        List<GoldenDatasetLoader.GoldenCase> cases = loader.load();
        int passed = 0;

        for (GoldenDatasetLoader.GoldenCase c : cases) {
            EvaluationService.EvaluationResult result = 
                evaluationService.evaluate(c.question(), c.expectedContext());

            if (result.relevant() && result.factual()) {
                passed++;
            }
        }

        double passRate = (double) passed / cases.size();
        assertThat(passRate).isGreaterThanOrEqualTo(0.90);
    }
}
```

---

## 8. Run + Test

```bash
export OPENAI_API_KEY=sk-...
mvn test

# Output:
# Golden dataset pass rate: 100% (2/2)
# Tests run: 1, Failures: 0
```

---

## 9. Common Gotchas

### Gotcha 1 — Exact Match

```text
Problem:
LLM non-deterministic (exact match fail)

Fix:
Evaluator (LLM-as-judge)
```

### Gotcha 2 — Score Assert (FactChecking)

```text
Problem:
FactChecking score = 0.0 (pass=true)

Fix:
Assert pass only (not score)
```

### Gotcha 3 — Small Dataset

```text
Problem:
<20 cases = noisy

Fix:
20-50 cases (start)
```

### Gotcha 4 — No CI Gate

```text
Problem:
Regression undetected

Fix:
CI gate (pass rate >= 90%)
```

---

## 10. Interview Questions

| Question | Answer |
|---|---|
| Evaluation kya? | LLM output quality (non-deterministic). |
| Evaluator interface? | evaluate(EvaluationRequest) → EvaluationResponse. |
| RelevancyEvaluator? | Relevance (question + context + answer). |
| FactCheckingEvaluator? | Grounded factuality (claim vs document). |
| RAGAS metrics? | Faithfulness + relevancy + precision + recall. |
| Golden dataset? | Q + expected answer (regression baseline). |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Relevance check | RelevancyEvaluator |
| Hallucination check | FactCheckingEvaluator |
| RAG evaluation | RAGAS (4 metrics) |
| Regression | Golden dataset + CI gate |
| LLM-as-judge | Second model evaluate first |

---

## 11. Key Takeaways

```text
1. Evaluation = LLM output quality.

2. Evaluator = evaluate(request) → response.

3. RelevancyEvaluator = relevance.

4. FactCheckingEvaluator = factuality.

5. RAGAS = 4 metrics (gen + retrieval).

6. Faithfulness = supported / total claims.

7. Answer relevancy = cosine similarity.

8. Golden dataset = Q + expected.

9. Regression = CI gate (90%+).

10. LLM-as-judge = second model.
```

---

## 12. Next Part Preview

**Part 24: Observability (Micrometer + Grafana + Distributed Tracing)**

Isme hum cover karenge:

```text
- Observability deep dive
- Micrometer (metrics)
- Grafana (dashboards)
- Distributed tracing (Micrometer Tracing)
- OpenTelemetry
```