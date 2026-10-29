# Part 11: Spring AI — Function Calling Basics

> Function calling ka core — **`@Tool` annotation (method → tool) + `@ToolParam` (param description) + JSON schema (auto-generate) + tool loop (model → tool call → result → model → final answer)** — Spring AI `ToolCallingManager` loop handle karta hai, model kabhi bhi direct code touch nahi karta. [2292][2293][2300][2303]

---

## 1. Function Calling Mental Model

```text
Problem:
LLM sirf text generate karta hai
= Real-world actions nahi kar sakta
= Weather / DB / API access nahi

Solution:
Function Calling = LLM ko "tools" do
= LLM decide kare kaunsa tool call kare
= App execute kare
= Result LLM ko wapas
= LLM final answer
```

### Tool Calling Flow (6 Steps)

```text
1. Tools define (name + description + schema)
2. Model -> Tool call request (name + args)
3. App -> Tool execute (with args)
4. App -> Result -> Model
5. Model -> Final answer (with result context)
6. User -> Answer
```

Tool calling — 6 steps (define → call → execute → result → final). [2293][2298][2300]

### Key Insight

```text
Model = "Decision maker" (kaunsa tool, kya args)
App   = "Executor" (actual code run)
= Model kabhi bhi direct code touch nahi karta
```

Model = decision maker, App = executor (safe). [2293][2303]

---

## 2. @Tool Annotation

```text
@Tool = Method → Tool (declarative)
= Name + Description + JSON Schema (auto)
```

### Basic @Tool

```java
@Component
public class WeatherTools {

    @Tool(description = "Get current weather for a city")
    public String getWeather(
        @ToolParam(description = "City name, e.g. Amsterdam") String city) {
        return "Sunny, 22°C in " + city;
    }
}
```

`@Tool(description)` — model ko tool ka purpose batata hai, `@ToolParam` — param ka purpose. [2292][2295][2296]

### Tool Definition (3 Parts)

```text
1. Name    = Method name (getWeather)
2. Description = @Tool(description)
3. Schema  = JSON Schema (auto from signature)
```

Tool definition — name + description + schema (auto). [2293][2300]

### Why Description Critical

```text
Model description se decide karta hai:
- Kaunsa tool call kare
- Kab call kare
- Kya args pass kare

Bad description = Wrong tool / wrong args
```

Description — model ka decision driver (critical). [2295][2296][2298]

---

## 3. Tool Loop (Automatic)

```text
Spring AI loop handle karta hai
= Manual re-submit nahi karna
```

### Loop Flow

```text
User: "What's the weather in Amsterdam?"
   |
   v
ChatClient (tools: WeatherTools)
   |
   v
Model -> Tool call (getWeather, {city: "Amsterdam"})
   |
   v
ToolCallingManager -> getWeather("Amsterdam")
   |
   v
Result -> Model
   |
   v
Model -> "It's sunny, 22°C in Amsterdam"
   |
   v
User
```

Tool loop — model → tool → result → model → final (auto). [2293][2301][2303]

### Loop Bound

```text
Max iterations = Configurable
= Infinite loop prevent
```

Loop bound — max iterations (infinite prevent). [2304]

---

## 4. Tools Registration

```text
Tools = ChatClient ko pass karo
= .tools() method
```

### Per-Request Tools

```java
String response = chatClient.prompt()
    .user("What's the weather in Amsterdam?")
    .tools(new WeatherTools())
    .call()
    .content();
```

`.tools(instance)` — per-request registration. [2292][2297]

### Default Tools (Builder)

```java
@Bean
public ChatClient chatClient(ChatClient.Builder builder, WeatherTools weatherTools) {
    return builder
        .defaultTools(weatherTools)
        .build();
}
```

`defaultTools()` — har call me available. [2293][2302]

### ToolCallback (Low-Level)

```java
ToolCallback callback = FunctionToolCallback
    .builder("getWeather", (WeatherRequest req) -> weatherService.getWeather(req))
    .description("Get current weather")
    .inputType(WeatherRequest.class)
    .build();
```

`FunctionToolCallback` — programmatic (low-level). [2293][2294]

---

## 5. Complex Params (Records)

```text
Complex = Nested records / lists
= JSON Schema auto-generate
```

### Complex Tool Example

```java
public record OrderRequest(String orderId, String customerEmail) {}

@Tool(description = "Get order status by order ID")
public String getOrderStatus(
    @ToolParam(description = "Order ID, e.g. ORD-1039") String orderId) {
    return "Order " + orderId + " is shipped";
}

@Tool(description = "Search orders by customer")
public List<Order> searchOrders(
    @ToolParam(description = "Customer email") String email,
    @ToolParam(description = "Max results") int limit) {
    return orderService.search(email, limit);
}
```

Complex params — records + lists (schema auto). [2293][2298]

### inputType (Explicit Schema)

```java
@Tool(description = "Get weather", inputType = WeatherRequest.class)
public String getWeather(WeatherRequest request) {
    return weatherService.getWeather(request);
}

public record WeatherRequest(String city, String date) {}
```

`inputType` — explicit schema (complex input). [2293][2298]

---

## 6. Error Handling

```text
Tool error = LLM ko inform karo
= LLM retry / alternative
```

### Error Handling Example

```java
@Tool(description = "Get order status")
public String getOrderStatus(String orderId) {
    try {
        return orderService.getStatus(orderId);
    } catch (OrderNotFoundException e) {
        return "Order not found: " + orderId;
    }
}
```

Error — LLM ko string return (retry / alternative). [2298][2300]

### Parallel Tool Calling

```text
Parallel = Multiple tools ek saath
= OpenAI supports
```

Parallel — multiple tool calls ek response me. [2298]

---

## 7. Full Working Example

### WeatherTools

```java
package com.example.tools.tools;

import org.springframework.ai.tool.annotation.Tool;
import org.springframework.ai.tool.annotation.ToolParam;
import org.springframework.stereotype.Component;

import java.time.LocalDate;
import java.util.Random;

@Component
public class WeatherTools {

    private final Random random = new Random();

    @Tool(description = "Get current weather for a city. Use this for any weather-related question.")
    public String getWeather(
        @ToolParam(description = "City name, e.g. Amsterdam") String city) {
        String[] conditions = {"Sunny", "Cloudy", "Rainy", "Windy"};
        int temp = 15 + random.nextInt(15);
        return conditions[random.nextInt(conditions.length)] + ", " + temp + "°C in " + city;
    }

    @Tool(description = "Get today's date. Use this to resolve relative dates like 'tomorrow' or 'next week'.")
    public String getCurrentDate() {
        LocalDate today = LocalDate.now();
        return today.toString() + " (" + today.getDayOfWeek() + ")";
    }
}
```

### OrderTools

```java
package com.example.tools.tools;

import org.springframework.ai.tool.annotation.Tool;
import org.springframework.ai.tool.annotation.ToolParam;
import org.springframework.stereotype.Component;

import java.util.List;
import java.util.Map;

@Component
public class OrderTools {

    private static final Map<String, String> ORDERS = Map.of(
        "ORD-1039", "Shipped - arriving Oct 30",
        "ORD-1040", "Processing",
        "ORD-1041", "Delivered"
    );

    @Tool(description = "Get order status by order ID. Use this when user asks about an order.")
    public String getOrderStatus(
        @ToolParam(description = "Order ID, e.g. ORD-1039") String orderId) {
        return ORDERS.getOrDefault(orderId, "Order not found: " + orderId);
    }

    @Tool(description = "List all order IDs for a customer")
    public List<String> listOrders(
        @ToolParam(description = "Customer email") String email) {
        return List.copyOf(ORDERS.keySet());
    }
}
```

### ToolConfig

```java
package com.example.tools.config;

import com.example.tools.tools.OrderTools;
import com.example.tools.tools.WeatherTools;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ToolConfig {

    @Bean
    public ChatClient chatClient(
        ChatClient.Builder builder,
        WeatherTools weatherTools,
        OrderTools orderTools) {
        return builder
            .defaultSystem("""
                You are a helpful assistant.
                Use tools when needed to answer accurately.
                """)
            .defaultTools(weatherTools, orderTools)
            .build();
    }
}
```

### ToolController

```java
package com.example.tools.controller;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/tools")
public class ToolController {

    private final ChatClient chatClient;

    public ToolController(ChatClient chatClient) {
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

## 8. Run + Test

```bash
export OPENAI_API_KEY=sk-...
mvn spring-boot:run

# Weather (tool call)
curl -X POST http://localhost:8080/api/tools \
  -H "Content-Type: application/json" \
  -d '{"message":"What is the weather in Amsterdam?"}'

# Order (tool call)
curl -X POST http://localhost:8080/api/tools \
  -H "Content-Type: application/json" \
  -d '{"message":"What is the status of order ORD-1039?"}'

# Multi-tool (weather + date)
curl -X POST http://localhost:8080/api/tools \
  -H "Content-Type: application/json" \
  -d '{"message":"What is the weather today in Delhi?"}'
```

---

## 9. Common Gotchas

### Gotcha 1 — Vague Description

```text
Problem:
Wrong tool / wrong args

Fix:
Specific description + examples
```

### Gotcha 2 — No Error Handling

```text
Problem:
Tool fail = LLM confused

Fix:
Try-catch + string return
```

### Gotcha 3 — Too Many Tools

```text
Problem:
Model confused (selection)

Fix:
Relevant tools only (per-request)
```

### Gotcha 4 — Side Effects Without Guard

```text
Problem:
LLM delete / write kare

Fix:
Read-only tools + approval gate
```

---

## 10. Interview Questions

| Question | Answer |
|---|---|
| Function calling kya? | LLM ko tools do (real-world actions). |
| @Tool kya? | Method → tool (declarative). |
| Tool loop? | Model → tool → result → model → final. |
| Tool definition? | Name + description + schema. |
| Description kyun critical? | Model ka decision driver. |
| Parallel tool calling? | Multiple tools ek response me. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Weather / DB access | @Tool + ChatClient |
| Complex input | inputType (record) |
| Tool error | Try-catch + string |
| Side effects | Approval gate |
| Cost optimize | Relevant tools only |

---

## 11. Key Takeaways

```text
1. Function calling = LLM + tools.

2. @Tool = method → tool.

3. @ToolParam = param description.

4. JSON schema = auto-generate.

5. Tool loop = auto (ToolCallingManager).

6. Description = critical (decision driver).

7. Error = string return (LLM inform).

8. Parallel = multiple tools.

9. Side effects = approval gate.

10. Model = decision, App = executor.
```

---

## 12. Next Part Preview

**Part 12: Advanced Tool Calling**

Isme hum cover karenge:

```text
- Advanced tool calling
- Complex params (nested records)
- Tool validation
- MCP (Model Context Protocol)
- Agentic patterns
```