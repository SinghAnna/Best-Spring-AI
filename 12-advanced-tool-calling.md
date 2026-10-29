# Part 12: Spring AI — Advanced Tool Calling & MCP

> Advanced tool calling ka core — **complex params (nested records) + tool validation + MCP (Model Context Protocol — tools decouple, dynamic discovery) + agentic patterns (chaining, parallelization, routing, orchestrator-workers, evaluator-optimizer)** — MCP me tools alag service me, client dynamically discover karta hai (no restart). [2305][2306][2311][2315]

---

## 1. Advanced Tool Calling Mental Model

```text
Basic (Part 11):
@Tool + simple params + single service

Advanced:
1. Complex params (nested records)
2. Tool validation
3. MCP (tools decouple)
4. Agentic patterns (multi-agent)
```

### Why Advanced

```text
Basic:
- Simple params (String, int)
- Tools same service me
- Single agent

Advanced:
- Nested records (Order + Items)
- Validation (schema + business)
- MCP (tools alag service)
- Multi-agent (planner + worker + critic)
```

Advanced — complex params, validation, MCP, multi-agent. [2306][2311][2315]

---

## 2. Complex Params (Nested Records)

```text
Nested = Record inside Record
= JSON Schema auto-generate (nested)
```

### Complex Tool Example

```java
public record OrderRequest(
    String orderId,
    Customer customer,
    List<Item> items
) {
    record Customer(String name, String email) {}
    record Item(String productId, int quantity, double price) {}
}

@Tool(description = "Create a new order", inputType = OrderRequest.class)
public String createOrder(OrderRequest request) {
    return "Order created: " + request.orderId() + 
           " for " + request.customer().name();
}
```

Nested records — JSON Schema auto-generate (nested). [2293][2315]

### inputType (Explicit Schema)

```text
inputType = Explicit schema (complex input)
= Spring AI schema generate
```

`inputType` — explicit schema (complex input). [2293][2298]

---

## 3. Tool Validation

```text
Validation = Input validate (before execute)
= Schema + business rules
```

### Validation Layers

```text
1. Schema validation (JSON Schema)
2. Bean Validation (@NotNull, @Size)
3. Business validation (custom)
```

Validation — schema + bean + business (3 layers). [2308][2315]

### Bean Validation Example

```java
public record OrderRequest(
    @NotBlank String orderId,
    @Email String customerEmail,
    @NotEmpty List<Item> items
) {}

@Tool(description = "Create order", inputType = OrderRequest.class)
public String createOrder(@Valid OrderRequest request) {
    // Validation automatic
}
```

Bean Validation — `@NotBlank`, `@Email`, `@NotEmpty` (auto). [2308]

### Business Validation

```java
@Tool(description = "Transfer funds")
public String transferFunds(
    @ToolParam(description = "From account") String from,
    @ToolParam(description = "To account") String to,
    @ToolParam(description = "Amount") double amount) {
    
    if (amount > 10000) {
        return "Transfer limit exceeded (max $10,000)";
    }
    if (!accountService.exists(from)) {
        return "Account not found: " + from;
    }
    // Execute
}
```

Business validation — custom rules (limit, existence). [2308]

---

## 4. MCP (Model Context Protocol)

```text
MCP = Standard protocol (tools decouple)
= Client + Server (separate services)
= Dynamic discovery (no restart)
```

### MCP Architecture

```text
User
 └─> AI Chat Service (:8081)
      └─> ChatClient (Spring AI)
      └─> MCP Client
      └─> MCP Tool Server (:8080)  ← POST /mcp
           └─> @McpTool methods
```

MCP — client + server (separate), dynamic discovery. [2309][2315]

### MCP vs Local Tools

| Aspect | Local Tools | MCP |
|---|---|---|
| Location | Same service | Separate service |
| Discovery | Static (compile) | Dynamic (runtime) |
| Restart | Required (new tool) | Not required |
| Coupling | Tight | Loose |
| Scaling | Same service | Independent |

MCP — decouple + dynamic + no restart. [2309][2315]

### MCP Server (@McpTool)

```java
@Service
public class OrderTool {

    @Tool(description = "Get order status by ID")
    public Map<String, Object> getOrderStatus(
        @ToolParam(description = "Order ID") String orderId) {
        return Map.of(
            "orderId", orderId,
            "status", "Shipped",
            "eta", "2026-10-30"
        );
    }
}
```

`@Tool` — MCP server me method → tool. [2315]

### MCP Client (Dynamic Discovery)

```java
@Bean
ChatClient chatClient(ChatModel chatModel,
                      SyncMcpToolCallbackProvider toolCallbackProvider) {
    return ChatClient.builder(chatModel)
        .defaultTools(toolCallbackProvider)  // Auto-discover
        .build();
}
```

`SyncMcpToolCallbackProvider` — auto-discover (no restart). [2309][2315]

### MCP Config

```yaml
# Server
spring:
  ai:
    mcp:
      server:
        name: tool-server
        version: 1.0.0
        protocol: STREAMABLE

# Client
spring:
  ai:
    mcp:
      client:
        toolcallback:
          enabled: true
        connections:
          tool-server:
            url: ${MCP_SERVER_URL}/mcp
            transport: STREAMABLE_HTTP
```

MCP config — server (protocol) + client (toolcallback + connection). [2309][2315]

---

## 5. Agentic Patterns

```text
Agent = LLM + Tools + Loop
= Autonomous task completion
```

### 5 Core Patterns

| Pattern | Flow | Use Case |
|---|---|---|
| Chaining | Step1 → Step2 → Step3 | Sequential pipeline |
| Parallelization | Parallel(A, B, C) → Merge | Independent tasks |
| Routing | Classify → Route → Handler | Triage |
| Orchestrator-Workers | Plan → Workers → Synthesize | Complex task |
| Evaluator-Optimizer | Generate → Evaluate → Loop | Quality |

5 patterns — chaining, parallelization, routing, orchestrator-workers, evaluator-optimizer. [2306][2311][2316]

### Pattern 1: Chaining

```text
Step 1 (Extract) -> Step 2 (Analyze) -> Step 3 (Report)
```

Chaining — sequential (output → input). [2311][2316]

### Pattern 2: Parallelization

```text
Query -> Parallel(Analysis A, Analysis B, Analysis C) -> Merge
```

Parallelization — concurrent + merge. [2306][2311]

### Pattern 3: Routing

```text
Query -> Classify (billing/tech/sales) -> Route -> Handler
```

Routing — classify → route → handler. [2306][2311]

### Pattern 4: Orchestrator-Workers

```text
Task -> Planner (subtasks) -> Workers (parallel) -> Synthesizer
```

Orchestrator-Workers — plan → workers → synthesize. [2306][2311]

### Pattern 5: Evaluator-Optimizer

```text
Draft -> Evaluator (feedback) -> Loop -> Final
```

Evaluator-Optimizer — generate → evaluate → loop. [2306][2311][2316]

---

## 6. Full Working Example

### MCP Server (Tool Server)

#### pom.xml

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-server-webmvc</artifactId>
</dependency>
```

#### application.yml

```yaml
spring:
  ai:
    mcp:
      server:
        name: tool-server
        version: 1.0.0
        protocol: STREAMABLE
server:
  port: 8080
```

#### OrderTool

```java
package com.example.mcpserver.tools;

import org.springframework.ai.tool.annotation.Tool;
import org.springframework.ai.tool.annotation.ToolParam;
import org.springframework.stereotype.Service;

import java.util.Map;

@Service
public class OrderTool {

    @Tool(description = "Get order status by ID")
    public Map<String, Object> getOrderStatus(
        @ToolParam(description = "Order ID, e.g. ORD-12345") String orderId) {
        return Map.of(
            "orderId", orderId,
            "status", "Shipped",
            "eta", "2026-10-30"
        );
    }

    @Tool(description = "List all order IDs")
    public java.util.List<String> listOrders() {
        return java.util.List.of("ORD-12345", "ORD-12346");
    }
}
```

### MCP Client (AI Chat Service)

#### pom.xml

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-client</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId>
</dependency>
```

#### application.yml

```yaml
spring:
  ai:
    mcp:
      client:
        toolcallback:
          enabled: true
        connections:
          tool-server:
            url: http://localhost:8080/mcp
            transport: STREAMABLE_HTTP
    openai:
      api-key: ${OPENAI_API_KEY}
server:
  port: 8081
```

#### ChatConfig

```java
package com.example.mcpclient.config;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.model.ChatModel;
import org.springframework.ai.mcp.SyncMcpToolCallbackProvider;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ChatConfig {

    @Bean
    ChatClient chatClient(ChatModel chatModel,
                          SyncMcpToolCallbackProvider toolCallbackProvider) {
        return ChatClient.builder(chatModel)
            .defaultTools(toolCallbackProvider)
            .build();
    }
}
```

#### ChatController

```java
package com.example.mcpclient.controller;

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

---

## 7. Run + Test

```bash
# 1. Start MCP Server (port 8080)
cd mcp-server && mvn spring-boot:run

# 2. Start MCP Client (port 8081)
cd mcp-client && mvn spring-boot:run

# 3. Test (client -> server -> tool -> LLM)
curl -X POST http://localhost:8081/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"What is the status of order ORD-12345?"}'

# Response:
{"reply":"Order ORD-12345 is Shipped, arriving 2026-10-30."}
```

---

## 8. Common Gotchas

### Gotcha 1 — No Validation

```text
Problem:
Invalid input = crash

Fix:
Schema + Bean + Business validation
```

### Gotcha 2 — MCP Not Enabled

```text
Problem:
Tools not discovered

Fix:
toolcallback.enabled=true
```

### Gotcha 3 — Sync/Async Mix

```text
Problem:
MCP client mix = error

Fix:
SYNC ya ASYNC (ek hi)
```

### Gotcha 4 — Agent Without Guard

```text
Problem:
Infinite loop / runaway

Fix:
Max iterations + budget cap
```

---

## 9. Interview Questions

| Question | Answer |
|---|---|
| MCP kya? | Model Context Protocol (tools decouple). |
| MCP vs local tools? | Separate service vs same service. |
| MCP discovery? | Dynamic (no restart). |
| Agentic patterns? | Chaining, parallel, routing, orchestrator, evaluator. |
| Orchestrator-workers? | Plan → workers → synthesize. |
| Evaluator-optimizer? | Generate → evaluate → loop. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Tools decouple | MCP |
| Dynamic discovery | MCP client |
| Complex task | Orchestrator-workers |
| Quality | Evaluator-optimizer |
| Triage | Routing |

---

## 10. Key Takeaways

```text
1. Complex params = nested records.

2. Validation = schema + bean + business.

3. MCP = tools decouple (client + server).

4. MCP discovery = dynamic (no restart).

5. Agentic patterns = 5 core.

6. Chaining = sequential.

7. Parallelization = concurrent + merge.

8. Routing = classify → route.

9. Orchestrator-workers = plan → workers.

10. Evaluator-optimizer = generate → evaluate.
```

---

## 11. Next Part Preview

**Part 13: Agentic Patterns (Deep Dive)**

Isme hum cover karenge:

```text
- Agentic patterns deep dive
- ReAct pattern
- Multi-agent orchestration
- Agent state management
- Real-world agent
```