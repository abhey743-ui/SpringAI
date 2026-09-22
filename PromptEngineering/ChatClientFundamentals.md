# 1. ChatClient Fundamentals

## The two layers: ChatModel vs. ChatClient

Spring AI actually gives you two ways to talk to an LLM, and it matters which one you're using.

- **`ChatModel`** is the low-level interface. There's one implementation per provider — `OpenAiChatModel`, `AnthropicChatModel`, `AzureOpenAiChatModel`, and so on. You build a `Prompt` object yourself, call the model, and read the raw `ChatResponse` yourself. This is the "JDBC" layer — powerful, verbose, full control.
- **`ChatClient`** is the high-level, fluent API built *on top of* `ChatModel`. You inject a builder, chain a few calls, and get your answer back. If you've used Spring's `RestClient`, `WebClient`, or `JdbcClient`, the shape will feel instantly familiar. It also gives you built-in support for memory, RAG, structured output, and tool calling without you writing that plumbing yourself.

For basically every application, `ChatClient` is the right starting point. You only drop down to `ChatModel` when you need something `ChatClient` doesn't expose yet.

## Getting an instance

The normal way is constructor injection of `ChatClient.Builder` — Spring Boot autoconfigures this for you the moment you have a model starter (like `spring-ai-starter-model-openai`) on the classpath and an API key configured:

```java
public class MyController {
    private final ChatClient chatClient;

    public MyController(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }
}
```

If you're wiring up more than one model (say, a cheap model for simple tasks and a stronger one for complex reasoning — more on why in file 5), you disable the single auto-configured builder with `spring.ai.chat.client.enabled=false` and build your own `ChatClient.Builder` instances programmatically from each `ChatModel` bean instead.

You can also skip the builder pattern entirely with the static factory: `ChatClient.create(chatModel)`.

## The fluent chain, piece by piece

Every call follows the same shape:

```
chatClient.prompt()      // start building a prompt
    .system(...)          // optional — sets the "rules" message
    .user(...)             // sets what the user is asking
    .call()  or  .stream() // decide execution mode
    .content()  /  .chatResponse()  /  .entity(...)   // this is what actually triggers the call
```

**`prompt()`** has three overloads:
- `prompt()` — no arguments, start building piece by piece with `.user()` / `.system()`.
- `prompt(String content)` — a shortcut that sets the user message directly.
- `prompt(Prompt prompt)` — pass in a fully-built `Prompt` object (useful when you need fine-grained `ChatOptions`, covered in file 2).

**`call()` vs `stream()`** — this is the part that confuses people the most. Neither one actually talks to the model yet. They just decide *how* the eventual call will behave:
- `.call()` → blocking, synchronous.
- `.stream()` → reactive, returns a `Flux` (built on WebClient under the hood), so you get tokens as they're generated.

**The terminal method is what actually fires the request.** You always need one of these after `.call()`/`.stream()`:
- `.content()` → just the plain text answer as a `String` (or `Flux<String>` when streaming).
- `.chatResponse()` → the full `ChatResponse` object — metadata, token usage, finish reason, everything.
- `.entity(SomeClass.class)` → parses the model's output straight into a Java object (file 4 goes deep on this).

So a one-liner chat endpoint really is just:

```java
String answer = chatClient.prompt()
        .user("Explain microservices in two sentences")
        .call()
        .content();
```

And getting the full response with metadata (useful when you want token counts for cost tracking):

```java
ChatResponse response = chatClient.prompt()
        .user("Tell me a joke")
        .call()
        .chatResponse();
```

And streaming:

```java
Flux<String> stream = chatClient.prompt()
        .user("Write a short story")
        .stream()
        .content();
```

## Message roles

Every message you send to the model carries a **role**, and the roles mean different things to the model:

- **System** — the "rules of the game." Persona, tone, hard constraints, things that shouldn't change mid-conversation. Set with `.system(...)`.
- **User** — what the actual person (or your application, on their behalf) is asking. Set with `.user(...)`.
- **Assistant** — a previous response from the model. You mostly only touch this manually when you're manually reconstructing conversation history.
- **Tool** — the result of a tool/function call being fed back to the model (file 4).

A rule of thumb from how production teams structure this: **the system prompt should describe things that are true for every request** (persona, tone, non-negotiable rules), and **the user prompt should describe the specific task at hand.** Mixing the two — jamming instructions into the user message every single time — is what makes prompts inconsistent across a large app.

## PromptTemplate — filling in the blanks

Hard-coding full prompt strings works for a demo, but real prompts need variables. Spring AI's `PromptTemplate` class handles this, and it uses `{}`-style placeholders by default:

```java
String template = "Tell me {count} facts about {topic}.";

String answer = chatClient.prompt()
        .user(u -> u.text(template)
                    .param("count", 3)
                    .param("topic", "black holes"))
        .call()
        .content();
```

Under the hood, this uses the `StringTemplate` engine (an open-source templating library) via a `TemplateRenderer` interface. That's swappable — if `{}` collides with something in your text (like JSON examples), you can configure a different delimiter, or use `NoOpTemplateRenderer` when your string is already fully built and needs no substitution at all.

You can also load a template from a resource file instead of an inline string — handy once your prompts get long or you want prompt text out of your Java source and into version-controlled `.st` files under `src/main/resources/prompts/`:

```java
PromptTemplate template = new PromptTemplate(resource); // resource = a .st file
```

That resource-file pattern is worth adopting early — it's the first step toward treating prompts like the versioned, reviewable artifacts that production teams treat them as (much more on this in file 5).

## Setting defaults vs. overriding at runtime

You don't have to repeat the same system prompt or options on every single call. You can bake defaults into the `ChatClient` itself at construction time, in an `@Configuration` class:

```java
@Bean
ChatClient chatClient(ChatClient.Builder builder) {
    return builder
            .defaultSystem("You are a concise, factual assistant.")
            .build();
}
```

Then every call automatically carries that system prompt unless a specific call overrides it. This "design time vs. runtime" separation is intentional — it keeps your controller/service code minimal, since it only has to supply what's actually variable per-request.
