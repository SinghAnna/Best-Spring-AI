# Part 14: Spring AI — Real-World Agent (Order Lookup + Inventory + Email)

> Real-world agent ka core — **multi-tool orchestration (order lookup + inventory + email) + guardrails (input/output/action) + human-in-the-loop (high-impact actions pe approval) + tool-call cap + tenant context in `ToolContext`** — "propose, do not execute" pattern (writes/sends/charges = intent record, app/person execute). [2331][2336][2338][2341]

---

## 1. Real-World Agent Mental Model

```text
Agent = LLM + Tools + Memory + Guardrails
= "Do things" (not just chat)

Real-world agent:
- Order lookup (read)
- Inventory check (read)
- Email send (write — approval)
```

### Production Agent Anatomy

```text
User
 ↓
Authentication
 ↓
Authorization
 ↓
Agent (LLM + Tools)
 ↓
Guardrails (input + output + action)
 ↓
Tool Selection
 ↓
Input Validation
 ↓
Human Approval? (high-impact)
 ↓
Business Logic
 ↓
External API
 ↓
Audit + Metrics
```

Production agent — authN → authZ → agent → guardrails → validation → approval → execute → audit. [2338]

### Production Requirements

```text
1. Guardrails (input, output, action)
2. Authorization (app-enforced, not model)
3. Validation (tool args)
4. Observability (logs, metrics, traces)
5. Evaluation (repeatable tests)
6. Retries (transient errors)
7. Human approval (sensitive actions)
8. Auditability (all tool calls)
```

Production agent — 8 requirements (guardrails, authZ, validation, observability, eval, retries, approval, audit). [2338]

---

## 2. Multi-Tool Orchestration

```text
Multi-tool = Agent multiple tools call kare
= "Check order 1042 + list low stock"
= 2 tools, 1 answer
```

### Multi-Tool Flow

```text
User: "Check order 1042 and list products with stock under 5"
   |
   v
LLM plans:
- getOrderStatus(1042)
- listLowStockProducts(5)
   |
   v
Parallel / sequential tool calls
   |
   v
LLM synthesizes one answer
```

Multi-tool — LLM plans + calls + synthesizes. [2332][2339]

### Tool Design (2-3 Narrow Tools)

```text
Best practice:
- 2-3 narrow tools (not 10 broad)
- Clear descriptions
- Read-only first
- Write = approval
```

Tool design — 2-3 narrow, clear, read-only first. [2336][2339]

---

## 3. Guardrails (3 Types)

```text
Guardrail = Control (restrict / validate)
```

### Guardrail Types

| Type | Purpose | Examples |
|---|---|---|
| Input | Before agent | Prompt injection, out-of-scope, unsafe content |
| Reasoning | During | Unsafe tool combos, permission bypass, uncertainty → escalate |
| Output | Before user | Sensitive data, hallucination check |
| Action | Before execute | Confirmation, high-risk block, reversibility |

Guardrails — input, reasoning, output, action (4 types). [2333][2341]

### Guardrail Patterns

| Pattern | Flow | Use Case |
|---|---|---|
| Confirm-Then-Execute | Prepare → user approve → execute | Low-risk (scheduling) |
| Propose-Then-Review | Prepare → internal reviewer → execute | Sensitive (refunds) |
| Watch-and-Veto | Act → reversible window | Low-risk (fast) |

Guardrail patterns — confirm-then-execute, propose-then-review, watch-and-veto. [2341]

### Guardrail Implementation

```java
// Input guardrail
if (isPromptInjection(userMessage)) {
    return "Cannot process this request";
}

// Action guardrail
if (isHighRisk(toolCall)) {
    return requestApproval(toolCall);  // Human-in-loop
}

// Output guardrail
if (containsSensitiveData(response)) {
    return redact(response);
}
```

Guardrails — input (injection), action (approval), output (redact). [2333][2341]

---

## 4. Human-in-the-Loop (Approval)

```text
High-impact = Human approval
= Refund, email send, charge, delete
```

### Approval Flow

```text
Agent -> Tool Request
   |
   v
Risk Evaluation
   |
   +--> Low risk -> Execute
   +--> High risk -> Approval Required
                        |
                        v
                    Human Review
                        |
                        +--> Approve -> Execute
                        +--> Reject -> Deny
```

Approval — risk eval → high-risk → human → approve/reject. [2331][2338][2341]

### Approval Service

```java
@Service
public class ApprovalService {

    public ApprovalRequest requestApproval(String action, String details) {
        // Create pending approval
        return approvalRepo.save(new ApprovalRequest(action, details, "PENDING"));
    }

    public boolean isApproved(String approvalId) {
        return approvalRepo.findById(approvalId)
            .map(a -> a.getStatus().equals("APPROVED"))
            .orElse(false);
    }
}
```

Approval service — pending → human review → approve/reject. [2331][2338]

### Propose-Do-Not-Execute

```text
Agent = "Propose" (intent record)
App / Human = "Execute"

Example:
- Email send = Agent propose, app send
- Refund = Agent propose, human approve, app execute
```

Propose-do-not-execute — writes/sends/charges = intent record. [2336][2338]

---

## 5. Tool-Call Cap + Tenant Context

```text
Tool-call cap = Loop bound (cost + runaway prevent)
Tenant context = ToolContext (isolation)
```

### Tool-Call Cap

```yaml
spring:
  ai:
    tools:
      limits:
        max-calls-per-tool: 40
        max-calls-per-turn: 150
```

Tool-call cap — 40/tool, 150/turn (Spring AI 2.0.1+). [2336]

### Tenant Context (ToolContext)

```java
chatClient.prompt()
    .user(userMessage)
    .toolContext(Map.of(
        "tenantId", tenantId,
        "userId", userId))
    .call()
    .content();

@Tool(description = "Get orders")
public List<Order> getOrders(ToolContext context) {
    String tenantId = (String) context.getContext().get("tenantId");
    return orderService.findByTenant(tenantId);  // Isolated
}
```

Tenant context — `ToolContext` (model never sees tenant ID). [2336][2338]

### Why ToolContext (Not Prompt)

```text
Prompt me tenant ID = Model can leak / manipulate
ToolContext = Server-side (model never sees)
```

ToolContext — server-side (model never sees tenant). [2336]

---

## 6. Full Working Example

### StoreTools (Multi-Tool)

```java
package com.example.agent.tools;

import org.springframework.ai.tool.annotation.Tool;
import org.springframework.ai.tool.annotation.ToolParam;
import org.springframework.ai.chat.model.ToolContext;
import org.springframework.stereotype.Component;

import java.util.List;
import java.util.Map;

@Component
public class StoreTools {

    private static final Map<String, String> ORDERS = Map.of(
        "1042", "Shipped - arriving Oct 30",
        "1043", "Processing"
    );

    private static final Map<String, Integer> STOCK = Map.of(
        "Laptop", 3,
        "Mouse", 25,
        "Keyboard", 2
    );

    @Tool(description = "Get order status and customer name by order ID")
    public String getOrderStatus(
        @ToolParam(description = "Order ID, e.g. 1042") String orderId,
        ToolContext context) {
        String tenantId = (String) context.getContext().get("tenantId");
        return ORDERS.getOrDefault(orderId, "Order not found: " + orderId);
    }

    @Tool(description = "List product names with stock below the threshold")
    public List<String> listLowStockProducts(
        @ToolParam(description = "Stock threshold") int threshold,
        ToolContext context) {
        return STOCK.entrySet().stream()
            .filter(e -> e.getValue() < threshold)
            .map(Map.Entry::getKey)
            .toList();
    }

    @Tool(description = "Send email to customer. Requires approval for high-impact actions.")
    public String sendEmail(
        @ToolParam(description = "Customer email") String to,
        @ToolParam(description = "Subject") String subject,
        @ToolParam(description = "Body") String body,
        ToolContext context) {
        // Propose-Do-Not-Execute: intent record only
        return "Email intent recorded (pending approval): " + subject;
    }
}
```

### AgentConfig

```java
package com.example.agent.config;

import com.example.agent.tools.StoreTools;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.client.advisor.MessageChatMemoryAdvisor;
import org.springframework.ai.chat.memory.ChatMemory;
import org.springframework.ai.chat.memory.MessageWindowChatMemory;
import org.springframework.ai.chat.model.ChatModel;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AgentConfig {

    @Bean
    public ChatMemory chatMemory() {
        return MessageWindowChatMemory.builder()
            .maxMessages(20)
            .build();
    }

    @Bean
    public ChatClient agentChatClient(
        ChatModel chatModel, ChatMemory chatMemory, StoreTools storeTools) {
        return ChatClient.builder(chatModel)
            .defaultSystem("""
                You are a customer support agent for an e-commerce store.
                Use tools to look up orders and inventory.
                For email sending, always propose (never execute directly).
                Be concise and helpful.
                """)
            .defaultAdvisors(
                MessageChatMemoryAdvisor.builder(chatMemory).build())
            .defaultTools(storeTools)
            .build();
    }
}
```

### AgentController

```java
package com.example.agent.controller;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.model.ToolContext;
import org.springframework.web.bind.annotation.*;

import java.util.Map;
import java.util.UUID;

@RestController
@RequestMapping("/api/agent")
public class AgentController {

    private final ChatClient agentChatClient;

    public AgentController(ChatClient agentChatClient) {
        this.agentChatClient = agentChatClient;
    }

    public record AgentRequest(String message, String conversationId) {}
    public record AgentResponse(String answer, String conversationId) {}

    @PostMapping
    public AgentResponse run(@RequestBody AgentRequest req) {
        String conversationId = req.conversationId() != null
            ? req.conversationId()
            : UUID.randomUUID().toString();

        String answer = agentChatClient.prompt()
            .user(req.message())
            .toolContext(Map.of(
                "tenantId", "tenant-1",
                "userId", "user-123",
                "conversationId", conversationId))
            .call()
            .content();

        return new AgentResponse(answer, conversationId);
    }
}
```

---

## 7. Run + Test

```bash
export OPENAI_API_KEY=sk-...
mvn spring-boot:run

# Multi-tool (order + inventory)
curl -X POST http://localhost:8080/api/agent \
  -H "Content-Type: application/json" \
  -d '{"message":"Check order 1042 and list products with stock under 5"}'

# Email (propose, not execute)
curl -X POST http://localhost:8080/api/agent \
  -H "Content-Type: application/json" \
  -d '{"message":"Send email to customer about order 1042"}'
```

---

## 8. Common Gotchas

### Gotcha 1 — No Guardrails

```text
Problem:
Unsafe actions execute

Fix:
Input + output + action guardrails
```

### Gotcha 2 — Tenant in Prompt

```text
Problem:
Model leak / manipulate

Fix:
ToolContext (server-side)
```

### Gotcha 3 — Agent Executes Writes

```text
Problem:
Irreversible damage

Fix:
Propose-Do-Not-Execute
```

### Gotcha 4 — No Tool-Call Cap

```text
Problem:
Runaway / cost blow

Fix:
max-calls-per-tool + per-turn
```

---

## 9. Interview Questions

| Question | Answer |
|---|---|
| Real-world agent kya? | LLM + tools + memory + guardrails. |
| Multi-tool orchestration? | LLM plans + calls + synthesizes. |
| Guardrails types? | Input, reasoning, output, action. |
| Human-in-the-loop? | High-impact pe approval. |
| Tool-call cap? | 40/tool, 150/turn (Spring AI 2.0.1+). |
| ToolContext? | Per-user state (server-side). |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Order + inventory | Multi-tool orchestration |
| Email send | Propose-Do-Not-Execute |
| Refund | Human approval |
| Tenant isolation | ToolContext |
| Runaway prevent | Tool-call cap |

---

## 10. Key Takeaways

```text
1. Agent = LLM + tools + memory + guardrails.

2. Multi-tool = LLM plans + calls + synthesizes.

3. Guardrails = input + reasoning + output + action.

4. Human-in-loop = high-impact approval.

5. Propose-Do-Not-Execute = writes/sends/charges.

6. Tool-call cap = 40/tool, 150/turn.

7. ToolContext = server-side tenant.

8. 2-3 narrow tools = best practice.

9. Read-only first, write = approval.

10. Audit = all tool calls.
```

---

## 11. Next Part Preview

**Part 15: MCP Deep Dive (Model Context Protocol)**

Isme hum cover karenge:

```text
- MCP deep dive
- Client + server architecture
- Transport (STDIO, SSE, Streamable HTTP)
- Tool discovery
- Real-world MCP server
```