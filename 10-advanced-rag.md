# Part 10: Spring AI — Advanced RAG

> Advanced RAG ka core — **`RetrievalAugmentationAdvisor` (modular RAG) + `MultiQueryExpander` (query expansion) + `CompressionQueryTransformer` (history → standalone query) + hybrid search (vector + BM25, RRF) + reranking (cross-encoder, top 50 → top 5)** — "Lost in the Middle" fix + precision/recall boost. [2280][2282][2283][2288]

---

## 1. Advanced RAG Mental Model

```text
Naive RAG:
Query -> Vector search -> Top-K -> LLM
= Basic, gaps hain

Advanced RAG:
Query -> Transform/Expand -> Hybrid search -> Rerank -> Top-N -> LLM
= Precision + recall boost
```

### Naive vs Advanced

| Aspect | Naive | Advanced |
|---|---|---|
| Query | Raw | Rewrite + expand |
| Retrieval | Vector only | Hybrid (vector + BM25) |
| Ranking | Similarity | Rerank (cross-encoder) |
| Context | Top-K all | Compressed + filtered |
| Accuracy | Baseline | Higher |

Naive vs advanced — query transform + hybrid + rerank. [2282][2288]

### Advanced RAG Categories

```text
Pre-retrieval:  Query transform, HyDE, expansion
Retrieval-time: Hybrid search, contextual chunking
Post-retrieval: Rerank, compression, self-reflection
Architecture:   Self-RAG, Adaptive RAG, Modular RAG
```

Advanced RAG — pre / retrieval / post / architecture. [2282]

---

## 2. RetrievalAugmentationAdvisor (Modular RAG)

```text
Modular RAG = Building blocks (pluggable)
= Pre-retrieval + Retrieval + Post-retrieval
```

### Modular Components

| Component | Interface | Purpose |
|---|---|---|
| QueryTransformer | `Query -> Query` | Rewrite / compress |
| QueryExpander | `Query -> List<Query>` | Multi-query |
| DocumentRetriever | `Query -> List<Document>` | Retrieve |
| DocumentJoiner | `List<Query> -> List<Document>` | Merge (RRF) |
| QueryAugmenter | `Query + Docs -> Prompt` | Inject context |
| DocumentPostProcessor | `Docs -> Docs` | Rerank / filter |

Modular RAG — QueryTransformer, QueryExpander, DocumentRetriever, DocumentJoiner, QueryAugmenter, DocumentPostProcessor. [2281][2283]

### RetrievalAugmentationAdvisor Builder

```java
RetrievalAugmentationAdvisor advisor = RetrievalAugmentationAdvisor.builder()
    .queryTransformers(compressionTransformer)
    .queryExpander(multiQueryExpander)
    .documentRetriever(vectorStoreRetriever)
    .documentPostProcessors(reranker)
    .queryAugmenter(ContextualQueryAugmenter.builder()
        .allowEmptyContext(true)
        .build())
    .build();
```

`RetrievalAugmentationAdvisor.builder()` — modular composition. [2280][2281][2283]

---

## 3. Query Transformation

```text
Query Transform = User query -> Better query
```

### CompressionQueryTransformer (History → Standalone)

```text
Problem:
"What about premium users?" (no context)
= Ambiguous

Solution:
History + follow-up -> Standalone query
= "What are the features for premium users?"
```

```java
CompressionQueryTransformer transformer = 
    CompressionQueryTransformer.builder()
        .chatClientBuilder(chatClientBuilder)
        .build();

Query query = Query.builder()
    .text("What about premium users?")
    .history(previousMessages)
    .build();

Query transformed = transformer.transform(query);
```

`CompressionQueryTransformer` — history + follow-up → standalone. [2280][2283]

### RewriteQueryTransformer

```java
RewriteQueryTransformer transformer = 
    RewriteQueryTransformer.builder()
        .chatClientBuilder(chatClientBuilder)
        .targetSearchSystem("vector store")
        .build();

Query rewritten = transformer.transform(query);
```

`RewriteQueryTransformer` — LLM se query rewrite (better retrieval). [2283]

---

## 4. Multi-Query Expansion (RAG Fusion)

```text
Multi-Query = 1 query -> N queries
= Different perspectives
= Better recall
```

### MultiQueryExpander

```java
MultiQueryExpander expander = MultiQueryExpander.builder()
    .chatClientBuilder(chatClientBuilder)
    .numberOfQueries(3)
    .build();

List<Query> queries = expander.expand(
    new Query("What's our refund policy?"));

// ["What is the refund policy?",
//  "How do I get a refund?",
//  "What are the refund terms?"]
```

`MultiQueryExpander` — 1 query → N queries (perspectives). [2280][2283]

### RAG Fusion Flow

```text
Query -> 3 variants -> 3 searches -> RRF merge -> Top-K
```

RAG Fusion — multi-query + RRF merge (broader coverage). [2282][2289]

### Why Multi-Query

```text
Ambiguous query:
"Compare Enterprise vs Pro pricing"
= 2 topics, 1 query fails

Multi-query:
1. "Enterprise Plan pricing"
2. "Pro Plan pricing"
= Both retrieved
```

Multi-query — ambiguous / multi-topic queries. [2288]

---

## 5. Hybrid Search (Vector + BM25)

```text
Hybrid = Dense (semantic) + Sparse (keyword)
= Best of both
```

### Hybrid Flow

```text
Query -> Vector search (top 30) -> Results A
Query -> BM25 search (top 30)   -> Results B
A + B -> RRF merge              -> Top-K
```

Hybrid — vector + BM25 → RRF merge. [2286][2288]

### RRF (Reciprocal Rank Fusion)

```text
RRF = Rank-based merge (scores ignore)
= 1/(k + rank) sum

Example:
Doc A: Vector rank 1, BM25 rank 3
RRF = 1/(60+1) + 1/(60+3) = 0.032

Doc B: Vector rank 3, BM25 rank 1
RRF = 1/(60+3) + 1/(60+1) = 0.032
```

RRF — rank-based (k=60 default), scores ignore. [2286][2288]

### Why Hybrid

```text
Embeddings weak at: Exact IDs, error codes, rare names
Keyword weak at: Paraphrasing, synonyms

Hybrid = Both covered
```

Hybrid — semantic + exact (embeddings weak at IDs/codes). [2286][2288]

---

## 6. Reranking (Cross-Encoder)

```text
Rerank = Second pass (precision)
= Top 50 -> Top 5
```

### Bi-Encoder vs Cross-Encoder

| Model | Input | Speed | Accuracy |
|---|---|---|---|
| Bi-Encoder | Query + Doc separately | Fast | Lower |
| Cross-Encoder | Query + Doc together | Slow | Higher |

Bi-encoder (fast, separate) vs Cross-encoder (slow, together, accurate). [2284][2288]

### Rerank Pipeline

```text
1. Retrieval: Vector + BM25 -> Top 50 (fast)
2. Rerank: Cross-encoder -> Score 0-1 -> Top 5 (precise)
3. LLM: Top 5 context
```

Rerank — 50 candidates → 5 precise. [2286][2288]

### Why Rerank

```text
"Lost in the Middle":
LLM middle chunks ignore

Rerank:
Best chunks first/last
= Attention on relevant
```

Rerank — "Lost in the Middle" fix. [2286][2288]

### Reranker Implementation (Cohere)

```java
@Service
public class RerankingService {

    private final RestClient restClient;

    public RerankingService(RestClient.Builder builder) {
        this.restClient = builder
            .baseUrl("[https://api.cohere.com/v2](https://api.cohere.com/v2)")
            .defaultHeader("Authorization", "Bearer " + apiKey)
            .build();
    }

    public List<Document> rerank(String query, List<Document> candidates, int topN) {
        var response = restClient.post()
            .uri("/rerank")
            .body(Map.of(
                "model", "rerank-v3.5",
                "query", query,
                "top_n", topN,
                "documents", candidates.stream()
                    .map(Document::getText)
                    .toList()))
            .retrieve()
            .body(RerankResponse.class);

        return response.results().stream()
            .map(r -> candidates.get(r.index()))
            .toList();
    }
}
```

Reranker — Cohere API (cross-encoder), top 50 → top 5. [2287][2288][2291]

---

## 7. Full Working Example

### AdvancedRagConfig

```java
package com.example.advancedrag.config;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.rag.advisor.RetrievalAugmentationAdvisor;
import org.springframework.ai.rag.preretrieval.query.expansion.MultiQueryExpander;
import org.springframework.ai.rag.preretrieval.query.transformation.CompressionQueryTransformer;
import org.springframework.ai.rag.retrieval.search.VectorStoreDocumentRetriever;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AdvancedRagConfig {

    @Bean
    public ChatClient chatClient(
        ChatClient.Builder builder, VectorStore vectorStore) {
        return builder.build();
    }

    @Bean
    public CompressionQueryTransformer compressionTransformer(
        ChatClient.Builder builder) {
        return CompressionQueryTransformer.builder()
            .chatClientBuilder(builder)
            .build();
    }

    @Bean
    public MultiQueryExpander multiQueryExpander(ChatClient.Builder builder) {
        return MultiQueryExpander.builder()
            .chatClientBuilder(builder)
            .numberOfQueries(3)
            .build();
    }

    @Bean
    public RetrievalAugmentationAdvisor advancedRagAdvisor(
        VectorStore vectorStore,
        CompressionQueryTransformer compressionTransformer,
        MultiQueryExpander multiQueryExpander) {
        return RetrievalAugmentationAdvisor.builder()
            .queryTransformers(compressionTransformer)
            .queryExpander(multiQueryExpander)
            .documentRetriever(VectorStoreDocumentRetriever.builder()
                .vectorStore(vectorStore)
                .similarityThreshold(0.5)
                .topK(20)
                .build())
            .queryAugmenter(ContextualQueryAugmenter.builder()
                .allowEmptyContext(true)
                .build())
            .build();
    }
}
```

### AdvancedRagService

```java
package com.example.advancedrag.service;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.rag.advisor.RetrievalAugmentationAdvisor;
import org.springframework.stereotype.Service;

@Service
public class AdvancedRagService {

    private final ChatClient chatClient;
    private final RetrievalAugmentationAdvisor advisor;

    public AdvancedRagService(ChatClient chatClient, 
                              RetrievalAugmentationAdvisor advisor) {
        this.chatClient = chatClient;
        this.advisor = advisor;
    }

    public String ask(String question) {
        return chatClient.prompt()
            .advisors(advisor)
            .user(question)
            .call()
            .content();
    }
}
```

### AdvancedRagController

```java
package com.example.advancedrag.controller;

import com.example.advancedrag.service.AdvancedRagService;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/advanced-rag")
public class AdvancedRagController {

    private final AdvancedRagService service;

    public AdvancedRagController(AdvancedRagService service) {
        this.service = service;
    }

    public record QuestionRequest(String question) {}
    public record AnswerResponse(String answer) {}

    @PostMapping("/ask")
    public AnswerResponse ask(@RequestBody QuestionRequest req) {
        return new AnswerResponse(service.ask(req.question()));
    }
}
```

---

## 8. Run + Test

```bash
export OPENAI_API_KEY=sk-...
mvn spring-boot:run

# Ingest docs (Part 7 se)
curl -X POST http://localhost:8080/api/etl/ingest

# Advanced RAG (multi-query + compression + hybrid)
curl -X POST http://localhost:8080/api/advanced-rag/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Compare Enterprise vs Pro pricing"}'
```

---

## 9. Common Gotchas

### Gotcha 1 — No Query Transform

```text
Problem:
Ambiguous query fail

Fix:
CompressionQueryTransformer
```

### Gotcha 2 — Rerank on All

```text
Problem:
Slow + expensive

Fix:
Top 50 -> Rerank -> Top 5
```

### Gotcha 3 — No Hybrid

```text
Problem:
Exact terms miss

Fix:
Vector + BM25 (RRF)
```

### Gotcha 4 — Too Many Chunks

```text
Problem:
"Lost in the Middle"

Fix:
Rerank + top 5
```

---

## 10. Interview Questions

| Question | Answer |
|---|---|
| Advanced RAG kya? | Query transform + hybrid + rerank. |
| Modular RAG? | Pluggable components (RetrievalAugmentationAdvisor). |
| Multi-query? | 1 → N queries (RAG Fusion). |
| Hybrid search? | Vector + BM25 (RRF). |
| Rerank kya? | Cross-encoder (top 50 → top 5). |
| "Lost in the Middle"? | Middle chunks ignore (rerank fix). |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Ambiguous query | Compression + multi-query |
| Exact terms | Hybrid search |
| Precision | Rerank (cross-encoder) |
| Context limit | Rerank + top 5 |
| Modular RAG | RetrievalAugmentationAdvisor |

---

## 11. Key Takeaways

```text
1. Advanced RAG = transform + hybrid + rerank.

2. Modular RAG = pluggable components.

3. Multi-query = 1 → N (RAG Fusion).

4. Hybrid = vector + BM25 (RRF).

5. Rerank = cross-encoder (precision).

6. "Lost in the Middle" = rerank fix.

7. Compression = history → standalone.

8. Rewrite = better retrieval.

9. Top 50 → top 5 = rerank.

10. Precision + recall = advanced RAG.
```

---

## 12. Next Part Preview

**Part 11: Function Calling Basics**

Isme hum cover karenge:

```text
- Function calling deep dive
- @Tool annotation
- JSON schema
- Tool loop
- Real-world tools
```