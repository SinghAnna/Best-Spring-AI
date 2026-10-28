# Part 8: Spring AI — Embeddings & Vector Stores Deep Dive

> Embeddings ka core — **`EmbeddingModel` (text → `float[]`, cosine similarity) + `VectorStore` (`add` / `similaritySearch` / `delete`) + `SearchRequest` (topK + threshold + filterExpression)** — same embedding model ingestion + query dono ke liye, pgvector (prod) / Pinecone (managed) / Milvus (scale). [2253][2254][2256][2262]

---

## 1. Embeddings Mental Model

```text
Embedding = Text → vector (semantic meaning)
= "Java is a language" → [0.12, -0.34, ...]

Similar meaning = Similar vectors
= Cosine similarity high
```

### Embedding Flow

```text
"Java is a programming language"
   |
   v
EmbeddingModel
   |
   v
float[1536]  (OpenAI text-embedding-3-small)
```

Embedding — text → `float[]` (1536 dims OpenAI). [2253][2256]

### EmbeddingModel Interface

```java
public interface EmbeddingModel extends Model<EmbeddingRequest, EmbeddingResponse> {
    EmbeddingResponse call(EmbeddingRequest request);
    float[] embed(String text);
    List<float[]> embed(List<String> texts);
    int dimensions();
}
```

`EmbeddingModel` — `embed(String)`, `embed(List)`, `dimensions()`. [2253]

### Same Model Rule

```text
Ingestion model == Query model
= Same vector space
= Comparable
```

Same embedding model — ingestion + query (vector space match). [2262]

---

## 2. VectorStore Interface

```java
public interface VectorStore extends DocumentWriter {
    void add(List<Document> documents);
    Optional<Boolean> delete(List<String> idList);
    List<Document> similaritySearch(SearchRequest request);
}
```

`VectorStore` — `add` + `delete` + `similaritySearch`. [2254][2256]

### VectorStore Flow

```text
add(docs):
  docs -> EmbeddingModel -> vectors -> Store

similaritySearch(query):
  query -> EmbeddingModel -> vector -> Store (cosine) -> Top-K docs
```

`add` — embed + store; `similaritySearch` — embed + cosine + top-K. [2254][2259]

---

## 3. SearchRequest (Tuning)

```text
SearchRequest = Query + topK + threshold + filter
```

### SearchRequest Builder

```java
SearchRequest request = SearchRequest.builder()
    .query("What is embedding?")
    .topK(5)                    // Max results (default 4)
    .similarityThreshold(0.7)   // Min similarity (0.0-1.0)
    .filterExpression("type == 'concept' AND version == '1.0'")
    .build();

List<Document> results = vectorStore.similaritySearch(request);
```

`SearchRequest.builder()` — query + topK + threshold + filter. [2256][2262]

### Tuning Guide

| Parameter | Default | Guide |
|---|---|---|
| topK | 4 | 3-8 (RAG) |
| similarityThreshold | 0.0 | 0.7-0.8 (relevance) |
| filterExpression | null | tenant/type/version |

topK (3-8), threshold (0.7-0.8), filter (tenant/type). [2256][2262]

---

## 4. Metadata Filtering

```text
Filter = Metadata-based scoping (tenant, type, version)
```

### Filter Expressions

| Expression | Meaning |
|---|---|
| `type == 'concept'` | Equal |
| `type != 'concept'` | Not equal |
| `year >= 2020` | Greater/equal |
| `type in ['a','b']` | In list |
| `a == 1 AND b == 2` | AND |
| `a == 1 OR b == 2` | OR |
| `!(a == 1)` | NOT |

Filter expressions — ==, !=, >=, in, AND, OR, NOT. [2252][2263]

### FilterExpressionBuilder (Java)

```java
FilterExpressionBuilder b = new FilterExpressionBuilder();
Filter.Expression expr = b.and(
    b.eq("tenant", "acme"),
    b.gte("year", 2024)
).build();

List<Document> results = vectorStore.similaritySearch(
    SearchRequest.builder()
        .query("leave policy")
        .topK(5)
        .filterExpression(expr)
        .build());
```

`FilterExpressionBuilder` — eq/gte/and/or (type-safe). [2259][2262]

### Multi-Tenant Filtering

```text
tenant_id filter = Data isolation
= Tenant A only sees Tenant A docs
```

Multi-tenant — `tenant_id == 'X'` filter (isolation). [2259][2262]

---

## 5. Vector Store Comparison

| Store | Type | Hybrid Search | Scale | Use Case |
|---|---|---|---|---|
| pgvector | PostgreSQL extension | Best (SQL + vector) | Medium | Prod (SQL + vector) |
| Pinecone | Managed | Yes (metadata + full-text) | High | Serverless |
| Milvus | Open-source | Yes (expression) | Massive | High-scale |
| Qdrant | Open-source | Yes (built-in) | High | Self-hosted |
| Chroma | Lightweight | Basic | Small | Dev / small |
| SimpleVectorStore | In-memory | No | Tiny | Demo / test |

Vector stores — pgvector (hybrid best), Pinecone (managed), Milvus (scale), Qdrant (self-hosted). [2255][2258][2260][2264]

### Selection Guide

```text
SQL + vector (single DB)?  -> pgvector
Serverless / managed?      -> Pinecone
Massive scale?             -> Milvus
Self-hosted + hybrid?      -> Qdrant
Dev / demo?                -> SimpleVectorStore
```

Selection — pgvector (SQL+vector), Pinecone (managed), Milvus (scale), Qdrant (self-hosted). [2255][2258][2260]

---

## 6. pgvector Setup

### docker-compose.yml

```yaml
services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: vector_store
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
```

### pom.xml

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-vector-store-pgvector</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```

### application.yml

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/vector_store
    username: postgres
    password: postgres
  ai:
    vectorstore:
      pgvector:
        dimensions: 1536
        distance-type: cosine_distance
        index-type: hnsw
```

pgvector — `pgvector/pgvector:pg16` image, HNSW index, cosine distance. [2252][2257][2259]

---

## 7. Hybrid Search (Vector + Keyword)

```text
Hybrid = Vector (semantic) + Keyword (exact)
= Better recall (exact terms)
```

### Hybrid Pattern

```text
Vector search (semantic) -> Results A
Keyword search (exact)   -> Results B
Merge (RRF / rerank)     -> Final results
```

Hybrid — vector + keyword → merge (RRF/rerank). [2255][2258][2260]

### Why Hybrid

```text
Embeddings = Meaning (semantic)
Weak at: Part numbers, error codes, rare names

Keyword = Exact match
Strong at: IDs, codes, names

Hybrid = Best of both
```

Hybrid — semantic + exact (embeddings weak at exact terms). [2262][2264]

---

## 8. Full Working Example

### EmbeddingService

```java
package com.example.vectors.service;

import org.springframework.ai.document.Document;
import org.springframework.ai.embedding.EmbeddingModel;
import org.springframework.ai.vectorstore.SearchRequest;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;

@Service
public class EmbeddingService {

    private final VectorStore vectorStore;
    private final EmbeddingModel embeddingModel;

    public EmbeddingService(VectorStore vectorStore, EmbeddingModel embeddingModel) {
        this.vectorStore = vectorStore;
        this.embeddingModel = embeddingModel;
    }

    // 1. Add docs
    public void addDocs(List<Document> docs) {
        vectorStore.add(docs);
    }

    // 2. Similarity search
    public List<Document> search(String query, int topK, double threshold) {
        return vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(query)
                .topK(topK)
                .similarityThreshold(threshold)
                .build());
    }

    // 3. Filtered search (multi-tenant)
    public List<Document> searchTenant(String query, String tenant) {
        return vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(query)
                .topK(5)
                .filterExpression("tenant == '" + tenant + "'")
                .build());
    }

    // 4. Raw embedding inspect
    public Map<String, Object> inspect(String text) {
        float[] embedding = embeddingModel.embed(text);
        return Map.of(
            "text", text,
            "dimensions", embedding.length,
            "sample", java.util.Arrays.copyOfRange(embedding, 0, 5)
        );
    }
}
```

### VectorController

```java
package com.example.vectors.controller;

import com.example.vectors.service.EmbeddingService;
import org.springframework.ai.document.Document;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/vector")
public class VectorController {

    private final EmbeddingService service;

    public VectorController(EmbeddingService service) {
        this.service = service;
    }

    public record AddRequest(List<String> texts, String tenant) {}
    public record SearchRequest(String query, int topK, double threshold) {}
    public record TenantSearchRequest(String query, String tenant) {}

    @PostMapping("/add")
    public String add(@RequestBody AddRequest req) {
        List<Document> docs = req.texts().stream()
            .map(t -> new Document(t, Map.of("tenant", req.tenant())))
            .toList();
        service.addDocs(docs);
        return "Added " + docs.size();
    }

    @PostMapping("/search")
    public List<Document> search(@RequestBody SearchRequest req) {
        return service.search(req.query(), req.topK(), req.threshold());
    }

    @PostMapping("/search-tenant")
    public List<Document> searchTenant(@RequestBody TenantSearchRequest req) {
        return service.searchTenant(req.query(), req.tenant());
    }

    @PostMapping("/inspect")
    public Map<String, Object> inspect(@RequestBody Map<String, String> body) {
        return service.inspect(body.get("text"));
    }
}
```

---

## 9. Run + Test

```bash
# 1. Start pgvector
docker-compose up -d

# 2. API key
export OPENAI_API_KEY=sk-...

# 3. Run
mvn spring-boot:run

# 4. Add docs
curl -X POST http://localhost:8080/api/vector/add \
  -H "Content-Type: application/json" \
  -d '{"texts":["Java is a language","Python is a language"],"tenant":"acme"}'

# 5. Search
curl -X POST http://localhost:8080/api/vector/search \
  -H "Content-Type: application/json" \
  -d '{"query":"What is Java?","topK":3,"threshold":0.7}'

# 6. Tenant-filtered search
curl -X POST http://localhost:8080/api/vector/search-tenant \
  -H "Content-Type: application/json" \
  -d '{"query":"What is Java?","tenant":"acme"}'

# 7. Inspect embedding
curl -X POST http://localhost:8080/api/vector/inspect \
  -H "Content-Type: application/json" \
  -d '{"text":"What is Java?"}'
```

---

## 10. Common Gotchas

### Gotcha 1 — Different Embedding Models

```text
Problem:
Ingestion model ≠ Query model

Fix:
Same model (vector space match)
```

### Gotcha 2 — No Threshold

```text
Problem:
Irrelevant results

Fix:
similarityThreshold (0.7-0.8)
```

### Gotcha 3 — No Tenant Filter

```text
Problem:
Cross-tenant leak

Fix:
filterExpression (tenant_id)
```

### Gotcha 4 — Dimension Mismatch

```text
Problem:
1536 vs 768 = error

Fix:
Match (store config + model)
```

---

## 11. Interview Questions

| Question | Answer |
|---|---|
| Embedding kya? | Text → vector (semantic). |
| VectorStore interface? | add + delete + similaritySearch. |
| SearchRequest? | Query + topK + threshold + filter. |
| Metadata filtering? | tenant/type/version scoping. |
| Hybrid search? | Vector + keyword (better recall). |
| pgvector vs Pinecone? | SQL+vector vs managed. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| SQL + vector | pgvector |
| Managed | Pinecone |
| Massive scale | Milvus |
| Multi-tenant | filterExpression |
| Exact terms | Hybrid search |

---

## 12. Key Takeaways

```text
1. Embedding = text → vector.

2. Same model = ingestion + query.

3. VectorStore = add + delete + search.

4. SearchRequest = query + topK + threshold.

5. Filter = tenant/type/version.

6. pgvector = SQL + vector (best hybrid).

7. Pinecone = managed.

8. Milvus = massive scale.

9. Hybrid = semantic + exact.

10. Dimensions = match (1536 / 768).
```

---

## 13. Next Part Preview

**Part 9: QuestionAnswerAdvisor Deep Dive**

Isme hum cover karenge:

```text
- QuestionAnswerAdvisor deep dive
- Custom advisor
- RAG tuning (topK, threshold)
- Citations
- Advanced RAG patterns
```