# Part 27: Spring AI — Streaming + SSE (Real-Time Responses)

> Streaming ka core — **`stream()` vs `call()` (`Flux<String>` vs `String`) + `content()` / `chatResponse()` / `chatClientResponse()` (3 stream types) + SSE (`text/event-stream` MIME) + Spring MVC (`SseEmitter`) vs WebFlux (native `Flux`) + token-by-token (delta chunks) + backpressure (Reactor) + cancellation + frontend (`EventSource` / `fetch` + `ReadableStream`)** — chunk = word / part / whitespace / fragment (no boundary assumption), `produces = TEXT_EVENT_STREAM_VALUE` required, `Flux` = lazy (subscribe → execute). [2154][2183][2496][2499]

---

## 1. Streaming Mental Model

```text
Problem:
LLM response = Slow (seconds)
= User waits (bad UX)

Solution:
Streaming = Token-by-token
= Progressive display
= Perceived latency down
```

### Streaming vs Non-Streaming

| Aspect | Non-Streaming | Streaming |
|---|---|---|
| Response | Full (after done) | Chunks (as generated) |
| Latency | High (wait all) | Low (first token fast) |
| UX | Wait spinner | Progressive text |
| API | `call()` → `String` | `stream()` → `Flux<String>` |

Streaming vs non-streaming — full vs chunks, latency, UX, API. [2154][2496][2499]

### Streaming Flow

```text
1. User → Request
2. Server → LLM (stream=true)
3. LLM → Token chunks (SSE)
4. Server → Client (SSE)
5. Client → Progressive display
```

Streaming flow — request → LLM (stream) → chunks → SSE → display. [2494][2499][2502]

---

## 2. Spring AI Streaming API

```text
stream() = StreamingChatClientResponseSpec
= 3 response types
```

### 3 Stream Types

| Method | Return | Use |
|---|---|---|
| `content()` | `Flux<String>` | Text only |
| `chatResponse()` | `Flux<ChatResponse>` | Metadata (tokens, finish) |
| `chatClientResponse()` | `Flux<ChatClientResponse>` | Advisor context (RAG docs) |

3 stream types — content + chatResponse + chatClientResponse. [2154]

### stream() vs call()

```text
call()  = Synchronous (String)
stream() = Reactive (Flux<String>)

call()  = Wait all → return
stream() = Emit chunks → return
```

`stream()` vs `call()` — reactive vs synchronous. [2154][2183]

### Flux (Lazy)

```text
Flux = Lazy (nothing until subscribe)
= Subscribe → Execute
= Backpressure + cancellation
```

`Flux` — lazy (subscribe → execute), backpressure + cancellation. [2495][2502]

### Chunk Nature

```text
Chunk = Word / part / whitespace / fragment
= No boundary assumption
= Client must handle (append)
```

Chunk — word / part / whitespace / fragment (no boundary). [2499]

---

## 3. SSE (Server-Sent Events)

```text
SSE = Server → Client (one-way)
= HTTP + text/event-stream
= Auto-reconnect (browser)
```

### SSE Basics

```text
SSE = Server-Sent Events
= One-way (server → client)
= HTTP (text/event-stream)
= Browser auto-reconnect
```

SSE — one-way, HTTP, auto-reconnect. [2492][2493][2499]

### SSE Format

```text
event: message
data: Hello

event: message
data:  world

event: done
data: [DONE]
```

SSE format — event + data (per chunk). [2499][2501]

### SSE vs WebSocket

| Aspect | SSE | WebSocket |
|---|---|---|
| Direction | One-way (server → client) | Two-way |
| Protocol | HTTP | WS |
| Reconnect | Auto | Manual |
| Use | LLM streaming | Chat (bidirectional) |

SSE vs WebSocket — one-way vs two-way, auto vs manual reconnect. [2492][2493]

---

## 4. Spring MVC vs WebFlux

```text
Spring MVC = Servlet + SseEmitter
WebFlux = Reactive + native Flux
```

### Spring MVC (SseEmitter)

```java
@GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter stream(@RequestParam String message) {
    SseEmitter emitter = new SseEmitter(120_000L);
    
    chatClient.prompt()
        .user(message)
        .stream()
        .content()
        .subscribe(
            token -> emitter.send(SseEmitter.event().data(token)),
            emitter::completeWithError,
            emitter::complete
        );
    
    return emitter;
}
```

Spring MVC — `SseEmitter` (subscribe → send → complete). [2495][2501]

### WebFlux (Native Flux)

```java
@GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> stream(@RequestParam String message) {
    return chatClient.prompt()
        .user(message)
        .stream()
        .content();
}
```

WebFlux — native `Flux` (direct return). [2494][2496][2499]

### MVC vs WebFlux

| Aspect | Spring MVC | WebFlux |
|---|---|---|
| Model | Servlet (blocking) | Reactive (non-blocking) |
| Streaming | `SseEmitter` | Native `Flux` |
| Code | More (subscribe) | Less (direct) |
| Use | Existing MVC app | Reactive app |

MVC vs WebFlux — servlet vs reactive, SseEmitter vs Flux. [2495][2496]

---

## 5. Frontend Integration

```text
Frontend = EventSource (SSE) / fetch + ReadableStream
```

### EventSource (Simple)

```javascript
const eventSource = new EventSource("/chat/stream?message=Explain Spring AI");

eventSource.onmessage = (event) => {
    document.getElementById("output").innerHTML += event.data;
};

eventSource.addEventListener("done", () => {
    eventSource.close();
});
```

EventSource — `onmessage` + `addEventListener` (done). [2494][2499]

### fetch + ReadableStream (POST)

```javascript
const response = await fetch("/chat/stream", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ message: "Explain Spring AI" })
});

const reader = response.body.getReader();
const decoder = new TextDecoder();

while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    document.getElementById("output").innerHTML += decoder.decode(value);
}
```

fetch + ReadableStream — POST + reader + decoder (progressive). [2493][2499]

---

## 6. Full Working Example

### pom.xml (Dependencies)

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

### application.yml

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
```

### StreamingController

```java
package com.example.streaming.controller;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Flux;

@RestController
@RequestMapping("/api/chat")
public class StreamingController {

    private final ChatClient chatClient;

    public StreamingController(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    public record ChatRequest(String message) {}

    @PostMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> stream(@RequestBody ChatRequest req) {
        return chatClient.prompt()
            .user(req.message())
            .stream()
            .content();
    }

    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> streamGet(@RequestParam String message) {
        return chatClient.prompt()
            .user(message)
            .stream()
            .content();
    }
}
```

### StreamingService

```java
package com.example.streaming.service;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Flux;

@Service
public class StreamingService {

    private final ChatClient chatClient;

    public StreamingService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    public Flux<String> stream(String message) {
        return chatClient.prompt()
            .user(message)
            .stream()
            .content();
    }

    public Flux<ChatResponse> streamWithMetadata(String message) {
        return chatClient.prompt()
            .user(message)
            .stream()
            .chatResponse();
    }
}
```

### index.html (Frontend)

```html
<!DOCTYPE html>
<html>
<head>
    <title>Spring AI Streaming</title>
</head>
<body>
    <input id="message" placeholder="Ask something..." />
    <button onclick="stream()">Send</button>
    <div id="output"></div>

    <script>
        async function stream() {
            const message = document.getElementById("message").value;
            const response = await fetch("/api/chat/stream", {
                method: "POST",
                headers: { "Content-Type": "application/json" },
                body: JSON.stringify({ message })
            });

            const reader = response.body.getReader();
            const decoder = new TextDecoder();
            const output = document.getElementById("output");

            while (true) {
                const { done, value } = await reader.read();
                if (done) break;
                output.innerHTML += decoder.decode(value);
            }
        }
    </script>
</body>
</html>
```

---

## 7. Run + Test

```bash
export OPENAI_API_KEY=sk-...
mvn spring-boot:run

# Stream (curl)
curl -N -X POST http://localhost:8080/api/chat/stream \
  -H "Content-Type: application/json" \
  -d '{"message":"Tell me a joke"}'

# Stream (browser)
# http://localhost:8080/index.html
```

---

## 8. Common Gotchas

### Gotcha 1 — Missing produces

```text
Problem:
SSE not working (JSON response)

Fix:
produces = TEXT_EVENT_STREAM_VALUE
```

### Gotcha 2 — Blocking in WebFlux

```text
Problem:
Blocking call in reactive chain

Fix:
Non-blocking (Flux operators)
```

### Gotcha 3 — No Error Handling

```text
Problem:
Stream fails silently

Fix:
doOnError + fallback
```

### Gotcha 4 — No Cancellation

```text
Problem:
Client disconnect = LLM still runs

Fix:
Flux cancellation (auto)
```

---

## 9. Interview Questions

| Question | Answer |
|---|---|
| Streaming kya? | Token-by-token (progressive). |
| stream() vs call()? | Flux<String> vs String. |
| 3 stream types? | content + chatResponse + chatClientResponse. |
| SSE kya? | Server → Client (one-way, HTTP). |
| MVC vs WebFlux? | SseEmitter vs native Flux. |
| Flux kya? | Lazy (subscribe → execute). |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Real-time UX | Streaming (SSE) |
| Metadata needed | chatResponse() |
| RAG context needed | chatClientResponse() |
| Existing MVC app | SseEmitter |
| Reactive app | Native Flux |

---

## 10. Key Takeaways

```text
1. Streaming = token-by-token (progressive).

2. stream() = Flux<String> (reactive).

3. 3 types = content + chatResponse + chatClientResponse.

4. SSE = one-way (server → client).

5. MVC = SseEmitter, WebFlux = native Flux.

6. produces = TEXT_EVENT_STREAM_VALUE (required).

7. Flux = lazy (subscribe → execute).

8. Chunk = word / part / whitespace / fragment.

9. Frontend = EventSource / fetch + ReadableStream.

10. Backpressure + cancellation = built-in.
```

---

## 11. Next Part Preview

**Part 28: Testing (Unit + Integration + Mock LLM)**

Isme hum cover karenge:

```text
- Testing deep dive
- Unit testing (mock LLM)
- Integration testing (Testcontainers)
- ChatClient testing
- Prompt testing
```