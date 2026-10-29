# Spring AI: Production-Oriented Complete Guide

> Spring AI ka core — **ChatClient (unified LLM API) + RAG (QuestionAnswerAdvisor + VectorStore) + Tool Calling (@Tool/MCP) + Advisors (middleware) + Observability (Micrometer/OTel)** — production patterns (circuit breaker, token budget, PII scrubbing, evaluation) ke saath. [2152][2154][2157][2163]

---

## 📋 Tutorial Overview

Ye tutorial **Spring AI ko production-grade level** tak cover karta hai — theory + full working code + real-world patterns + interview prep. Har part me:

```text
- Theory (concept + mental model)
- Full code (runnable, production-ready)
- Best practices
- Common gotchas
- Interview questions
- Real-world patterns
```

### Target Audience

```text
- Java/Spring developers AI integrate karna chahte hain
- Backend engineers LLM-based features build karne wale
- Architects production AI systems design karne wale
- Interview prep (AI/ML + Spring)
```

### Prerequisites

```text
- Java 17+ / Spring Boot 3.x
- Maven / Gradle
- Basic REST API knowledge
- Basic AI/LLM concepts (helpful, not mandatory)
```

---

## 🎯 What We'll Cover

### Module 1: Spring AI Fundamentals

| Part | Topic | Key Concepts |
|---|---|---|
| 1 | Spring AI Introduction & Setup | What is Spring AI, starters, config, first app |
| 2 | ChatClient Deep Dive | Fluent API, prompt, call/stream, options |
| 3 | Chat Models & Providers | OpenAI, Anthropic, Ollama, Bedrock, Azure |
| 4 | Prompts & Prompt Templates | System/user prompts, templating, variables |
| 5 | Structured Output | Records/POJOs, JSON schema, typed responses |

**Outcome:** LLM calls kar sakte ho, structured responses le sakte ho, multiple providers switch kar sakte ho.

---

### Module 2: RAG (Retrieval-Augmented Generation)

| Part | Topic | Key Concepts |
|---|---|---|
| 6 | RAG Fundamentals | What is RAG, why, architecture, ETL pipeline |
| 7 | Document Loading & ETL | Readers (PDF/HTML/Text), transformers, chunking |
| 8 | Embeddings & Vector Stores | EmbeddingModel, VectorStore interface, pgvector/Milvus/Pinecone |
| 9 | QuestionAnswerAdvisor | RAG one-liner, retrieval + prompt assembly + citations |
| 10 | Advanced RAG | Reranking, hybrid search, similarity gating, multi-tenant |

**Outcome:** Apne documents pe Q&A chatbot bana sakte ho, production-grade RAG pipeline.

---

### Module 3: Tool Calling & Agents

| Part | Topic | Key Concepts |
|---|---|---|
| 11 | Function Calling Basics | @Tool annotation, JSON schema, tool loop |
| 12 | Advanced Tool Calling | Complex params, records, error handling |
| 13 | MCP (Model Context Protocol) | Client + server, @McpTool, transport |
| 14 | Agentic Patterns | Multi-step agents, planning, reflection |
| 15 | Real-World Agent | Order lookup + inventory + email agent |

**Outcome:** LLM apne Spring beans call kar sakta hai, MCP integrate kar sakte ho, agentic systems bana sakte ho.

---

### Module 4: Memory & Conversation

| Part | Topic | Key Concepts |
|---|---|---|
| 16 | Chat Memory | MessageChatMemoryAdvisor, conversation history |
| 17 | Persistent Memory | JDBC/Redis memory, multi-session |
| 18 | Long-Term Memory | VectorStoreChatMemoryAdvisor, semantic recall |

**Outcome:** Multi-turn conversations, persistent context, long-term recall.

---

### Module 5: Production Patterns

| Part | Topic | Key Concepts |
|---|---|---|
| 19 | Observability | Micrometer, OTel, traces, token metrics |
| 20 | Cost Management | Token budget, caching, model routing |
| 21 | Resilience | Circuit breaker, fallback chain, retry |
| 22 | Security | PII scrubbing, prompt injection guard, output validation |
| 23 | Evaluation | RAGAS, golden dataset, recall@k, regression testing |
| 24 | Testing | Unit (mock), integration (Testcontainers), eval suite |

**Outcome:** Production-ready — observable, cost-controlled, resilient, secure, evaluated.

---

### Module 6: Advanced Topics

| Part | Topic | Key Concepts |
|---|---|---|
| 25 | Multimodality | Image, audio, vision models |
| 26 | Streaming | Reactive streams, SSE, real-time |
| 27 | Embedding Models | Custom embeddings, dimension, batching |
| 28 | Vector Store Deep Dive | pgvector, Milvus, Pinecone, Chroma comparison |
| 29 | Advanced Advisors | Custom advisor, SafeGuard, PII redaction |
| 30 | Model Comparison | GPT-4 vs Claude vs Llama trade-offs |

**Outcome:** Multimodal, streaming, custom embeddings, advanced advisors, model selection.

---

### Module 7: Real-World Projects

| Part | Topic | Key Concepts |
|---|---|---|
| 31 | Project 1: Docs Q&A Chatbot | RAG + pgvector + citations |
| 32 | Project 2: Customer Support Agent | Tool calling + memory + escalation |
| 33 | Project 3: Code Review Assistant | Structured output + multi-model |
| 34 | Project 4: Multi-Agent System | Planner + worker + critic |
| 35 | Project 5: Enterprise RAG | Multi-tenant + hybrid search + eval |

**Outcome:** 5 production-grade projects, portfolio-ready.

---

### Module 8: Interview & Career

| Part | Topic | Key Concepts |
|---|---|---|
| 36 | Interview Prep | System design, behavioral, coding |
| 37 | Case Studies | Real-world AI incidents + lessons |
| 38 | Career Path | AI engineer roles, skills, salary |

**Outcome:** Interview-ready, career roadmap.

---

## 📊 Coverage Summary

```text
Total Parts: 38
Modules: 8
Projects: 5
Interview Prep: Yes
Production Patterns: Yes
Full Code: Yes (every part)
Theory: Yes (every part)
```

### Tech Stack

```text
- Java 17+ / Spring Boot 3.x
- Spring AI 1.0+
- Maven / Gradle
- PostgreSQL + pgvector (vector store)
- Redis (cache + memory)
- Micrometer + OpenTelemetry (observability)
- Testcontainers (testing)
- Docker (deployment)
```

### LLM Providers Covered

```text
- OpenAI (GPT-4, GPT-4o)
- Anthropic (Claude)
- Ollama (local, Llama)
- AWS Bedrock
- Azure OpenAI
```

---

## 🏗️ Learning Path

```text
Beginner (Part 1-5):
Fundamentals + ChatClient + Prompts + Structured Output

Intermediate (Part 6-18):
RAG + Tool Calling + MCP + Memory

Advanced (Part 19-30):
Production Patterns + Multimodal + Streaming + Advanced Advisors

Expert (Part 31-38):
Projects + Interview + Career
```

### Recommended Pace

```text
- Beginner: 1-2 parts/day (1 week)
- Intermediate: 1 part/day (2 weeks)
- Advanced: 1 part/2 days (3 weeks)
- Expert: 1 project/week (5 weeks) + interview prep
Total: ~8-10 weeks (part-time)
```

---

## 📁 Repository Structure

```text
spring-ai-tutorial/
├── docs/
│   ├── README.md (this file)
│   ├── 01-introduction.md
│   ├── 02-chatclient.md
│   ├── ...
│   └── 38-career.md
├── code/
│   ├── 01-hello-ai/
│   ├── 02-chatclient-demo/
│   ├── ...
│   ├── 31-docs-qa-chatbot/
│   └── ...
├── docker-compose.yml
└── pom.xml (parent)
```

---

## 🚀 Quick Start

```bash
# Clone / create project
spring init --dependencies=web,actuator spring-ai-demo

# Add Spring AI BOM + OpenAI starter
# (Part 1 me detail)

# Run
mvn spring-boot:run
```

---

## 📚 Key Spring AI Concepts (Preview)

| Concept | What | Why |
|---|---|---|
| ChatClient | Fluent API for LLM calls | Unified across providers |
| ChatModel | Provider abstraction | OpenAI/Anthropic/Ollama swap |
| Prompt | Input to LLM | System + user + context |
| Advisor | Middleware around LLM call | RAG, memory, safety |
| VectorStore | Embedding storage + search | RAG retrieval |
| EmbeddingModel | Text → vector | Semantic search |
| @Tool | Java method as LLM tool | Function calling |
| MCP | Model Context Protocol | Standard tool protocol |
| Structured Output | Typed Java object from LLM | No manual parsing |
| Observability | Metrics + traces | Production visibility |

Spring AI core — ChatClient, ChatModel, Prompt, Advisor, VectorStore, EmbeddingModel, @Tool, MCP, structured output, observability. [2154][2157][2160][2163]

---

## 🎓 Learning Outcomes

By the end, you'll be able to:

```text
✅ Build production-grade RAG applications
✅ Integrate LLMs with Spring Boot (multiple providers)
✅ Implement tool calling + MCP agents
✅ Add memory + long-term recall
✅ Add observability + cost control
✅ Secure AI apps (PII, prompt injection)
✅ Evaluate + test AI systems
✅ Deploy production-grade AI services
✅ Crack AI/Spring interviews
```

---

## 📖 How to Use This Tutorial

```text
1. Read theory (mental model)
2. Run code (hands-on)
3. Modify + experiment
4. Build project (module end)
5. Review interview questions
6. Move to next part
```

---

## 🔗 Useful Links

```text
- Spring AI Docs: [https://docs.spring.io/spring-ai/reference/](https://docs.spring.io/spring-ai/reference/)
- Spring AI GitHub: [https://github.com/spring-projects/spring-ai](https://github.com/spring-projects/spring-ai)
- MCP Spec: [https://modelcontextprotocol.io/](https://modelcontextprotocol.io/)
- OWASP LLM Top 10: [https://owasp.org/www-project-top-10-for-large-language-model-applications/](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
```

---

## 📝 Next Step

**Part 1: Spring AI Introduction & Setup** — start karne ke liye bolo:

```text
"Spring AI Part 1 start karo"
```

Main `docs/01-introduction.md` + `code/01-hello-ai/` complete bana dunga — theory + full runnable code + setup + first LLM call.

---

Spring AI ka core — **ChatClient (unified LLM API) + RAG (QuestionAnswerAdvisor + VectorStore) + Tool Calling (@Tool/MCP) + Advisors (middleware) + Observability (Micrometer/OTel)** — production patterns (circuit breaker, token budget, PII scrubbing, evaluation) ke saath. [2152][2154][2157][2163]

