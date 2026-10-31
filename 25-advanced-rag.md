# Part 25: Spring AI — Advanced RAG (HyDE + Reranking + Query Decomposition)

> Advanced RAG ka core — **`RetrievalAugmentationAdvisor` (modular: query transformers + retriever + post-processor) + `RewriteQueryTransformer` (query rewrite) + `MultiQueryExpander` (N variants, `includeOriginal`) + HyDE (hypothetical answer → embed → search) + `DocumentPostProcessor` (rerank / dedupe / compress) + RRF (Reciprocal Rank Fusion) + hybrid search (vector + BM25)** — two-stage retrieval (broad → rerank), cross-encoder (Cohere Rerank / Flashrank), latency trade-off. [2225][2280][2472][2476]

---

## 1. Advanced RAG Mental Model

```text
Problem:
Naive RAG = Vector search only
= Semantic gap (question vs doc)
= Top-k noise (irrelevant chunks)
= Complex query fail (multi-part)

Solution:
Advanced RAG:
1. Query transformation (rewrite / expand / HyDE)
2. Hybrid search (vector + BM25)
3. Reranking (cross-encoder)
4. Fusion (RRF)
```

### Naive vs Advanced RAG

| Stage | Naive | Advanced |
|---|---|---|
| Query | Raw | Rewrite / expand / HyDE |
| Retrieval | Vector only | Vector + BM25 (hybrid) |
| Ranking | Similarity score | Cross-encoder rerank |
| Fusion | Single list | RRF (multi-list) |
| Context | Top-k raw | Filtered + compressed |

Naive vs advanced — query + retrieval + ranking + fusion + context. [2225][2288][2476]

### Advanced RAG Pipeline

```text
Query → Transform (rewrite / expand / HyDE)
     → Retrieve (vector + BM25)
     → Fuse (RRF)
     → Rerank (cross-encoder)
     → Compress (dedupe / filter)
     → Generate (LLM)
```

Pipeline — transform → retrieve → fuse → rerank → compress → generate. [2225][2473][2476]

---

## 2. Query Transformation

```text
Query transformation = Improve query before retrieval
= Rewrite / expand / decompose
```

### Query Transformation Types

| Type | What | When |
|---|---|---|
| Rewrite | Rephrase (clearer) | Vague query |
| Expand | N variants | Different phrasings |
| Decompose | Sub-queries | Complex / multi-part |
| HyDE | Hypothetical answer | Vocabulary gap |

Query transformation — rewrite + expand + decompose + HyDE. [2225][2288][2480]

### RewriteQueryTransformer

```java
QueryTransformer transformer = RewriteQueryTransformer.builder()
    .chatClientBuilder(chatClientBuilder.build().mutate())
    .build();

Query rewritten = transformer.transform(new Query(userQuery));
```

`RewriteQueryTransformer` — LLM rephrase (clearer query). [2225][2471]

### MultiQueryExpander

```java
MultiQueryExpander expander = MultiQueryExpander.builder()
    .chatClientBuilder(chatClientBuilder)
    .numberOfQueries(3)
    .includeOriginal(true)   // Default: true
    .build();

List<Query> queries = expander.expand(new Query("What's our refund policy?"));
// [original, variant1, variant2, variant3]
```

`MultiQueryExpander` — N variants + `includeOriginal` (default true). [2225][2280]

### Multi-Query Retrieval

```text
Multi-query = N variants → N searches
= Union + dedupe
= Higher recall
```

Multi-query — N variants → N searches → union + dedupe. [2480]

### Query Decomposition

```text
Decomposition = Complex query → sub-queries
= "Compare A and B" → "What is A?" + "What is B?"
= Each sub-query retrieve
= Merge results
```

Decomposition — complex → sub-queries (each retrieve). [2473][2479]

---

## 3. HyDE (Hypothetical Document Embeddings)

```text
HyDE = Hypothetical answer → embed → search
= Vocabulary gap bridge
```

### HyDE Flow

```text
1. Query → LLM → Hypothetical answer (fake doc)
2. Hypothetical answer → Embed → Vector
3. Vector → Similarity search → Real docs
4. Real docs → LLM → Final answer
```

HyDE — hypothetical answer → embed → search → real docs → answer. [2472][2477][2482]

### Why HyDE Works

```text
Question: "How do I reset the transaction manager?"
= Short + question-style

Relevant doc: "To rollback the platform transaction..."
= Long + answer-style

Gap = Question vs answer (semantic mismatch)

HyDE:
LLM generate hypothetical answer (answer-style)
= Embed → search → match real docs
```

HyDE — question vs answer gap (hypothetical bridge). [2472][2477][2480]

### HyDE Implementation

```java
public class HyDERetriever {

    private final ChatClient chatClient;
    private final EmbeddingModel embeddingModel;
    private final VectorStore vectorStore;

    public HyDERetriever(ChatClient chatClient, 
                         EmbeddingModel embeddingModel,
                         VectorStore vectorStore) {
        this.chatClient = chatClient;
        this.embeddingModel = embeddingModel;
        this.vectorStore = vectorStore;
    }

    public List<Document> retrieve(String query) {
        // 1. Generate hypothetical answer
        String hypothetical = chatClient.prompt()
            .user("Write a detailed answer to: " + query)
            .call()
            .content();

        // 2. Embed hypothetical answer
        float[] embedding = embeddingModel.embed(hypothetical);

        // 3. Search with hypothetical embedding
        return vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(query)
                .topK(5)
                .build());
    }
}
```

HyDE — generate → embed → search (hypothetical as probe). [2472][2477]

### HyDE Trade-off

```text
+ Higher recall (vocabulary gap bridge)
- Extra LLM call (latency + cost)
- Hypothetical may be wrong (but only probe)
```

HyDE trade-off — recall up, latency + cost up. [2472][2477]

---

## 4. Reranking (Cross-Encoder)

```text
Reranking = Two-stage retrieval
= Broad (top 20-50) → Rerank → Top 5
```

### Reranking Flow

```text
1. Broad retrieval (vector / hybrid) → Top 20-50
2. Cross-encoder → Score each (query, doc)
3. Sort by score → Top 5
4. Top 5 → LLM (context)
```

Reranking — broad → cross-encoder → top 5 → LLM. [2474][2476][2479]

### Bi-Encoder vs Cross-Encoder

| Type | Input | Speed | Accuracy |
|---|---|---|---|
| Bi-encoder | Query + doc separate | Fast | Lower |
| Cross-encoder | Query + doc together | Slow | Higher |

Bi-encoder vs cross-encoder — separate vs together (speed vs accuracy). [2474][2476]

### Why Cross-Encoder Better

```text
Bi-encoder = Query + doc separate → cosine
= No interaction (limited)

Cross-encoder = Query + doc together → score
= Full interaction (better)
```

Cross-encoder — query + doc together (full interaction). [2474][2476]

### Reranking Implementation (Cohere)

```java
public class RerankingService {

    private final RestClient restClient;

    public RerankingService(RestClient restClient) {
        this.restClient = restClient;
    }

    public List<Document> rerank(String query, List<Document> docs) {
        // Cohere Rerank API
        // Returns ranked doc IDs
        // Map back to documents
        return docs; // Simplified
    }
}
```

Reranking — Cohere Rerank API (cross-encoder). [2288][2476]

### Reranking Trade-off

```text
+ Higher precision (top 5 relevant)
- Extra API call (latency + cost)
- Bounded candidates (10-20, not all)
```

Reranking trade-off — precision up, latency + cost up (bounded). [2476][2479]

---

## 5. Hybrid Search + RRF

```text
Hybrid search = Vector + BM25
= Semantic + keyword
= RRF fuse
```

### Hybrid Search Flow

```text
1. Vector search (semantic) → Top 20
2. BM25 search (keyword) → Top 20
3. RRF fuse → Single ranked list
4. Rerank → Top 5
```

Hybrid — vector + BM25 → RRF → rerank. [2288][2476]

### RRF (Reciprocal Rank Fusion)

```text
RRF = Rank-based fusion (ignore absolute scores)
= 1 / (k + rank)
= k = 60 (default)
```

RRF — rank-based (1 / (k + rank), k=60). [2473][2476]

### RRF Formula

```text
RRF(d) = Σ (1 / (k + rank_i(d)))

Example:
Doc A: rank 1 (vector), rank 3 (BM25)
= 1/(60+1) + 1/(60+3) = 0.0164 + 0.0159 = 0.0323

Doc B: rank 5 (vector), rank 1 (BM25)
= 1/(60+5) + 1/(60+1) = 0.0154 + 0.0164 = 0.0318

A > B (higher RRF)
```

RRF — rank-based (higher = better). [2473][2476]

### RRF Why

```text
RRF = Rank-based (not score-based)
= Different scales (vector vs BM25) OK
= Robust (no score normalization)
```

RRF — rank-based (different scales OK, robust). [2473][2476]

---

## 6. DocumentPostProcessor

```text
DocumentPostProcessor = Post-process retrieved docs
= Rerank / dedupe / compress
```

### DocumentPostProcessor API

```java
public interface DocumentPostProcessor {
    Query transform(Query query, List<List<Document>> documents);
}
```

`DocumentPostProcessor` — rerank / dedupe / compress. [2225][2471]

### Post-Processor Use Cases

```text
1. Rerank (cross-encoder)
2. Dedupe (remove redundant)
3. Compress (reduce noise)
4. Filter (remove irrelevant)
```

Post-processor — rerank + dedupe + compress + filter. [2225][2471]

---

## 7. Full Working Example

### AdvancedRagConfig

```java
package com.example.advancedrag.config;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.client.advisor.RetrievalAugmentationAdvisor;
import org.springframework.ai.rag.advisor.RetrievalAugmentationAdvisor;
import org.springframework.ai.rag.retrieval.search.VectorStoreDocumentRetriever;
import org.springframework.ai.transformer.RewriteQueryTransformer;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AdvancedRagConfig {

    @Bean
    public RetrievalAugmentationAdvisor retrievalAdvisor(
        ChatClient.Builder chatClientBuilder, VectorStore vectorStore) {

        return RetrievalAugmentationAdvisor.builder()
            .queryTransformers(
                RewriteQueryTransformer.builder()
                    .chatClientBuilder(chatClientBuilder.build().mutate())
                    .build())
            .documentRetriever(
                VectorStoreDocumentRetriever.builder()
                    .similarityThreshold(0.50)
                    .vectorStore(vectorStore)
                    .build())
            .build();
    }
}
```

### AdvancedRagService

```java
package com.example.advancedrag.service;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.client.advisor.RetrievalAugmentationAdvisor;
import org.springframework.stereotype.Service;

@Service
public class AdvancedRagService {

    private final ChatClient chatClient;
    private final RetrievalAugmentationAdvisor retrievalAdvisor;

    public AdvancedRagService(ChatClient chatClient, 
                              RetrievalAugmentationAdvisor retrievalAdvisor) {
        this.chatClient = chatClient;
        this.retrievalAdvisor = retrievalAdvisor;
    }

    public String answer(String question) {
        return chatClient.prompt()
            .advisors(retrievalAdvisor)
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
@RequestMapping("/api/rag")
public class AdvancedRagController {

    private final AdvancedRagService ragService;

    public AdvancedRagController(AdvancedRagService ragService) {
        this.ragService = ragService;
    }

    public record RagRequest(String question) {}
    public record RagResponse(String answer) {}

    @PostMapping
    public RagResponse ask(@RequestBody RagRequest req) {
        return new RagResponse(ragService.answer(req.question()));
    }
}
```

---

## 8. Run + Test

```bash
export OPENAI_API_KEY=sk-...
mvn spring-boot:run

# Advanced RAG (rewrite + retrieve)
curl -X POST http://localhost:8080/api/rag \
  -H "Content-Type: application/json" \
  -d '{"question":"What is our refund policy?"}'
```

---

## 9. Common Gotchas

### Gotcha 1 — HyDE Always

```text
Problem:
Extra LLM call (latency + cost)

Fix:
HyDE only for hard queries (fallback)
```

### Gotcha 2 — Rerank All Docs

```text
Problem:
Latency blow

Fix:
Bounded candidates (10-20)
```

### Gotcha 3 — No Fusion

```text
Problem:
Vector + BM25 separate (no merge)

Fix:
RRF (rank-based fusion)
```

### Gotcha 4 — No Compression

```text
Problem:
Noise in context

Fix:
DocumentPostProcessor (dedupe / compress)
```

---

## 10. Interview Questions

| Question | Answer |
|---|---|
| Advanced RAG kya? | Transform + hybrid + rerank + fusion. |
| HyDE kya? | Hypothetical answer → embed → search. |
| Reranking kya? | Two-stage (broad → cross-encoder → top 5). |
| RRF kya? | Rank-based fusion (1/(k+rank)). |
| Multi-query kya? | N variants → N searches → union. |
| DocumentPostProcessor? | Rerank / dedupe / compress. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Vocabulary gap | HyDE |
| Different phrasings | Multi-query expander |
| Top-k noise | Reranking (cross-encoder) |
| Complex query | Query decomposition |
| Vector + keyword | Hybrid search + RRF |

---

## 11. Key Takeaways

```text
1. Advanced RAG = transform + hybrid + rerank + fusion.

2. Query transform = rewrite + expand + decompose + HyDE.

3. HyDE = hypothetical answer → embed → search.

4. Reranking = broad → cross-encoder → top 5.

5. RRF = rank-based fusion (k=60).

6. Hybrid search = vector + BM25.

7. DocumentPostProcessor = rerank / dedupe / compress.

8. RetrievalAugmentationAdvisor = modular (transformer + retriever + post-processor).

9. Two-stage = broad (fast) → rerank (precise).

10. Trade-off = recall vs latency vs cost.
```

---

## 12. Next Part Preview

**Part 26: Multi-Agent Systems (Supervisor + Worker Pattern)**

Isme hum cover karenge:

```text
- Multi-agent deep dive
- Supervisor + worker pattern
- Agent communication
- Task delegation
- State management
```