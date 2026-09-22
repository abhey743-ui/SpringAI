# 7. Built-in Advisors — Deep Dive (Deps, Config, Code, When to Use Each)

File 6 covered the architecture. This file goes through every advisor Spring AI ships out of the box, one at a time, with exactly what you need to add and configure to actually run it — not just the happy-path snippet.

---

## Part A — Chat Memory Advisors

Before touching the advisors themselves, you need to understand that **memory in Spring AI is two separate concerns, cleanly split:**

- **`ChatMemoryRepository`** — pure storage. Just saves and loads raw messages somewhere (in memory, a database, a vector store). It doesn't know or care about conversation windowing.
- **`ChatMemory`** — the policy layer on top. `MessageWindowChatMemory` is the built-in implementation; it decides *how much* history to keep (a sliding window of messages, default 20), evicting the oldest non-system messages once the window fills up.

You almost always end up wiring: **a repository (where) + `MessageWindowChatMemory` (how much) + a memory advisor (how it's injected into the prompt).**

### Setting up the repository

**In-memory (default, zero config, lost on restart):**
```java
// Auto-configured for you if nothing else is on the classpath — you can also build it directly:
ChatMemoryRepository repository = new InMemoryChatMemoryRepository();
```

**Persisted via JDBC (the realistic choice for anything production-facing):**

Maven:
```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-chat-memory-repository-jdbc</artifactId>
</dependency>
```

`application.yml` — you still need a real datasource, this just tells Spring AI to manage its own table on it:
```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: myuser
    password: mypassword
  ai:
    chat:
      memory:
        repository:
          jdbc:
            initialize-schema: embedded   # embedded | always | never
```

Spring AI supports Postgres, MySQL/MariaDB, SQL Server, HSQLDB, and Oracle out of the box via a dialect abstraction — the correct SQL dialect is auto-detected from your JDBC URL.

Wiring it together:
```java
@Bean
ChatMemory chatMemory(JdbcChatMemoryRepository repository) {
    return MessageWindowChatMemory.builder()
        .chatMemoryRepository(repository)
        .maxMessages(20)
        .build();
}
```

Cassandra and Neo4j repositories exist too, following the identical pattern — swap the starter artifact, swap the connection properties, same `ChatMemory` bean wiring.

### The three memory advisors — and how they actually differ

- **`MessageChatMemoryAdvisor`** — retrieves prior turns and adds them to the prompt as *proper, separate message objects* (real conversation history, role-tagged). This is the one to reach for by default; it's the cleanest representation and works well with models that handle multi-message context properly.

```java
ChatClient chatClient = ChatClient.builder(chatModel)
    .defaultAdvisors(MessageChatMemoryAdvisor.builder(chatMemory).build())
    .build();

String reply = chatClient.prompt()
    .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, "user-42"))
    .user("What did I just ask you?")
    .call()
    .content();
```

- **`PromptChatMemoryAdvisor`** — achieves the same goal, but textually: it appends a rendered `MEMORY: ...` block directly into the prompt text rather than as separate messages. Reach for this if you're working with a model/setup that doesn't handle multi-message history cleanly, or you specifically want the memory content visible as plain text in the final prompt (useful for debugging what the model actually saw).

- **`VectorStoreChatMemoryAdvisor`** — stores and retrieves memory from a vector store instead of a plain table, using semantic similarity rather than "most recent N messages." This is the right choice for very long-running conversations or knowledge-base-style memory, where "the most *relevant* prior turn" matters more than "the most *recent* prior turn." It needs whatever vector store dependency you're already using for RAG (see Part B).

**A critical detail: always set a conversation ID.** Without one, every user hitting the same `ChatClient` bean can end up sharing (or overwriting) the same memory bucket. Pass it per-request via `.advisors(a -> a.param(ChatMemory.CONVERSATION_ID, someId))`, where `someId` is something durable per real conversation — a session ID, a user ID plus thread ID, etc.

---

## Part B — Retrieval-Augmented Generation (RAG) Advisors

### Dependency
```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-advisors-vector-store</artifactId>
</dependency>
```
Plus, separately, whatever vector store you're actually using (e.g. `spring-ai-starter-vector-store-pgvector`) and an embedding model, since RAG needs to turn text into vectors to do similarity search in the first place.

### `QuestionAnswerAdvisor` — the simple, out-of-the-box RAG flow
Before the call, it runs a similarity search against your vector store using the user's question, then stuffs the matching documents into the prompt as grounding context.

```java
var qaAdvisor = QuestionAnswerAdvisor.builder(vectorStore)
    .searchRequest(SearchRequest.builder()
        .similarityThreshold(0.8d)
        .topK(6)
        .build())
    .build();

ChatResponse response = chatClient.prompt()
    .advisors(qaAdvisor)
    .user("What's our refund policy?")
    .call()
    .chatResponse();
```

- `similarityThreshold` filters out weak matches (0.0–1.0; higher = stricter).
- `topK` caps how many documents get pulled in — directly affects both answer quality *and* your token bill (file 5's context-trimming advice applies here directly).
- You can restrict *which* documents are searchable with a portable, SQL-like filter expression, either fixed at advisor-creation time, or dynamically per request:

```java
String content = chatClient.prompt()
    .user("Please answer my question about pricing")
    .advisors(a -> a.param(QuestionAnswerAdvisor.FILTER_EXPRESSION, "type == 'Spring'"))
    .call()
    .content();
```

### `RetrievalAugmentationAdvisor` — the modular, professional RAG flow
This one needs an **additional** dependency beyond the vector-store advisor artifact:
```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-rag</artifactId>
</dependency>
```

Where `QuestionAnswerAdvisor` is "retrieve, then stuff into the prompt" as one fixed step, `RetrievalAugmentationAdvisor` lets you compose the RAG pipeline out of separate, swappable pieces — a document retriever, a query transformer (e.g. rewriting a vague follow-up question into a self-contained one before searching), and a query augmentor (how retrieved context actually gets merged into the prompt):

```java
Advisor retrievalAdvisor = RetrievalAugmentationAdvisor.builder()
    .documentRetriever(VectorStoreDocumentRetriever.builder()
        .vectorStore(vectorStore)
        .similarityThreshold(0.75)
        .build())
    .build();
```

**Rule of thumb for which to reach for:** start with `QuestionAnswerAdvisor` — it covers the majority of "answer questions using our documents" use cases with almost no setup. Move to `RetrievalAugmentationAdvisor` once you need query rewriting, multiple retrieval sources, or a genuinely custom augmentation strategy — don't reach for the more complex one by default.

---

## Part C — `SimpleLoggerAdvisor`

No extra dependency — it ships in core. It just observes the request and response and logs them; it never modifies either.

```java
chatClient.prompt()
    .advisors(new SimpleLoggerAdvisor())
    .user("Hello!")
    .call()
    .content();
```

To actually see the output, turn the logger on:
```properties
logging.level.org.springframework.ai.chat.client.advisor=DEBUG
```

Keep this at a low order value (it defaults to `0`) if you want it to log the *original* incoming request and the *fully processed final* outgoing response — remember the stack behavior from file 6: a lower order means it's the outermost layer, seeing the rawest request and the most fully-processed response.

---

## Part D — `SafeGuardAdvisor`

No extra dependency. It's a genuinely simple sensitive-word list checker — if the prompt or (depending on configuration) the response contains a listed word, it blocks the call and returns a canned response instead of forwarding to the model.

```java
Advisor safeGuard = SafeGuardAdvisor.builder()
    .sensitiveWords(List.of("confidential", "internal-only"))
    .build();
```

**Be honest with yourself about what this actually is:** it's a basic keyword filter. It's trivially defeated by a synonym, a typo, different casing, or a different language. Real production guardrails treat this as one thin layer among several, not the whole solution — the honest, professional pattern is layering: a strong system prompt, this advisor for the cheap/obvious cases, a dedicated moderation model or classifier call for the real content-safety decision, and output-side scanning for anything that shouldn't leave the system (leaked secrets, PII). File 8 shows what a second, custom layer looks like in practice.

One more sharp edge worth knowing: ordering `SafeGuardAdvisor` *after* a memory advisor in the chain can cause it to misfire — if prior conversation history (pulled in by the memory advisor) happens to contain a sensitive word, `SafeGuardAdvisor` can trigger on old conversation content instead of the new message. Keep it early in the chain (a low order value, so it inspects the request before memory/RAG have had a chance to inject anything into it).

---

## Quick reference — what to reach for and when

| You need... | Reach for | Extra dependency |
|---|---|---|
| The assistant to remember earlier turns in a conversation | `MessageChatMemoryAdvisor` + a `ChatMemoryRepository` | None (in-memory) or a repository starter (persisted) |
| Memory that's searched by relevance, not recency | `VectorStoreChatMemoryAdvisor` | A vector store starter |
| Answers grounded in your own documents | `QuestionAnswerAdvisor` | `spring-ai-advisors-vector-store` + a vector store starter |
| A composable, multi-step RAG pipeline | `RetrievalAugmentationAdvisor` | The above, plus `spring-ai-rag` |
| Visibility into exactly what's sent/received | `SimpleLoggerAdvisor` | None |
| A first, cheap line of defense against obviously bad input | `SafeGuardAdvisor` | None (but pair it with more, see file 8) |

Next: file 8 builds a real, layered, professional advisor implementation from scratch — including testing it and getting the ordering right.
