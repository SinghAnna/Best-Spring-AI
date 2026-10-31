# Part 26: Spring AI — Multi-Agent Systems (Supervisor + Worker Pattern)

> Multi-agent ka core — **`ToolCallingAdvisor` (recursive advisor, tool loop) + supervisor = ChatClient with specialists as tools (`@Tool` method → specialist client) + worker = isolated context (own system prompt + tools + memory namespace) + Agent Registry (Task tool, markdown config) + orchestrator-workers (plan → dispatch → synthesize) + evaluator-optimizer (generate → review → refine)** — specialists never see each other / supervisor reasoning, context pollution prevent, multi-model routing (expensive reasoning → cheap generation). [2185][2483][2489][2311]

---

## 1. Multi-Agent Mental Model

```text
Problem:
Single agent = Too many tools + long context
= Confusion + cost + latency

Solution:
Multi-agent:
1. Supervisor (orchestrator)
2. Workers (specialists)
3. Isolated context
4. Task delegation
```

### Single vs Multi-Agent

| Aspect | Single | Multi-Agent |
|---|---|---|
| Tools | Many (10+) | Few per agent (3-5) |
| Context | Long (all history) | Isolated (per task) |
| Cost | High (big context) | Lower (focused) |
| Latency | High | Lower (parallel) |
| Accuracy | Lower (confusion) | Higher (specialized) |

Single vs multi-agent — tools + context + cost + latency + accuracy. [2483][2489]

### Multi-Agent Patterns

| Pattern | Flow | When |
|---|---|---|
| Supervisor + Worker | Supervisor → Worker → Result | Clear domains |
| Orchestrator-Workers | Plan → Dispatch → Synthesize | Complex task |
| Evaluator-Optimizer | Generate → Review → Refine | Quality critical |
| Routing | Classify → Route | Simple dispatch |

Multi-agent patterns — supervisor + orchestrator + evaluator + routing. [2311][2484][2485]

---

## 2. Tool Calling (Foundation)

```text
Tool calling = Model → Tool → Result → Model
= Loop until done
```

### Tool Calling Loop

```text
1. Model → Tool call request
2. App → Execute tool
3. App → Result back to model
4. Model → Final response (or another tool call)
5. Repeat until no tool calls
```

Tool calling loop — model → tool → result → model (repeat). [2185][2293]

### ToolCallingAdvisor (Recursive)

```text
ToolCallingAdvisor = Recursive advisor
= Re-enter chain each iteration
= Auto-registered by ChatClient
```

`ToolCallingAdvisor` — recursive advisor (re-enter chain). [2185]

### Tool Definition

```java
@Tool(description = "Get order status by order ID")
public String getOrderStatus(
    @ToolParam(description = "Order ID") String orderId) {
    return orderService.getStatus(orderId);
}
```

Tool definition — `@Tool` + `@ToolParam` (name + description + schema). [2185][2293]

---

## 3. Supervisor + Worker Pattern

```text
Supervisor = ChatClient with specialists as tools
Worker = Isolated ChatClient (own prompt + tools + memory)
```

### Supervisor + Worker Flow

```text
1. User → Supervisor
2. Supervisor → Classify → Pick specialist
3. Supervisor → Call specialist (as tool)
4. Specialist → Isolated context → Answer
5. Supervisor → Combine → User
```

Supervisor + worker — classify → pick → call → isolated → combine. [2483][2489]

### Supervisor (Orchestrator)

```java
@Service
public class ShoppingSupervisor {

    private final ChatClient supervisorClient;
    private final ChatClient orderAgent;
    private final ChatClient productAgent;

    public ShoppingSupervisor(ChatClient.Builder builder,
                              ChatClient orderAgent,
                              ChatClient productAgent) {
        this.orderAgent = orderAgent;
        this.productAgent = productAgent;
        this.supervisorClient = builder
            .defaultSystem("""
                You are the supervisor of a shopping support team.
                You have two specialists: orderAgent, productAgent.
                Read the user's question, choose the one specialist
                that owns the topic, and call only that tool.
                Do not answer the question yourself.
                """)
            .defaultTools(this)
            .build();
    }

    @Tool(description = "Handle order questions: status, delivery, returns")
    public String delegateOrder(
        @ToolParam(description = "User's full question") String question) {
        return orderAgent.prompt()
            .user(question)
            .call()
            .content();
    }

    @Tool(description = "Handle product questions: search, price, stock")
    public String delegateProduct(
        @ToolParam(description = "User's full question") String question) {
        return productAgent.prompt()
            .user(question)
            .call()
            .content();
    }

    public String chat(String userMessage) {
        return supervisorClient.prompt()
            .user(userMessage)
            .call()
            .content();
    }
}
```

Supervisor — `@Tool` method → specialist client (delegate). [2489]

### Worker (Specialist)

```java
@Configuration
public class AgentConfig {

    @Bean
    public ChatClient orderAgent(ChatClient.Builder builder) {
        return builder
            .defaultSystem("""
                You are an order specialist.
                Only answer questions about orders, delivery, returns.
                Use the order tools to look up information.
                """)
            .defaultTools(new OrderTools())
            .build();
    }

    @Bean
    public ChatClient productAgent(ChatClient.Builder builder) {
        return builder
            .defaultSystem("""
                You are a product specialist.
                Only answer questions about products, price, stock.
                Use the product tools to look up information.
                """)
            .defaultTools(new ProductTools())
            .build();
    }
}
```

Worker — isolated ChatClient (own prompt + tools). [2483][2489]

### Isolation (Key)

```text
Worker = Isolated context
= Own system prompt + tools + memory namespace
= No supervisor history
= No other worker history
= Context pollution prevent
```

Isolation — own prompt + tools + memory (no cross-contamination). [2483][2489]

---

## 4. Agent Registry (Task Tool)

```text
Agent Registry = Catalog of subagents
= Name + description (Task tool)
```

### Agent Registry Flow

```text
1. Startup → Load subagent configs (markdown)
2. Registry → Name + description
3. Main agent → Task tool (delegate)
4. Task tool → Spawn subagent (isolated)
5. Result → Main agent (synthesize)
```

Agent registry — load → catalog → delegate → spawn → synthesize. [2483][2330]

### Agent Config (Markdown)

```markdown
# agent-order.md
name: order-agent
description: Handles order status, delivery, and return eligibility
tools: [getOrderStatus, getDeliveryInfo]
model: gpt-4o-mini
system_prompt: |
  You are an order specialist.
  Only answer questions about orders.
```

Agent config — markdown (name + description + tools + model + prompt). [2483][2330]

### Task Tool (Delegation)

```java
@Tool(description = "Delegate task to a specialized subagent")
public String task(
    @ToolParam(description = "Subagent name") String subagentType,
    @ToolParam(description = "Task description") String prompt) {
    // Spawn subagent (isolated context)
    // Execute → Return result
    return subagentRegistry.execute(subagentType, prompt);
}
```

Task tool — delegate (subagent name + task). [2483][2330]

---

## 5. Orchestrator-Workers Pattern

```text
Orchestrator-Workers = Plan → Dispatch → Synthesize
= Complex task → Sub-tasks → Workers
```

### Orchestrator-Workers Flow

```text
1. User → Orchestrator (plan)
2. Orchestrator → Sub-tasks (N)
3. Orchestrator → Dispatch (workers, parallel)
4. Workers → Results
5. Orchestrator → Synthesize → User
```

Orchestrator-workers — plan → dispatch → synthesize. [2311]

### Orchestrator-Workers Implementation

```java
@Service
public class OrchestratorService {

    private final ChatClient orchestrator;
    private final List<ChatClient> workers;

    public OrchestratorService(ChatClient orchestrator, List<ChatClient> workers) {
        this.orchestrator = orchestrator;
        this.workers = workers;
    }

    public String execute(String task) {
        // 1. Plan (sub-tasks)
        List<String> subTasks = plan(task);

        // 2. Dispatch (parallel)
        List<String> results = subTasks.parallelStream()
            .map(sub -> workers.get(0).prompt().user(sub).call().content())
            .toList();

        // 3. Synthesize
        return synthesize(task, results);
    }

    private List<String> plan(String task) {
        String plan = orchestrator.prompt()
            .user("Break this into sub-tasks: " + task)
            .call()
            .content();
        return List.of(plan.split("\n"));
    }

    private String synthesize(String task, List<String> results) {
        return orchestrator.prompt()
            .user("Combine results for: " + task + "\n\n" + String.join("\n", results))
            .call()
            .content();
    }
}
```

Orchestrator-workers — plan → dispatch (parallel) → synthesize. [2311]

---

## 6. Evaluator-Optimizer Pattern

```text
Evaluator-Optimizer = Generate → Review → Refine
= Quality critical
```

### Evaluator-Optimizer Flow

```text
1. Generator → Draft
2. Evaluator → Review (pass / feedback)
3. If fail → Refine (feedback) → Draft
4. Repeat until pass
```

Evaluator-optimizer — generate → review → refine (loop). [2311]

### Evaluator-Optimizer Implementation

```java
@Service
public class EvaluatorOptimizerService {

    private final ChatClient generator;
    private final ChatClient evaluator;

    public EvaluatorOptimizerService(ChatClient generator, ChatClient evaluator) {
        this.generator = generator;
        this.evaluator = evaluator;
    }

    public String generateWithReview(String task) {
        String draft = generator.prompt().user(task).call().content();
        
        for (int i = 0; i < 3; i++) {
            String review = evaluator.prompt()
                .user("Review this: " + draft)
                .call()
                .content();
            
            if (review.contains("PASS")) {
                return draft;
            }
            
            draft = generator.prompt()
                .user(task + "\n\nFeedback: " + review + "\n\nRefine:")
                .call()
                .content();
        }
        return draft;
    }
}
```

Evaluator-optimizer — generate → review → refine (loop, max 3). [2311]

---

## 7. Full Working Example

### pom.xml (Dependencies)

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId>
</dependency>
```

### application.yml

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
```

### AgentConfig

```java
package com.example.multiagent.config;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AgentConfig {

    @Bean
    public ChatClient orderAgent(ChatClient.Builder builder) {
        return builder
            .defaultSystem("""
                You are an order specialist.
                Only answer questions about orders, delivery, returns.
                """)
            .build();
    }

    @Bean
    public ChatClient productAgent(ChatClient.Builder builder) {
        return builder
            .defaultSystem("""
                You are a product specialist.
                Only answer questions about products, price, stock.
                """)
            .build();
    }
}
```

### ShoppingSupervisor

```java
package com.example.multiagent.service;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.tool.annotation.Tool;
import org.springframework.ai.tool.annotation.ToolParam;
import org.springframework.stereotype.Service;

@Service
public class ShoppingSupervisor {

    private final ChatClient supervisorClient;
    private final ChatClient orderAgent;
    private final ChatClient productAgent;

    public ShoppingSupervisor(ChatClient.Builder builder,
                              ChatClient orderAgent,
                              ChatClient productAgent) {
        this.orderAgent = orderAgent;
        this.productAgent = productAgent;
        this.supervisorClient = builder
            .defaultSystem("""
                You are the supervisor of a shopping support team.
                You have two specialists: orderAgent, productAgent.
                Read the user's question, choose the one specialist
                that owns the topic, and call only that tool.
                Do not answer the question yourself.
                """)
            .defaultTools(this)
            .build();
    }

    @Tool(description = "Handle order questions: status, delivery, returns")
    public String delegateOrder(
        @ToolParam(description = "User's full question") String question) {
        return orderAgent.prompt()
            .user(question)
            .call()
            .content();
    }

    @Tool(description = "Handle product questions: search, price, stock")
    public String delegateProduct(
        @ToolParam(description = "User's full question") String question) {
        return productAgent.prompt()
            .user(question)
            .call()
            .content();
    }

    public String chat(String userMessage) {
        return supervisorClient.prompt()
            .user(userMessage)
            .call()
            .content();
    }
}
```

### MultiAgentController

```java
package com.example.multiagent.controller;

import com.example.multiagent.service.ShoppingSupervisor;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/chat")
public class MultiAgentController {

    private final ShoppingSupervisor supervisor;

    public MultiAgentController(ShoppingSupervisor supervisor) {
        this.supervisor = supervisor;
    }

    public record ChatRequest(String message) {}
    public record ChatResponse(String reply) {}

    @PostMapping
    public ChatResponse chat(@RequestBody ChatRequest req) {
        return new ChatResponse(supervisor.chat(req.message()));
    }
}
```

---

## 8. Run + Test

```bash
export OPENAI_API_KEY=sk-...
mvn spring-boot:run

# Order question (supervisor → orderAgent)
curl -X POST http://localhost:8080/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"What is the status of order 12345?"}'

# Product question (supervisor → productAgent)
curl -X POST http://localhost:8080/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"What is the price of iPhone 15?"}'
```

---

## 9. Common Gotchas

### Gotcha 1 — No Isolation

```text
Problem:
Context pollution (worker sees all)

Fix:
Isolated context (own prompt + tools + memory)
```

### Gotcha 2 — Too Many Agents

```text
Problem:
Overhead (latency + cost)

Fix:
Only when needed (clear domains)
```

### Gotcha 3 — No Loop Limit

```text
Problem:
Infinite loop (tool call)

Fix:
Max iterations (3-5)
```

### Gotcha 4 — Supervisor Answers

```text
Problem:
Supervisor does work (not delegate)

Fix:
System prompt (delegate only)
```

---

## 10. Interview Questions

| Question | Answer |
|---|---|
| Multi-agent kya? | Supervisor + workers (specialized). |
| Supervisor pattern? | Supervisor → specialist (as tool). |
| Worker isolation? | Own prompt + tools + memory. |
| Agent registry? | Catalog (name + description). |
| Orchestrator-workers? | Plan → dispatch → synthesize. |
| Evaluator-optimizer? | Generate → review → refine. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Clear domains | Supervisor + worker |
| Complex task | Orchestrator-workers |
| Quality critical | Evaluator-optimizer |
| Simple dispatch | Routing |
| Context pollution | Isolated context |

---

## 11. Key Takeaways

```text
1. Multi-agent = supervisor + workers.

2. Supervisor = ChatClient with specialists as tools.

3. Worker = isolated context (own prompt + tools + memory).

4. Tool calling = model → tool → result → model (loop).

5. Agent registry = catalog (name + description).

6. Task tool = delegate (spawn subagent).

7. Orchestrator-workers = plan → dispatch → synthesize.

8. Evaluator-optimizer = generate → review → refine.

9. Isolation = context pollution prevent.

10. Trade-off = accuracy vs overhead.
```

---

## 12. Next Part Preview

**Part 27: Streaming + SSE (Real-Time Responses)**

Isme hum cover karenge:

```text
- Streaming deep dive
- SSE (Server-Sent Events)
- Flux<String> (reactive)
- Token-by-token streaming
- Frontend integration
```