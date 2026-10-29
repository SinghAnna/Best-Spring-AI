# Part 13: Spring AI — Agentic Patterns (Deep Dive)

> Agentic patterns ka core — **Workflow (predefined code paths) vs Agent (LLM self-directed) + ReAct (Reason + Act + Observe) + Orchestrator-Workers (plan → parallel workers → synthesize) + Evaluator-Optimizer (generate → evaluate → loop)** — `ChatMemory` + `MessageChatMemoryAdvisor` (conversation ID se scoped) + `ToolContext` (per-user state). [2317][2318][2321][2323]

---

## 1. Agent Mental Model

```text
Workflow = Predefined code path (LLM steps follow)
Agent    = LLM self-directed (tools + loop)

Workflow:
Code decides flow
LLM = step executor

Agent:
LLM decides flow
Tools = capabilities
Loop = iterate until done
```

### Workflow vs Agent

| Aspect | Workflow | Agent |
|---|---|---|
| Flow | Code-defined | LLM-defined |
| Flexibility | Low | High |
| Predictability | High | Low |
| Use case | Simple pipelines | Complex tasks |

Workflow (code-defined) vs Agent (LLM-defined). [2318][2316]

### Agent Anatomy

```text
Agent = LLM + Tools + Memory + Loop
   |
   +--> LLM (reasoning)
   +--> Tools (actions)
   +--> Memory (context)
   +--> Loop (iterate)
```

Agent — LLM + tools + memory + loop. [2318][2321]

---

## 2. ReAct Pattern

```text
ReAct = Reason + Act + Observe
= Think -> Do -> See -> Repeat
```

### ReAct Loop

```text
1. Thought: "Mujhe weather chahiye"
2. Action: getWeather("Delhi")
3. Observation: "Sunny, 22°C"
4. Thought: "Ab answer de sakta hoon"
5. Answer: "Delhi me sunny, 22°C"
```

ReAct — Thought → Action → Observation → Repeat. [2327][2318]

### ReAct Implementation

```java
String answer = chatClient.prompt()
    .user("What's the weather in Delhi?")
    .tools(weatherTools)
    .call()
    .content();
```

Spring AI — ReAct loop auto (tool calling). [2318][2323]

### ReAct Benefits

```text
- Complex tasks (multi-step)
- Real-world data (tools)
- Explainable (thoughts)
- Adaptive (observe → adjust)
```

ReAct — complex, real-world, explainable, adaptive. [2327]

---

## 3. Orchestrator-Workers Pattern

```text
Orchestrator = Planner (subtasks)
Workers = Parallel (specialized)
Synthesizer = Merge (final)
```

### Orchestrator-Workers Flow

```text
Task: "Write a blog post on Spring AI"
   |
   v
Orchestrator (plan):
- Research Spring AI
- Write intro
- Write body
- Write conclusion
   |
   v
Workers (parallel):
- Researcher
- Writer
- Editor
   |
   v
Synthesizer (merge):
Final blog post
```

Orchestrator-Workers — plan → parallel workers → synthesize. [2318][2325][2328]

### Implementation

```java
@Service
public class OrchestratorWorkersWorkflow {

    private final ChatClient chatClient;

    public OrchestratorWorkersWorkflow(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    public record Subtask(String description) {}
    public record OrchestratorResponse(List<Subtask> subtasks) {}
    public record WorkerResponse(String result) {}

    public String process(String task) {
        // 1. Orchestrator: plan
        OrchestratorResponse plan = chatClient.prompt()
            .user("Break this task into subtasks: " + task)
            .call()
            .entity(OrchestratorResponse.class);

        // 2. Workers: parallel
        List<WorkerResponse> results = plan.subtasks().parallelStream()
            .map(subtask -> executeWorker(subtask.description()))
            .toList();

        // 3. Synthesizer: merge
        return chatClient.prompt()
            .user("Merge these results into final answer: " + results)
            .call()
            .content();
    }

    private WorkerResponse executeWorker(String subtask) {
        String result = chatClient.prompt()
            .user(subtask)
            .call()
            .content();
        return new WorkerResponse(result);
    }
}
```

Orchestrator-Workers — plan (entity) → parallel workers → synthesize. [2318][2325]

---

## 4. Evaluator-Optimizer Pattern

```text
Evaluator = Quality check
Optimizer = Improve
Loop = Until quality threshold
```

### Evaluator-Optimizer Flow

```text
1. Generate (draft)
2. Evaluate (feedback)
3. If quality < threshold -> Improve
4. Repeat
5. Final (quality pass)
```

Evaluator-Optimizer — generate → evaluate → loop. [2318][2316]

### Implementation

```java
public String generateWithQuality(String task) {
    String draft = generate(task);
    int iterations = 0;
    
    while (iterations < 3) {
        String feedback = evaluate(draft);
        if (feedback.contains("PASS")) {
            return draft;
        }
        draft = improve(draft, feedback);
        iterations++;
    }
    return draft;
}
```

Evaluator-Optimizer — loop (max 3) + quality threshold. [2318][2316]

---

## 5. Agent Memory (ChatMemory)

```text
Memory = Context across turns
= Conversation ID se scoped
```

### ChatMemory Types

| Type | Storage | Scope |
|---|---|---|
| InMemory | In-process | Session |
| JDBC | PostgreSQL | Persistent |
| Redis | Redis | Distributed |
| VectorStore | Vector DB | Long-term (semantic) |

ChatMemory — InMemory, JDBC, Redis, VectorStore. [2317][2321][2326]

### MessageChatMemoryAdvisor

```java
@Bean
public ChatClient chatClient(ChatModel chatModel, ChatMemory chatMemory) {
    return ChatClient.builder(chatModel)
        .defaultAdvisors(
            MessageChatMemoryAdvisor.builder(chatMemory).build())
        .build();
}
```

`MessageChatMemoryAdvisor` — history retrieve + inject + store (auto). [2317][2323]

### Conversation ID (Per-Request)

```java
String answer = chatClient.prompt()
    .user(userMessage)
    .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, conversationId))
    .call()
    .content();
```

Conversation ID — per-request (session scoping). [2317][2323]

### Conversation ID Security

```text
WRONG: Client se raw ID accept
= Session poisoning

RIGHT: Server-side session / auth context
```

Conversation ID — server-side (poisoning prevent). [2323]

---

## 6. ToolContext (Per-User State)

```text
ToolContext = Per-user state (cart, preferences)
= Conversation ID pass
```

### ToolContext Example

```java
chatClient.prompt()
    .user("Add to cart")
    .toolContext(Map.of("conversationId", conversationId))
    .call()
    .content();

@Tool(description = "Add item to cart")
public String addToCart(
    @ToolParam(description = "Product ID") String productId,
    ToolContext context) {
    String conversationId = (String) context.getContext().get("conversationId");
    // Per-user cart
}
```

`ToolContext` — per-user state (cart). [2323]

---

## 7. Full Working Example

### AgentConfig

```java
package com.example.agent.config;

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
    public ChatClient chatClient(
        ChatModel chatModel, 
        ChatMemory chatMemory,
        WeatherTools weatherTools,
        OrderTools orderTools) {
        return ChatClient.builder(chatModel)
            .defaultSystem("""
                You are a helpful assistant.
                Use tools when needed.
                Remember conversation context.
                """)
            .defaultAdvisors(
                MessageChatMemoryAdvisor.builder(chatMemory).build())
            .defaultTools(weatherTools, orderTools)
            .build();
    }
}
```

### AgentController

```java
package com.example.agent.controller;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.memory.ChatMemory;
import org.springframework.web.bind.annotation.*;

import java.util.UUID;

@RestController
@RequestMapping("/api/agent")
public class AgentController {

    private final ChatClient chatClient;

    public AgentController(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    public record ChatRequest(String message, String conversationId) {}
    public record ChatResponse(String reply, String conversationId) {}

    @PostMapping
    public ChatResponse chat(@RequestBody ChatRequest req) {
        String conversationId = req.conversationId() != null 
            ? req.conversationId() 
            : UUID.randomUUID().toString();

        String reply = chatClient.prompt()
            .user(req.message())
            .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, conversationId))
            .call()
            .content();

        return new ChatResponse(reply, conversationId);
    }
}
```

---

## 8. Run + Test

```bash
export OPENAI_API_KEY=sk-...
mvn spring-boot:run

# Turn 1 (new conversation)
curl -X POST http://localhost:8080/api/agent \
  -H "Content-Type: application/json" \
  -d '{"message":"What is the weather in Delhi?"}'

# Turn 2 (same conversation, memory)
curl -X POST http://localhost:8080/api/agent \
  -H "Content-Type: application/json" \
  -d '{"message":"What about tomorrow?","conversationId":"<from-turn-1>"}'
```

---

## 9. Common Gotchas

### Gotcha 1 — No Memory

```text
Problem:
Multi-turn fail

Fix:
MessageChatMemoryAdvisor
```

### Gotcha 2 — Client-Side Conversation ID

```text
Problem:
Session poisoning

Fix:
Server-side session / auth context
```

### Gotcha 3 — Shared Tool State

```text
Problem:
Users share cart

Fix:
ToolContext (conversation ID)
```

### Gotcha 4 — Agent Without Guard

```text
Problem:
Infinite loop

Fix:
Max iterations + budget cap
```

---

## 10. Interview Questions

| Question | Answer |
|---|---|
| Workflow vs Agent? | Code-defined vs LLM-defined. |
| ReAct kya? | Reason + Act + Observe. |
| Orchestrator-workers? | Plan → parallel workers → synthesize. |
| Evaluator-optimizer? | Generate → evaluate → loop. |
| ChatMemory? | Context across turns (conversation ID). |
| ToolContext? | Per-user state (cart). |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Multi-turn chat | MessageChatMemoryAdvisor |
| Complex task | Orchestrator-workers |
| Quality | Evaluator-optimizer |
| Per-user state | ToolContext |
| Infinite loop | Max iterations |

---

## 11. Key Takeaways

```text
1. Workflow = code-defined, Agent = LLM-defined.

2. ReAct = Reason + Act + Observe.

3. Orchestrator-workers = plan → workers → synthesize.

4. Evaluator-optimizer = generate → evaluate → loop.

5. ChatMemory = context (conversation ID).

6. MessageChatMemoryAdvisor = auto-manage.

7. Conversation ID = server-side.

8. ToolContext = per-user state.

9. Agent = LLM + tools + memory + loop.

10. Guard = max iterations + budget.
```

---

## 12. Next Part Preview

**Part 14: Real-World Agent (Order Lookup + Inventory + Email)**

Isme hum cover karenge:

```text
- Real-world agent
- Order lookup + inventory + email
- Multi-tool orchestration
- Production patterns
- Full working example
```