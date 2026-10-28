# Part 7: Spring AI — Document Loading & ETL Deep Dive

> ETL ka core — **DocumentReader (Text/PDF/Tika/JSON/Markdown/Jsoup) → DocumentTransformer (TokenTextSplitter, metadata enricher) → DocumentWriter (VectorStore)** — chunking strategy (500 tokens + 100 overlap default), metadata enrichment (keyword/summary), batch ingestion. [2237][2240][2245][2247]

---

## 1. ETL Mental Model

```text
ETL = Extract → Transform → Load
= Raw docs → Clean chunks → Vector store

Analogy:
Data pipeline (Airflow) -> RAG ingestion
```

### ETL Interfaces

| Interface | Signature | Purpose |
|---|---|---|
| DocumentReader | `Supplier<List<Document>>` | Extract (file → docs) |
| DocumentTransformer | `Function<List<Document>, List<Document>>` | Transform (clean + split) |
| DocumentWriter | `Consumer<List<Document>>` | Load (store) |

ETL — `DocumentReader` (Supplier), `DocumentTransformer` (Function), `DocumentWriter` (Consumer). [2237][2240]

### Document Class

```java
public class Document {
    private String id;          // Unique ID
    private String text;        // Content
    private Map<String,Object> metadata;  // Metadata
    private List<Media> media;  // Images/audio/video (optional)
}
```

`Document` — id + text + metadata + media. [2237][2250]

---

## 2. Document Readers

```text
Reader = File format → List<Document>
```

### Reader Options

| Reader | Format | Library |
|---|---|---|
| TextReader | .txt | Plain |
| PagePdfDocumentReader | .pdf (page-aware) | PDFBox |
| ParagraphPdfDocumentReader | .pdf (paragraph-aware) | PDFBox |
| TikaDocumentReader | PDF, DOCX, PPTX, HTML | Apache Tika |
| JsoupDocumentReader | .html | Jsoup |
| JsonReader | .json | Jackson |
| MarkdownDocumentReader | .md | CommonMark |

Readers — Text, PDF (page/paragraph), Tika (multi-format), Jsoup (HTML), JSON, Markdown. [2237][2238][2247]

### TextReader

```java
TextReader reader = new TextReader(resource);
reader.getCustomMetadata().put("source", "docs");
List<Document> docs = reader.read();
```

TextReader — plain text, custom metadata support. [2237]

### PagePdfDocumentReader (PDF)

```java
PagePdfDocumentReader reader = new PagePdfDocumentReader(
    resource,
    PdfDocumentReaderConfig.builder()
        .withPageTopMargin(0)
        .withPageBottomMargin(0)
        .withPageExtractedTextFormatter(
            ExtractedTextFormatter.builder()
                .withNumberOfTopTextLinesToDelete(1)   // Header remove
                .withNumberOfBottomTextLinesToDelete(1) // Footer remove
                .build())
        .withPagesPerDocument(1)
        .build());

List<Document> docs = reader.read();
```

PagePdfDocumentReader — page-aware, header/footer remove, pages-per-doc. [2237][2244]

### TikaDocumentReader (Multi-Format)

```java
TikaDocumentReader reader = new TikaDocumentReader(resource);
List<Document> docs = reader.read();
```

Tika — PDF, DOCX, PPTX, HTML (1000+ formats). [2238][2244]

### JsoupDocumentReader (HTML)

```java
JsoupDocumentReader reader = new JsoupDocumentReader(
    resource,
    JsoupDocumentReaderConfig.builder()
        .selector("article")        // Nav/footer exclude
        .charset("UTF-8")
        .build());

List<Document> docs = reader.read();
```

Jsoup — selector-based (nav/footer exclude). [2241][2247]

---

## 3. Document Transformers (Chunking)

```text
Transformer = Clean + Split + Enrich
```

### TokenTextSplitter

```java
TokenTextSplitter splitter = TokenTextSplitter.builder()
    .withChunkSize(500)              // Tokens per chunk
    .withMinChunkSizeChars(50)       // Min chars
    .withMinChunkLengthToEmbed(20)   // Min embed length
    .withMaxNumChunks(10000)         // Max chunks
    .withKeepSeparator(true)         // Keep separators
    .build();

List<Document> chunks = splitter.apply(documents);
```

TokenTextSplitter — chunkSize (500 default), minChunkSizeChars, keepSeparator. [2237][2242][2243]

### Chunking Strategies

| Strategy | Use Case | Chunk Size |
|---|---|---|
| Fixed-size + overlap | General prose | 500 tokens + 100 overlap |
| Recursive character | Production default | 500 tokens + 100 overlap |
| Semantic | Meaning-shift split | Embedding-based |
| Structure-aware | Docs with headers | Section-based |
| Adaptive | Mixed content | Dynamic |

Chunking — fixed, recursive, semantic, structure-aware, adaptive. [2245][2251]

### Chunking Guide

```text
Prose (blogs):      Recursive, 500 tokens, 100 overlap
Technical docs:     Structure-aware (headers)
Contracts/policy:   Structure-aware + section metadata
Code:               Language-aware (function/class)
Tables:             Row-per-chunk (headers included)
Chat/tickets:       Conversation-per-chunk
```

Chunking per doc type — prose, technical, contracts, code, tables, chat. [2245]

---

## 4. Metadata Enrichment

```text
Enrichment = Metadata add (filtering + context)
```

### Built-in Enrichers

| Enricher | Purpose |
|---|---|
| KeywordMetadataEnricher | LLM se keywords extract |
| SummaryMetadataEnricher | LLM se summary generate |

Enrichers — Keyword + Summary (LLM-based). [2247]

### KeywordMetadataEnricher

```java
KeywordMetadataEnricher enricher = 
    new KeywordMetadataEnricher(embeddingModel, 5);

List<Document> enriched = enricher.transform(chunks);
// Metadata: {"keywords": "java, spring, ai, ..."}
```

Keyword enricher — LLM se keywords (5 per doc). [2247]

### Custom Metadata (Manual)

```java
Document doc = new Document(text, Map.of(
    "source", "user-guide.pdf",
    "version", "2.1",
    "department", "engineering",
    "language", "en",
    "tenant_id", "tenant-1"
));
```

Custom metadata — source, version, department, language, tenant. [2237][2248]

### Metadata Use Cases

```text
- Multi-tenant filtering (tenant_id)
- Version filtering (version)
- Source attribution (citations)
- Department scoping (RBAC)
```

Metadata — tenant, version, source, department (filtering). [2237][2248]

---

## 5. Batch Ingestion

```text
Batch = Multiple docs, efficient (embedding batch)
```

### Batch Ingestion Pattern

```java
@Service
public class BatchIngestionService {

    private final VectorStore vectorStore;

    public BatchIngestionService(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }

    public void ingestBatch(List<Resource> resources) {
        List<Document> allDocs = new ArrayList<>();
        
        for (Resource resource : resources) {
            TikaDocumentReader reader = new TikaDocumentReader(resource);
            allDocs.addAll(reader.read());
        }

        TokenTextSplitter splitter = TokenTextSplitter.builder()
            .withChunkSize(500)
            .build();

        List<Document> chunks = splitter.apply(allDocs);
        vectorStore.add(chunks);  // Batch embed + store
    }
}
```

Batch — collect all → split → single `vectorStore.add()` (efficient). [2243][2248]

### Idempotent Ingestion

```text
Deterministic IDs = Re-run safe
```

```java
String docId = DigestUtils.md5DigestAsHex(
    (source + content).getBytes());

Document doc = new Document(docId, text, metadata);
```

Deterministic ID — re-run harmless (dedup). [2241][2245]

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
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-tika-document-reader</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-jsoup-document-reader</artifactId>
</dependency>
```

### EtlConfig

```java
package com.example.etl.config;

import org.springframework.ai.embedding.EmbeddingModel;
import org.springframework.ai.vectorstore.SimpleVectorStore;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class EtlConfig {

    @Bean
    public VectorStore vectorStore(EmbeddingModel embeddingModel) {
        return SimpleVectorStore.builder(embeddingModel).build();
    }
}
```

### EtlService

```java
package com.example.etl.service;

import org.springframework.ai.document.Document;
import org.springframework.ai.reader.TextReader;
import org.springframework.ai.reader.tika.TikaDocumentReader;
import org.springframework.ai.transformer.splitter.TokenTextSplitter;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.core.io.Resource;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;

@Service
public class EtlService {

    private final VectorStore vectorStore;

    public EtlService(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }

    // 1. Text file
    public int ingestText(Resource resource, String source) {
        TextReader reader = new TextReader(resource);
        reader.getCustomMetadata().put("source", source);
        List<Document> docs = reader.read();

        TokenTextSplitter splitter = TokenTextSplitter.builder()
            .withChunkSize(500)
            .withMinChunkLengthToEmbed(20)
            .build();

        List<Document> chunks = splitter.apply(docs);
        vectorStore.add(chunks);
        return chunks.size();
    }

    // 2. PDF (Tika)
    public int ingestPdf(Resource resource, String source) {
        TikaDocumentReader reader = new TikaDocumentReader(resource);
        List<Document> docs = reader.read();

        TokenTextSplitter splitter = TokenTextSplitter.builder()
            .withChunkSize(500)
            .build();

        List<Document> chunks = splitter.apply(docs);
        chunks.forEach(c -> c.getMetadata().put("source", source));
        vectorStore.add(chunks);
        return chunks.size();
    }

    // 3. Batch
    public int ingestBatch(List<Resource> resources, String source) {
        List<Document> all = new ArrayList<>();
        for (Resource r : resources) {
            all.addAll(new TikaDocumentReader(r).read());
        }
        TokenTextSplitter splitter = TokenTextSplitter.builder()
            .withChunkSize(500)
            .build();
        List<Document> chunks = splitter.apply(all);
        chunks.forEach(c -> c.getMetadata().put("source", source));
        vectorStore.add(chunks);
        return chunks.size();
    }
}
```

### EtlController

```java
package com.example.etl.controller;

import com.example.etl.service.EtlService;
import org.springframework.core.io.ClassPathResource;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/etl")
public class EtlController {

    private final EtlService etlService;

    public EtlController(EtlService etlService) {
        this.etlService = etlService;
    }

    @PostMapping("/text")
    public ResponseEntity<String> ingestText() {
        int count = etlService.ingestText(
            new ClassPathResource("docs/sample.txt"), "sample");
        return ResponseEntity.ok("Ingested " + count + " chunks");
    }

    @PostMapping("/pdf")
    public ResponseEntity<String> ingestPdf() {
        int count = etlService.ingestPdf(
            new ClassPathResource("docs/sample.pdf"), "sample-pdf");
        return ResponseEntity.ok("Ingested " + count + " chunks");
    }

    @PostMapping("/batch")
    public ResponseEntity<String> ingestBatch() {
        int count = etlService.ingestBatch(
            List.of(
                new ClassPathResource("docs/doc1.txt"),
                new ClassPathResource("docs/doc2.txt")),
            "batch");
        return ResponseEntity.ok("Ingested " + count + " chunks");
    }
}
```

---

## 7. Run + Test

```bash
export OPENAI_API_KEY=sk-...
mvn spring-boot:run

# Text ingestion
curl -X POST http://localhost:8080/api/etl/text

# PDF ingestion
curl -X POST http://localhost:8080/api/etl/pdf

# Batch ingestion
curl -X POST http://localhost:8080/api/etl/batch
```

---

## 8. Common Gotchas

### Gotcha 1 — No Header/Footer Removal

```text
Problem:
PDF headers pollute chunks

Fix:
ExtractedTextFormatter (top/bottom delete)
```

### Gotcha 2 — Chunk Too Large

```text
Problem:
Context blow + dilution

Fix:
500 tokens + overlap
```

### Gotcha 3 — No Metadata

```text
Problem:
No filtering / citations

Fix:
source + version + tenant
```

### Gotcha 4 — Non-Idempotent

```text
Problem:
Re-run = duplicates

Fix:
Deterministic IDs
```

---

## 9. Interview Questions

| Question | Answer |
|---|---|
| ETL pipeline? | Reader → Transformer → Writer. |
| Document class? | id + text + metadata + media. |
| Readers? | Text, PDF, Tika, Jsoup, JSON, Markdown. |
| TokenTextSplitter? | Token-based chunking (500 default). |
| Chunking strategies? | Fixed, recursive, semantic, structure-aware. |
| Metadata enrichment? | Keyword + Summary (LLM-based). |

### Scenario-Based

| Scenario | Solution |
|---|---|
| PDF ingestion | PagePdfDocumentReader |
| Multi-format | TikaDocumentReader |
| HTML | JsoupDocumentReader |
| Prose chunking | Recursive, 500 + 100 overlap |
| Multi-tenant | Metadata (tenant_id) |

---

## 10. Key Takeaways

```text
1. ETL = Reader → Transformer → Writer.

2. Document = id + text + metadata.

3. Readers = Text, PDF, Tika, Jsoup.

4. TokenTextSplitter = 500 tokens default.

5. Chunking = 500 + 100 overlap (default).

6. Metadata = tenant + version + source.

7. Enrichers = Keyword + Summary.

8. Batch = efficient (single add).

9. Idempotent = deterministic IDs.

10. Header/footer = remove (PDF).
```

---

## 11. Next Part Preview

**Part 8: Embeddings & Vector Stores Deep Dive**

Isme hum cover karenge:

```text
- EmbeddingModel deep dive
- Vector stores (pgvector, Milvus, Pinecone)
- Similarity search
- Metadata filtering
- Hybrid search
```