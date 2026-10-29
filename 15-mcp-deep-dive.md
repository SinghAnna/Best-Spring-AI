# Part 15: Spring AI — MCP Deep Dive (Model Context Protocol)

> MCP ka core — **open protocol (JSON-RPC 2.0) + Host/Clients/Servers (3 roles) + Tools/Resources/Prompts (3 primitives) + transports (STDIO / SSE / Streamable HTTP) + `SyncMcpToolCallbackProvider` (auto-discover, duplicate-name validation)** — client tool schemas locally define nahi karta, server se startup pe discover karta hai. [2342][2343][2344][2350]

---

## 1. MCP Mental Model

```text
MCP = Model Context Protocol
= Open standard (Anthropic, 2024)
= LLM apps <-> External data/tools

Problem:
N×M integration (N LLM apps × M tools)
= Custom code per pair

Solution:
MCP = Standard protocol
= 1 integration per side
```

### MCP = USB-C for AI

```text
USB-C = Universal port (devices <-> peripherals)
MCP   = Universal protocol (LLM apps <-> tools/data)

Before MCP:
ChatGPT -> Custom API
Claude  -> Custom API
Cursor  -> Custom API
= N×M

After MCP:
ChatGPT -> MCP -> Tools
Claude  -> MCP -> Tools
Cursor  -> MCP -> Tools
= N + M
```

MCP — N×M → N+M (standard protocol). [2343][2347]

### MCP Architecture (3 Roles)

```text
Host (LLM app: Claude Desktop, Cursor, IDE)
   |
   v
Client (connector in host)
   |
   v (JSON-RPC 2.0)
Server (service: tools/data provider)
```

MCP — Host (LLM app) + Client (connector) + Server (provider). [2343][2349]

---

## 2. MCP Primitives (3 Server Features)

```text
Server offers:
1. Tools     = Functions (model-controlled)
2. Resources = Data (app-controlled, read-only)
3. Prompts   = Templates (user-controlled)
```

### Primitives Comparison

| Primitive | What | Who Controls | Example |
|---|---|---|---|
| Tools | Functions (actions) | Model | `get_weather`, `create_order` |
| Resources | Data (read-only) | Application | `file:///docs/api.md` |
| Prompts | Templates | User | `/review-code` |

Tools (model), Resources (app), Prompts (user). [2343][2349][2351]

### Tools (Model-Controlled)

```text
Tools = LLM invoke kare (actions)
= name + description + input schema
```

```text
tools/list     -> Discover tools
tools/call     -> Execute tool
notifications/tools/list_changed -> Server notifies
```

Tools — `tools/list` + `tools/call` + `list_changed` notification. [2351]

### Resources (App-Controlled)

```text
Resources = Read-only data (URI-addressable)
= files, DB rows, API responses
```

```text
resources/list      -> List resources
resources/read      -> Read content
resources/subscribe -> Monitor changes
```

Resources — URI + MIME type + `resources/read`. [2349][2351]

### Prompts (User-Controlled)

```text
Prompts = Pre-built templates (user invokes)
= /review-code, /debug-issue
```

```text
prompts/list -> List prompts
prompts/get  -> Get prompt + args
```

Prompts — parameterized templates, user-invoked. [2349][2351]

---

## 3. MCP Transports

```text
Transport = How client <-> server communicate
```

### Transport Options

| Transport | Use Case | Mechanism |
|---|---|---|
| STDIO | Local processes | stdin/stdout (JSON-RPC) |
| SSE | HTTP streaming | HTTP POST + SSE |
| Streamable HTTP | Modern HTTP | HTTP POST + GET (SSE optional) |
| Stateless Streamable HTTP | Serverless | HTTP POST only |

Transports — STDIO (local), SSE (HTTP streaming), Streamable HTTP (modern), Stateless (serverless). [2344][2346][2349]

### Transport Selection

```text
Local tool (npm/CLI)?      -> STDIO
Remote HTTP service?       -> Streamable HTTP
Legacy SSE?                -> SSE
Serverless (Lambda)?       -> Stateless Streamable HTTP
```

Selection — STDIO (local), Streamable HTTP (remote), Stateless (serverless). [2344][2346]

### Streamable HTTP (Recommended)

```text
Streamable HTTP = SSE replacement
= HTTP POST + GET (SSE optional)
= Production recommended
```

Streamable HTTP — SSE replace, production recommended. [2344][2346]

---

## 4. Spring AI MCP Client

```text
Client = LLM app side
= Auto-discover tools from servers
```

### Client Starter

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-client</artifactId>
</dependency>
<!-- Or WebFlux (production recommended) -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-client-webflux</artifactId>
</dependency>
```

Client starter — STDIO + HTTP/SSE + Streamable HTTP. [2344]

### Client Config (Streamable HTTP)

```yaml
spring:
  ai:
    mcp:
      client:
        enabled: true
        name: spring-ai-mcp-client
        type: SYNC  # SYNC ya ASYNC (mix nahi)
        request-timeout: 20s
        toolcallback:
          enabled: true
        streamable-http:
          connections:
            tool-server:
              url: http://localhost:8080
              endpoint: /mcp
```

Client config — `type` (SYNC/ASYNC, mix nahi), `request-timeout`, `toolcallback.enabled`. [2344]

### Client Config (STDIO)

```yaml
spring:
  ai:
    mcp:
      client:
        stdio:
          connections:
            filesystem:
              command: npx
              args:
                - "-y"
                - "@modelcontextprotocol/server-filesystem"
                - "/tmp"
```

STDIO — command + args (local process). [2344][2350]

### SyncMcpToolCallbackProvider (Auto-Discover)

```java
@Bean
ChatClient chatClient(ChatModel chatModel,
                      SyncMcpToolCallbackProvider toolCallbackProvider) {
    return ChatClient.builder(chatModel)
        .defaultTools(toolCallbackProvider)  // Auto-discover
        .build();
}
```

`SyncMcpToolCallbackProvider` — multi-server tools discover + consolidate + duplicate validate. [2342][2345][2350]

### Discovery Flow

```text
1. Startup -> Config scan
2. MCP clients create (per server)
3. tools/list -> All servers
4. ToolCallback[] -> Consolidate
5. Duplicate name -> Validate (error)
6. ChatClient -> defaultTools(provider)
```

Discovery — startup → clients → tools/list → consolidate → validate. [2342][2345]

---

## 5. Spring AI MCP Server

```text
Server = Tools expose kare
= @Tool annotated beans
```

### Server Starter

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-server-webmvc</artifactId>
</dependency>
```

Server starter — `@Tool` beans scan + JSON schema + MCP handler. [2346][2350]

### Server Config

```yaml
spring:
  ai:
    mcp:
      server:
        name: tool-server
        version: 1.0.0
        protocol: STREAMABLE
        type: SYNC
        capabilities:
          tool: true
          resource: true
          prompt: true
server:
  port: 8080
```

Server config — name + version + protocol (STREAMABLE) + capabilities. [2346]

### Server Tool (@Tool)

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

Server `@Tool` — auto-register (schema + handler). [2346][2350]

### Server Flow

```text
1. Startup -> @Tool beans scan
2. JSON Schema generate
3. MCP handler register
4. /mcp endpoint expose
5. Client connect -> tools/list
6. Client call -> tools/call
```

Server flow — scan → schema → register → expose → discover → call. [2346][2350]

---

## 6. Full Working Example

### MCP Server (Port 8080)

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
        type: SYNC
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

### MCP Client (Port 8081)

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
        enabled: true
        type: SYNC
        toolcallback:
          enabled: true
        streamable-http:
          connections:
            tool-server:
              url: http://localhost:8080
              endpoint: /mcp
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
            .defaultSystem("""
                You are a helpful assistant.
                Use MCP tools to answer accurately.
                """)
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

### Gotcha 1 — Sync/Async Mix

```text
Problem:
SYNC + ASYNC clients = error

Fix:
Ek hi type (all SYNC ya all ASYNC)
```

### Gotcha 2 — Duplicate Tool Names

```text
Problem:
Multi-server same name

Fix:
SyncMcpToolCallbackProvider validate (error)
```

### Gotcha 3 — Wrong Transport

```text
Problem:
Client SSE, server Streamable

Fix:
Match (both Streamable ya both SSE)
```

### Gotcha 4 — No toolcallback.enabled

```text
Problem:
Tools not discovered

Fix:
toolcallback.enabled=true
```

---

## 9. Interview Questions

| Question | Answer |
|---|---|
| MCP kya? | Open protocol (LLM apps <-> tools/data). |
| MCP roles? | Host, Client, Server. |
| MCP primitives? | Tools, Resources, Prompts. |
| Transports? | STDIO, SSE, Streamable HTTP, Stateless. |
| SyncMcpToolCallbackProvider? | Auto-discover + consolidate + validate. |
| MCP vs local tools? | Decouple + dynamic vs same service + static. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Local tool | STDIO |
| Remote service | Streamable HTTP |
| Serverless | Stateless Streamable HTTP |
| Multi-server | SyncMcpToolCallbackProvider |
| Tool discovery | tools/list |

---

## 10. Key Takeaways

```text
1. MCP = open protocol (N×M → N+M).

2. Roles = Host + Client + Server.

3. Primitives = Tools + Resources + Prompts.

4. Transports = STDIO + SSE + Streamable HTTP.

5. Streamable HTTP = production recommended.

6. SyncMcpToolCallbackProvider = auto-discover.

7. Duplicate names = validate (error).

8. SYNC/ASYNC = mix nahi.

9. Server = @Tool beans scan.

10. Client = startup pe discover.
```

---

## 11. Next Part Preview

**Part 16: Chat Memory (Conversation History)**

Isme hum cover karenge:

```text
- Chat memory deep dive
- MessageChatMemoryAdvisor
- In-memory / JDBC / Redis
- Conversation ID
- Long-term memory
```