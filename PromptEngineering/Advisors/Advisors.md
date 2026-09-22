# 4. Structured Output, Tool Calling & Advisors

This is the file that explains what's actually happening when you call `.entity(SomeClass.class)`, how tool calling works end to end, and what "Advisors" are — three things that look like separate features but are really solving the same underlying problem: getting the model to do something *reliable and structured* instead of just chatting.

## How `.entity()` actually works

When you write:

```java
record ActorFilms(String actor, List<String> movies) {}

ActorFilms result = chatClient.prompt()
    .user("Generate the filmography for a random actor.")
    .call()
    .entity(ActorFilms.class);
```

`.entity()` is a convenience wrapper around Spring AI's **Structured Output Converter** abstraction, and it does two things — one *before* the model call, one *after*:

1. **Before the call:** it looks at your target type (`ActorFilms` in this case), generates a JSON Schema from it, and silently appends format instructions to your prompt telling the model exactly what shape of JSON to return.
2. **After the call:** it takes the model's raw text output, parses it as JSON, and deserializes it into an instance of your class using Jackson's `ObjectMapper`.

The class doing this work is `BeanOutputConverter<T>` — one of several converters Spring AI ships:

- **`BeanOutputConverter<T>`** — target is a Java class or record; generates a schema and deserializes into it.
- **`MapOutputConverter`** — target is a generic `Map<String, Object>` when you don't want a dedicated class.
- **`ListOutputConverter`** — target is a simple list of values.
- **`AbstractConversionServiceOutputConverter<T>` / `AbstractMessageOutputConverter<T>`** — lower-level building blocks if you're rolling your own converter (e.g. parsing YAML or CSV instead of JSON).

For generic types — like your `List<Prog>` example — a plain `.class` reference can't capture the generic parameter (Java erases it at runtime), which is exactly why `ParameterizedTypeReference` exists:

```java
List<ActorFilms> results = chatClient.prompt()
    .user("Generate filmographies for three random actors.")
    .call()
    .entity(new ParameterizedTypeReference<List<ActorFilms>>() {});
```

**Important honesty about this feature:** the model is not *guaranteed* to return valid, schema-conforming JSON just because you asked nicely — it's a best-effort mechanism. That's why point below (`responseFormat`) exists as a complementary, stronger guarantee from the provider side.

## Reach for the lower-level `StructuredOutputConverter` directly when...

Most apps never touch this — `.entity()` covers it. Drop down when you need to:
- Parse output the built-in converter rejects (e.g. the model wrapped valid JSON in a ```` ```json ```` markdown fence and the parser chokes on it).
- Produce a non-JSON format, like YAML or CSV.
- Use a converter against the low-level `ChatModel` API directly instead of through `ChatClient`.

## `responseFormat` — the provider-side guarantee

Separately from Spring AI's own converter, OpenAI (and other providers) offer their own server-side structured output modes, set through `OpenAiChatOptions.responseFormat`:

- **`JSON_OBJECT`** mode — guarantees the response is at least syntactically valid JSON (doesn't enforce *which* shape).
- **`JSON_SCHEMA`** mode — you supply an actual JSON Schema, and the provider constrains generation to match it exactly.

The strongest, most reliable pattern in production is combining both layers: use `responseFormat` with `JSON_SCHEMA` so the *provider* enforces the shape during generation, and still use `.entity()` / `BeanOutputConverter` on the Spring AI side to deserialize the result into a real Java object. Belt and suspenders — one guarantees the output is well-formed, the other turns it into something your code can actually use.

## Tool calling (function calling)

Tool calling is how you let the model *take actions* or *fetch live data* instead of only generating text — checking a weather API, looking up a record in your database, sending an email.

You define a tool (a plain method annotated `@Tool`, or a `java.util.Function`, or a lower-level `ToolCallback`), then register it on the call:

```java
String response = chatClient.prompt("What's the weather in Amsterdam?")
    .tools(new WeatherTools())
    .call()
    .content();
```

What happens behind the scenes, in Spring AI's current architecture, is a loop:

1. Spring AI sends your tool *definitions* (name, description, input schema — automatically extracted from your annotated method) to the model along with the prompt.
2. The model decides: answer directly, or request a tool call.
3. If it requests a tool call, Spring AI's `ToolCallingAdvisor` actually executes your Java method, appends the result back into the conversation, and sends the whole thing back to the model.
4. This repeats until the model produces a final answer with no more tool calls pending.

This loop is why tool calling is sometimes described as "agentic" — the model can chain several tool calls together (check weather, then book a flight if sunny) without you writing that control flow by hand. Both `.call()` and `.stream()` support this transparently.

## Advisors — AOP for AI calls

Advisors are Spring AI's answer to "how do I add cross-cutting behavior — memory, retrieval, logging, safety checks — without cluttering my business logic in every single controller?" If you've used Spring AOP or a `RestTemplate` interceptor before, the mental model is identical: an advisor sits in the request/response pipeline and can inspect or modify things on the way in, on the way out, or both.

You register them either as defaults on the `ChatClient` bean, or per-call:

```java
var chatClient = ChatClient.builder(chatModel)
    .defaultAdvisors(
        MessageChatMemoryAdvisor.builder(chatMemory).build(),
        QuestionAnswerAdvisor.builder(vectorStore).build()
    )
    .build();
```

**Built-in advisors worth knowing:**

- **`MessageChatMemoryAdvisor`** — retrieves prior conversation turns and injects them as proper message history in the prompt, so the model has real conversational memory across calls.
- **`PromptChatMemoryAdvisor`** — achieves a similar goal, but by textually appending a "MEMORY: ..." block into the prompt itself, rather than as separate message objects. Useful for models/setups that don't handle multi-message history well.
- **`VectorStoreChatMemoryAdvisor`** — stores/retrieves conversation memory from a vector store instead of in-memory or a plain table, useful for long-running or very large conversation histories.
- **`QuestionAnswerAdvisor`** — the RAG (Retrieval-Augmented Generation) advisor. Before the call, it searches a vector store for relevant documents and stuffs them into the prompt as grounding context, so answers can cite your own private data instead of only what the model learned during training.
- **`SimpleLoggerAdvisor`** — logs the request/response for debugging. Turn it on via `logging.level.org.springframework.ai.chat.client.advisor=DEBUG`.
- **`SafeGuardAdvisor`** — a basic content-safety advisor; it can block a request outright by short-circuiting the chain instead of forwarding it to the model.

**Order matters.** Advisors run in the order they're registered, and where an advisor sits relative to the `ToolCallingAdvisor` determines whether it only sees the *final* result of a multi-step tool-calling loop, or *every intermediate iteration* of it. As a concrete example: if you want memory to include every tool call and result from a conversation (not just the final answer), the memory advisor needs to sit "inside" the tool-calling loop rather than wrapped entirely around it.

**Writing your own advisor** is straightforward once you've seen the built-in ones — you implement an advisor interface with a method that receives the request, can transform it, calls `chain.next(...)` to continue down the pipeline, and can transform the response on the way back out. This is exactly where you'd put something like: redacting PII before it reaches the model, enforcing a per-user rate limit, or injecting a company-specific disclaimer into every response.

## How these three features usually combine in a real app

A realistic "ask a question about our internal docs" endpoint typically stacks all three ideas from this file at once:
1. A **memory advisor** so the conversation has context across turns.
2. A **QuestionAnswerAdvisor (RAG)** so answers are grounded in your actual documents, not just the model's training data.
3. **Tools**, if the assistant also needs to *do* something (create a ticket, look up an order status) rather than only answer questions.
4. **`.entity()`** on the final call if the calling code needs a structured object back instead of prose — e.g. the UI wants `{ answer: string, sources: string[] }`, not a paragraph it has to parse itself.

None of these four things replace each other — they compose.
