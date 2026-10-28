# Part 6: Spring AI — RAG Fundamentals

> RAG ka core — **ETL pipeline (DocumentReader → Transformer → VectorStore) + EmbeddingModel (text → vector) + QuestionAnswerAdvisor (one-liner RAG)** — apne documents pe Q&A, `SimpleVectorStore` (in-memory) se start, pgvector (prod) pe scale. [2225][2228][2234]

---

## 1. RAG Mental Model

```text
Problem:
LLM apne training data tak limited
= Company docs nahi jaanta
= Outdated info

Solution:
RAG = Retrieval + Augmentation + Generation
= Apne docs se relevant chunks fetch
= Prompt me inject
= LLM answer kare
```

### RAG vs Fine-Tuning

| Aspect | RAG | Fine-Tuning |
|---|---|---|
| Purpose | New knowledge inject | Behavior/tone change |
| Data | External (docs, DB) | Labeled training data |
| Update | Easy (add docs) | Hard (retrain) |
| Cost | Low upfront, per-query | High upfront, low per-query |
| Use case | Docs Q&A, support | Style, domain task |

RAG = knowledge inject, Fine-tuning = behavior change. [2223][2226][2227]

### RAG Flow (4 Stages)

```text
1. Query    -> User question
2. Retrieve -> Vector store se relevant chunks
3. Augment  -> Chunks + question -> prompt
4. Generate -> LLM answer
```

RAG 4 stages — query → retrieve → augment → generate. [2223][2226]

---

## 2. RAG Architecture

```text
Ingestion (Offline):
Docs -> Reader -> Splitter -> EmbeddingModel -> VectorStore

Query (Online):
Question -> EmbeddingModel -> VectorStore (similarity search)
   -> Top-K chunks -> Prompt -> LLM -> Answer
```

### Components

| Component | Role |
|---|---|
| DocumentReader | PDF/HTML/Text → List<Document> |
| DocumentTransformer | Split + clean + metadata |
| EmbeddingModel | Text → vector |
| VectorStore | Vector storage + similarity search |
| QuestionAnswerAdvisor | RAG glue (retrieve + inject) |

RAG components — reader, transformer, embedding, vector store, advisor. [2225][2228][2230]

---

## 3. ETL Pipeline

```text
ETL = Extract → Transform → Load
```

### ETL Interfaces

| Interface | Method | Purpose |
|---|---|---|
| DocumentReader | `read()` | Extract → List<Document> |
| DocumentTransformer | `transform()` | Process → List<Document> |
| DocumentWriter | `write()` | Store (VectorStore) |

ETL — `DocumentReader.read()` → `DocumentTransformer.transform()` → `DocumentWriter.write()`. [2228]

### ETL Flow

```text
PDF -> TextReader -> TokenTextSplitter -> VectorStore
   |                    |                    |
   v                    v                    v
Extract            Transform            Load
```

ETL — extract (reader) → transform (splitter) → load (vector store). [2228][2230]

---

## 4. Embeddings & Vector Stores

```text
Embedding = Text → vector (semantic meaning)
VectorStore = Vector storage + similarity search
```

### Embedding Flow

```text
"Java is a programming language"
   |
   v
EmbeddingModel
   |
   v
[0.12, -0.34, 0.56, ...]  (1536 dims)
```

Embedding — text → vector (semantic). [2226][2232]

### Similarity Search

```text
Query: "What is Java?"
   |
   v
Embedding
   |
   v
VectorStore (cosine similarity)
   |
   v
Top-K chunks (most similar)
```

Similarity search — query embedding → cosine → top-K. [2226][2232]

### Vector Store Options

| Store | Type | Use Case |
|---|---|---|
| SimpleVectorStore | In-memory | Demo / test |
| PGVector | PostgreSQL | Prod (SQL + vector) |
| Milvus | Dedicated | Scale |
| Pinecone | Managed | Serverless |
| Chroma | Lightweight | Dev / small |

Vector stores — Simple (demo), PGVector (prod), Milvus (scale), Pinecone (managed). [2232][2234]

---

## 5. QuestionAnswerAdvisor (One-Liner RAG)

```text
QuestionAnswerAdvisor = RAG glue
= Retrieve + inject + generate (one line)
```

### Basic Usage

```java
String answer = chatClient.prompt()
    .advisors(QuestionAnswerAdvisor.builder(vectorStore).build())
    .user("What is Spring AI?")
    .call()
    .content();
```

`QuestionAnswerAdvisor.builder(vectorStore)` — one-liner RAG. [2225][2228]

### Behind the Scenes

```text
1. User question -> EmbeddingModel
2. VectorStore -> Similarity search (top-K)
3. Chunks -> Prompt me inject
4. LLM -> Answer (grounded)
```

Advisor — embed → search → inject → generate. [2225][2228][2234]

### Custom Search Request

```java
QuestionAnswerAdvisor advisor = QuestionAnswerAdvisor.builder(vectorStore)
    .searchRequest(SearchRequest.builder()
        .topK(4)
        .similarityThreshold(0.75)
        .build())
    .build();
```

`SearchRequest` — topK + similarityThreshold (tuning). [2234]

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
    <artifactId>spring-ai-vector-store</artifactId>
</dependency>
```

### RagConfig

```java
package com.example.rag.config;

import org.springframework.ai.embedding.EmbeddingModel;
import org.springframework.ai.vectorstore.SimpleVectorStore;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RagConfig {

    @Bean
    public VectorStore vectorStore(EmbeddingModel embeddingModel) {
        return SimpleVectorStore.builder(embeddingModel).build();
    }
}
```

### DocumentIngestionService

```java
package com.example.rag.service;

import org.springframework.ai.document.Document;
import org.springframework.ai.reader.TextReader;
import org.springframework.ai.transformer.splitter.TokenTextSplitter;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.core.io.Resource;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class DocumentIngestionService {

    private final VectorStore vectorStore;

    public DocumentIngestionService(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }

    public void ingest(Resource resource) {
        // 1. Extract
        TextReader reader = new TextReader(resource);
        List<Document> documents = reader.read();

        // 2. Transform (split)
        TokenTextSplitter splitter = new TokenTextSplitter();
        List<Document> chunks = splitter.apply(documents);

        // 3. Load
        vectorStore.add(chunks);
    }
}
```

### RagChatService

```java
package com.example.rag.service;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.vectorstore.SearchRequest;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.ai.vectorstore.advisor.api.QuestionAnswerAdvisor;
import org.springframework.stereotype.Service;

@Service
public class RagChatService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;

    public RagChatService(ChatClient.Builder builder, VectorStore vectorStore) {
        this.chatClient = builder.build();
        this.vectorStore = vectorStore;
    }

    public String ask(String question) {
        QuestionAnswerAdvisor advisor = QuestionAnswerAdvisor.builder(vectorStore)
            .searchRequest(SearchRequest.builder()
                .topK(4)
                .similarityThreshold(0.75)
                .build())
            .build();

        return chatClient.prompt()
            .advisors(advisor)
            .user(question)
            .call()
            .content();
    }
}
```

### RagController

```java
package com.example.rag.controller;

import com.example.rag.service.RagChatService;
import org.springframework.core.io.ClassPathResource;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import com.example.rag.service.DocumentIngestionService;

@RestController
@RequestMapping("/api/rag")
public class RagController {

    private final RagChatService chatService;
    private final DocumentIngestionService ingestionService;

    public RagController(RagChatService chatService, 
                         DocumentIngestionService ingestionService) {
        this.chatService = chatService;
        this.ingestionService = ingestionService;
    }

    public record QuestionRequest(String question) {}
    public record AnswerResponse(String answer) {}

    @PostMapping("/ingest")
    public ResponseEntity<String> ingest() {
        ingestionService.ingest(
            new ClassPathResource("docs/spring-ai-overview.txt"));
        return ResponseEntity.ok("Ingested");
    }

    @PostMapping("/ask")
    public AnswerResponse ask(@RequestBody QuestionRequest req) {
        return new AnswerResponse(chatService.ask(req.question()));
    }
}
```

### docs/spring-ai-overview.txt (Sample Doc)

```text
Spring AI is a Spring framework extension for AI applications.
It provides ChatClient for LLM calls, VectorStore for embeddings,
and QuestionAnswerAdvisor for RAG.
Spring AI supports OpenAI, Anthropic, Ollama, Bedrock.
```

---

## 7. Run + Test

```bash
export OPENAI_API_KEY=sk-...
mvn spring-boot:run

# 1. Ingest doc
curl -X POST http://localhost:8080/api/rag/ingest

# 2. Ask (RAG)
curl -X POST http://localhost:8080/api/rag/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"What is Spring AI?"}'

# Response:
{"answer":"Spring AI is a Spring framework extension for AI applications..."}
```

---

## 8. Common Gotchas

### Gotcha 1 — No Chunking

```text
Problem:
Whole doc = 1 chunk (context blow)

Fix:
TokenTextSplitter (chunking)
```

### Gotcha 2 — Wrong Dimensions

```text
Problem:
Embedding dims ≠ store dims

Fix:
Match (1536 = OpenAI, 768 = Ollama)
```

### Gotcha 3 — No Similarity Threshold

```text
Problem:
Irrelevant chunks inject

Fix:
similarityThreshold (0.75)
```

### Gotcha 4 — In-Memory in Prod

```text
Problem:
Restart = data loss

Fix:
PGVector (persistent)
```

---

## 9. Interview Questions

| Question | Answer |
|---|---|
| RAG kya? | Retrieval + Augmentation + Generation. |
| RAG vs fine-tuning? | Knowledge inject vs behavior change. |
| ETL pipeline? | Reader → Transformer → Writer. |
| Embedding kya? | Text → vector (semantic). |
| QuestionAnswerAdvisor? | One-liner RAG (retrieve + inject). |
| Vector store options? | Simple, PGVector, Milvus, Pinecone. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Docs Q&A | RAG + pgvector |
| Demo | SimpleVectorStore |
| Scale | Milvus / Pinecone |
| Relevance tuning | topK + threshold |
| Prod | PGVector + persistent |

---

## 10. Key Takeaways

```text
1. RAG = retrieve + augment + generate.

2. RAG vs fine-tuning = knowledge vs behavior.

3. ETL = reader → splitter → vector store.

4. Embedding = text → vector.

5. QuestionAnswerAdvisor = one-liner RAG.

6. topK + threshold = tuning.

7. SimpleVectorStore = demo.

8. PGVector = prod.

9. Chunking = context control.

10. Dimensions = match (1536 / 768).
```

---

## 11. Next Part Preview

**Part 7: Document Loading & ETL Deep Dive**

Isme hum cover karenge:

```text
- Document readers (PDF, HTML, Text, JSON)
- Transformers (splitter, metadata)
- Chunking strategies
- Metadata enrichment
- Batch ingestion
```