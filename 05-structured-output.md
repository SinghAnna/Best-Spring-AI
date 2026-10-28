# Part 5: Spring AI — Structured Output

> Structured output ka core — **`.entity(Class)` (record → JSON schema → typed object) + `ParameterizedTypeReference` (List/Map) + nested records + enums** — `content()` ki jagah `entity()` use karo, Spring AI schema generate + parse karta hai. [2208][2209][2216][2221]

---

## 1. Structured Output Mental Model

```text
content() = String (parse manually)
entity()  = Typed Java object (auto)

Traditional:
LLM -> JSON string -> Manual parse -> Object

Spring AI:
LLM -> JSON (schema-guided) -> Auto-deserialize -> Record
```

### Behind the Scenes

```text
1. Record -> JSON Schema (auto-generate)
2. Schema -> Prompt me append (instructions)
3. LLM -> JSON response
4. JSON -> Record (auto-deserialize)
```

`entity()` — schema generate + prompt append + deserialize (3 steps auto). [2208][2209][2216]

---

## 2. Basic Entity (Record)

```java
record ActorsFilms(String actor, List<String> movies) {}

ActorsFilms films = chatClient.prompt()
    .user("Generate the filmography for a random actor.")
    .call()
    .entity(ActorsFilms.class);

films.actor();  // "Tom Hanks"
films.movies(); // ["Forrest Gump", ...]
```

`entity(Class)` — record → typed object, no manual parse. [2208][2221]

### Why Records

```text
- Immutable
- Compact (no boilerplate)
- JSON-friendly
- Java 14+ standard
```

Records — immutable, compact, JSON-friendly. [2208][2216]

---

## 3. Converters

| Converter | Target | Format |
|---|---|---|
| BeanOutputConverter | Record / POJO | JSON Schema |
| ListOutputConverter | List<String> | Comma-separated |
| MapOutputConverter | Map<String, Object> | JSON object |

Converters — Bean (record), List (comma), Map (JSON). [2209][2212][2214]

### BeanOutputConverter (Direct)

```java
BeanOutputConverter<ActorsFilms> converter = 
    new BeanOutputConverter<>(ActorsFilms.class);

String format = converter.getFormat();  // JSON schema instructions

Prompt prompt = new Prompt(
    "Generate filmography for Tom Hanks\n" + format);

ChatResponse response = chatModel.call(prompt);
ActorsFilms films = converter.convert(
    response.getResult().getOutput().getText());
```

`BeanOutputConverter` — `getFormat()` (schema) + `convert()` (parse). [2209][2211]

---

## 4. Generic Types (List / Map)

```text
Generic = List<T> / Map<K,V>
= ParameterizedTypeReference
```

### List<Entity>

```java
List<ActorsFilms> films = chatClient.prompt()
    .user("Generate filmographies for 3 random actors.")
    .call()
    .entity(new ParameterizedTypeReference<List<ActorsFilms>>() {});
```

`ParameterizedTypeReference<List<T>>` — generic type preserve. [2208][2216][2219]

### Map<String, Object>

```java
Map<String, Object> flags = chatClient.prompt()
    .user("Extract feature flags from: ...")
    .call()
    .entity(new ParameterizedTypeReference<Map<String, Object>>() {});
```

`Map<String, Object>` — flexible (no fixed schema). [2219][2216]

### List<String> (Simple)

```java
List<String> items = chatClient.prompt()
    .user("List 5 programming languages")
    .call()
    .entity(new ParameterizedTypeReference<List<String>>() {});
```

`List<String>` — simple list (comma-separated). [2212][2214]

---

## 5. Nested Records + Enums

```text
Nested = Record inside Record
Enum = Fixed set of values
```

### Complex Example

```java
record Recipe(
    String name,
    String cuisine,
    Difficulty difficulty,
    int prepTimeMinutes,
    List<Ingredient> ingredients,
    List<String> steps
) {
    record Ingredient(String name, String quantity) {}
    enum Difficulty { EASY, MEDIUM, HARD }
}

Recipe recipe = chatClient.prompt()
    .user("Generate a vegetarian lasagna recipe.")
    .call()
    .entity(Recipe.class);
```

Nested record + enum — complex hierarchy supported. [2216][2222]

### Nested Benefits

```text
- Type-safe (compile-time)
- No manual parsing
- Schema auto-generate (nested)
- Validation (enum)
```

Nested — type-safe, auto-schema, enum validation. [2216][2222]

---

## 6. responseEntity() (Entity + Metadata)

```text
entity() = Sirf object
responseEntity() = Object + ChatResponse (metadata)
```

### Code Example

```java
ResponseEntity<ChatResponse, ActorsFilms> result = 
    chatClient.prompt()
        .user("Generate filmography for Tom Hanks.")
        .call()
        .responseEntity(ActorsFilms.class);

ActorsFilms films = result.entity();
ChatResponse raw = result.response();
long tokens = raw.getMetadata().getUsage().getTotalTokens();
```

`responseEntity()` — entity + ChatResponse (token usage). [2221]

### Use Case

```text
- Token usage tracking (cost)
- Model info (observability)
- Entity + metadata dono chahiye
```

`responseEntity()` — cost tracking + observability. [2221]

---

## 7. Native Structured Output (Provider-Native)

```text
Default = Prompt-based (schema in prompt)
Native = Provider-native (schema in API)
```

### Native Example

```java
ActorsFilms films = chatClient.prompt()
    .user("Generate filmography for Tom Hanks.")
    .call()
    .entity(ActorsFilms.class, spec -> 
        spec.useProviderStructuredOutput());
```

`useProviderStructuredOutput()` — provider-native (OpenAI JSON mode). [2210][2211]

### Native vs Prompt-Based

| Aspect | Prompt-Based | Native |
|---|---|---|
| Schema | Prompt me text | API me schema |
| Reliability | Model-dependent | Provider-enforced |
| Compatibility | All providers | Supported providers only |
| Validation | Post-parse | Pre-validate |

Native — provider-enforced (more reliable), prompt-based — universal. [2210][2211]

### Validation

```java
.entity(ActorsFilms.class, spec -> 
    spec.useProviderStructuredOutput()
        .validateSchema());
```

`validateSchema()` — schema validation (strict). [2210][2218]

---

## 8. Full Working Example

### StructuredOutputService

```java
package com.example.structured.service;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.core.ParameterizedTypeReference;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;

@Service
public class StructuredOutputService {

    private final ChatClient chatClient;

    public StructuredOutputService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    // 1. Simple record
    public record Person(String name, int age, String city) {}

    public Person extractPerson(String text) {
        return chatClient.prompt()
            .user("Extract person info: " + text)
            .call()
            .entity(Person.class);
    }

    // 2. List of records
    public record Book(String title, String author, int year) {}

    public List<Book> recommendBooks(String genre) {
        return chatClient.prompt()
            .user("Recommend 3 " + genre + " books.")
            .call()
            .entity(new ParameterizedTypeReference<List<Book>>() {});
    }

    // 3. Nested + enum
    public record Recipe(
        String name,
        String cuisine,
        Difficulty difficulty,
        List<Ingredient> ingredients,
        List<String> steps
    ) {
        record Ingredient(String name, String quantity) {}
        enum Difficulty { EASY, MEDIUM, HARD }
    }

    public Recipe getRecipe(String dish) {
        return chatClient.prompt()
            .user("Generate a recipe for " + dish)
            .call()
            .entity(Recipe.class);
    }

    // 4. Map (flexible)
    public Map<String, Object> extractFlags(String text) {
        return chatClient.prompt()
            .user("Extract feature flags: " + text)
            .call()
            .entity(new ParameterizedTypeReference<Map<String, Object>>() {});
    }

    // 5. Entity + metadata
    public record Result<T>(T entity, long tokens) {}

    public Result<Person> extractWithTokens(String text) {
        var result = chatClient.prompt()
            .user("Extract person: " + text)
            .call()
            .responseEntity(Person.class);
        return new Result<>(result.entity(), 
            result.response().getMetadata().getUsage().getTotalTokens());
    }
}
```

### ChatController

```java
package com.example.structured.controller;

import com.example.structured.service.StructuredOutputService;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/structured")
public class StructuredOutputController {

    private final StructuredOutputService service;

    public StructuredOutputController(StructuredOutputService service) {
        this.service = service;
    }

    public record TextRequest(String text) {}
    public record GenreRequest(String genre) {}
    public record DishRequest(String dish) {}

    @PostMapping("/person")
    public StructuredOutputService.Person person(@RequestBody TextRequest req) {
        return service.extractPerson(req.text());
    }

    @PostMapping("/books")
    public List<StructuredOutputService.Book> books(@RequestBody GenreRequest req) {
        return service.recommendBooks(req.genre());
    }

    @PostMapping("/recipe")
    public StructuredOutputService.Recipe recipe(@RequestBody DishRequest req) {
        return service.getRecipe(req.dish());
    }

    @PostMapping("/flags")
    public Map<String, Object> flags(@RequestBody TextRequest req) {
        return service.extractFlags(req.text());
    }

    @PostMapping("/person-tokens")
    public StructuredOutputService.Result<StructuredOutputService.Person> 
        personWithTokens(@RequestBody TextRequest req) {
        return service.extractWithTokens(req.text());
    }
}
```

---

## 9. Run + Test

```bash
export OPENAI_API_KEY=sk-...
mvn spring-boot:run

# Person extraction
curl -X POST http://localhost:8080/api/structured/person \
  -H "Content-Type: application/json" \
  -d '{"text":"John Doe, 30, lives in NYC"}'

# Book recommendations
curl -X POST http://localhost:8080/api/structured/books \
  -H "Content-Type: application/json" \
  -d '{"genre":"sci-fi"}'

# Recipe (nested + enum)
curl -X POST http://localhost:8080/api/structured/recipe \
  -H "Content-Type: application/json" \
  -d '{"dish":"vegetarian lasagna"}'

# Person + tokens
curl -X POST http://localhost:8080/api/structured/person-tokens \
  -H "Content-Type: application/json" \
  -d '{"text":"John Doe, 30, NYC"}'
```

---

## 10. Common Gotchas

### Gotcha 1 — Generic Type Erasure

```text
Problem:
List<T> direct pass = erasure

Fix:
ParameterizedTypeReference
```

### Gotcha 2 — No Validation

```text
Problem:
LLM wrong format

Fix:
validateSchema() + enum
```

### Gotcha 3 — Complex Nested Too Deep

```text
Problem:
Provider schema limits

Fix:
Flat records + container
```

### Gotcha 4 — content() vs entity()

```text
Problem:
Manual parse (error-prone)

Fix:
entity() (auto)
```

---

## 11. Interview Questions

| Question | Answer |
|---|---|
| Structured output kya? | LLM output → typed Java object. |
| entity() kya? | Record → JSON schema → typed object. |
| Generic types? | ParameterizedTypeReference. |
| Nested records? | Supported (auto-schema). |
| Native vs prompt-based? | Provider-enforced vs prompt text. |
| responseEntity()? | Entity + ChatResponse (tokens). |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Typed object | entity(Class) |
| List of objects | ParameterizedTypeReference |
| Flexible schema | Map<String, Object> |
| Cost tracking | responseEntity() |
| Strict validation | validateSchema() |

---

## 12. Key Takeaways

```text
1. entity() = typed object (no parse).

2. Record = schema source.

3. ParameterizedTypeReference = generics.

4. Nested = supported (auto).

5. Enum = validation.

6. responseEntity() = entity + tokens.

7. Native = provider-enforced.

8. validateSchema() = strict.

9. content() = raw string (avoid).

10. Converters = Bean / List / Map.
```

---

## 13. Next Part Preview

**Part 6: RAG Fundamentals**

Isme hum cover karenge:

```text
- RAG deep dive
- Why RAG (vs fine-tuning)
- Architecture (ETL + retrieval + generation)
- Vector stores
- First RAG app
```