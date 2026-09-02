# Spring AI Architecture — How ChatClient, ChatModel & Auto-Configuration Actually Work

## 1. The Big Picture First

There are **two layers** you interact with in Spring AI, and understanding the difference between them is the whole key to understanding the architecture:

| Layer | Class | Job |
|---|---|---|
| High-level, developer-facing | `ChatClient` | The fluent API you actually call (`.prompt().user(...).call()`) — like `WebClient` |
| Low-level, provider-facing | `ChatModel` | The actual thing that talks to the AI provider's API (OpenAI, Anthropic, NVIDIA NIM, Ollama, etc.) |

**`ChatClient` does NOT talk to the AI provider directly.** It's a wrapper. Internally, every `ChatClient` is *bound to* a `ChatModel` instance, and it's the `ChatModel` that does the real HTTP communication with the provider.

```
Your Code → ChatClient (fluent API, advisors, prompt building)
                 │
                 ▼
             ChatModel (provider-specific implementation)
                 │
                 ▼
        Actual HTTP call to the AI provider (OpenAI / NVIDIA / Anthropic / etc.)
```

So when you write:

```java
chatClient.prompt().user("hello").call().content();
```

`ChatClient` builds the `Prompt` object, runs it through any configured Advisors (memory, RAG, tools), and then **delegates the actual call** to the `ChatModel` it was built with.

## 2. The Bean Creation Journey — Step by Step

This is the part that confused you, so let's go slowly through what happens **from the moment you add a dependency** to the moment you can `@Autowired` a working `ChatClient`.

### Step 1 — You add a starter dependency

Everything starts here. Example (Maven):

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId>
</dependency>
```

This single dependency pulls in:
- The `OpenAiChatModel` implementation class (implements the `ChatModel` interface)
- An **auto-configuration class** for it (e.g. `OpenAiChatAutoConfiguration`)
- The property classes that bind your YAML/properties (e.g. `OpenAiChatProperties`)

**This is the trigger for everything else.** Spring Boot's auto-configuration mechanism only activates provider-specific beans *if their classes are present on the classpath* (`@ConditionalOnClass`). That's why swapping providers is often "just swap the starter dependency."

### Step 2 — Spring Boot scans and activates the matching auto-configuration

At startup, Spring Boot's auto-configuration mechanism finds `OpenAiChatAutoConfiguration` (bundled inside the starter) and checks its conditions, roughly:

- Is the `OpenAiChatModel` class on the classpath? ✅ (because you added the starter)
- Is there already a custom `ChatModel` bean defined by you? If yes → back off, don't create the default one (`@ConditionalOnMissingBean`)
- Are the required properties present (like an API key)? ✅ if you configured them

If all conditions pass, Spring Boot creates and registers a **`ChatModel` bean** — in this case a `OpenAiChatModel` instance — fully configured from your `application.yml`.

### Step 3 — Your YAML properties get bound into that bean

This is where your config comes in:

```yaml
spring:
  ai:
    model:
      chat: openai
    openai:
      api-key: ${NVIDIA_API_KEY}
      base-url: https://integrate.api.nvidia.com/v1
      chat:
        model: nvidia/nemotron-3-ultra-550b-a55b
        max-tokens: 16384
```

Let's break this down piece by piece, because each line maps directly to something in the bean creation process:

- **`spring.ai.model.chat: openai`** → This tells Spring AI *which chat model implementation to activate* when there's ambiguity (more on this in section 3 below). It's essentially saying: "use the OpenAI-flavored auto-configuration for chat."
- **`spring.ai.openai.api-key`** → Bound into `OpenAiChatProperties`, then used to build the underlying API client (credentials).
- **`spring.ai.openai.base-url`** → This is the clever part of your config. You're using the **OpenAI starter**, but pointing its `base-url` at NVIDIA's NIM endpoint (`integrate.api.nvidia.com`) instead of `api.openai.com`. This works because NVIDIA's NIM service exposes an **OpenAI-compatible API**. Spring AI doesn't need a dedicated "NVIDIA starter" for this — it just needs an OpenAI-protocol-speaking endpoint, and the OpenAI starter handles the request/response shape.
- **`spring.ai.openai.chat.model`** → Which actual model to request from that endpoint (`nvidia/nemotron-3-ultra-550b-a55b`).
- **`spring.ai.openai.chat.max-tokens`** → Passed straight into the request options sent with every call.

All of this gets bound into an `OpenAiChatProperties` object by Spring Boot's standard `@ConfigurationProperties` mechanism (nothing special to Spring AI — same as how you'd configure `spring.datasource.*`).

### Step 4 — The `ChatModel` bean is now fully built and sitting in the Spring context

At this point, Spring's `ApplicationContext` contains a ready-to-use `ChatModel` bean (an `OpenAiChatModel` instance in your case), fully wired with your API key, base URL, and model name.

### Step 5 — `ChatClientAutoConfiguration` kicks in and builds on top of it

Separately, another auto-configuration class — `ChatClientAutoConfiguration` — activates (it only needs the `ChatClient` class on the classpath, which comes with `spring-ai-client-chat`). Its job:

- Take the auto-configured `ChatModel` bean from Step 4
- Wrap it into a **`ChatClient.Builder` bean**, registered with **prototype scope** (meaning every place you inject it gets its own fresh builder instance, not a shared singleton)
- Attach observability hooks and any `ChatClientBuilderCustomizer` beans you've defined

This is why you typically inject `ChatClient.Builder`, not `ChatClient` directly:

```java
@RestController
class MyController {
    private final ChatClient chatClient;

    MyController(ChatClient.Builder chatClientBuilder) {
        this.chatClient = chatClientBuilder.build(); // <-- bound to the auto-configured ChatModel here
    }
}
```

When you call `.build()`, **that's the exact moment `ChatClient` gets bound to the `ChatModel`.** Everything before this step was just preparing the `ChatModel`; this step ties the two layers together into one usable object.

## 3. "Default Bean is OpenAI" — What That Actually Means

You mentioned you know the default is "something" — here's the precise mechanic:

- Spring AI does **not** hardcode "OpenAI is the default provider" in a magical sense. What happens is: **whichever provider's starter dependency is on your classpath gets auto-configured.**
- If you only have `spring-ai-starter-model-openai` on the classpath, its `ChatModel` bean is the only candidate → it's used, no ambiguity.
- **The confusion starts when you have *multiple* model starters on the classpath at once** (e.g., you added both OpenAI and Ollama starters for a multi-model app). In that case, Spring AI can't automatically know which one should back the *default* `ChatClient`/`ChatModel` — so this is exactly where the **`spring.ai.model.chat`** property becomes important:

```yaml
spring:
  ai:
    model:
      chat: openai   # explicitly says "use the OpenAI ChatModel as the primary/default one"
```

- If you set `spring.ai.model.chat: none`, Spring AI **disables the default chat model auto-configuration entirely**, and you're expected to wire up `ChatClient` beans manually (very common in true multi-model apps where you want full manual control over which `ChatModel` backs which `ChatClient`).

So in your config, `spring.ai.model.chat: openai` is you being explicit (good practice) about which provider's auto-configuration should be treated as the primary one — even though in your case it's likely the *only* model starter anyway.

## 4. Multi-Model Scenario (for when you outgrow single-provider)

If later you add two starters (say OpenAI-compatible NVIDIA + Ollama locally), the pattern becomes:

```yaml
spring:
  ai:
    model:
      chat: none   # turn off default auto-wiring — you'll do it yourself
    openai:
      api-key: ${NVIDIA_API_KEY}
      base-url: https://integrate.api.nvidia.com/v1
    ollama:
      chat:
        model: llama3.2
```

```java
@Configuration
class AiConfig {

    @Bean
    ChatClient nvidiaChatClient(OpenAiChatModel openAiChatModel) {
        return ChatClient.create(openAiChatModel);
    }

    @Bean
    ChatClient localChatClient(OllamaChatModel ollamaChatModel) {
        return ChatClient.create(ollamaChatModel);
    }
}
```

Here you're skipping the auto-configured `ChatClient.Builder` and manually deciding which `ChatModel` each `ChatClient` should bind to — full control, at the cost of losing some auto-configured observability wiring (you can re-add it manually if needed).

## 5. Full Journey, Summarized as One Flow

```
1. Add starter dependency (e.g. spring-ai-starter-model-openai)
        │
2. Classpath now has OpenAiChatModel + OpenAiChatAutoConfiguration
        │
3. Spring Boot auto-config conditions pass → OpenAiChatProperties bound from application.yml
        │
4. OpenAiChatModel bean created & registered in ApplicationContext
        │        (spring.ai.model.chat=openai disambiguates this if multiple providers exist)
        │
5. ChatClientAutoConfiguration wraps that ChatModel into a ChatClient.Builder bean (prototype scope)
        │
6. You inject ChatClient.Builder → call .build()
        │
7. ChatClient is now BOUND to the ChatModel — ready to use
```

## 6. Your Specific Config, Annotated

```yaml
spring:
  application:
    name: AIApplication
  ai:
    model:
      chat: openai                                              # use OpenAI-flavored auto-config as the active chat model
    openai:
      api-key: ${NVIDIA_API_KEY}                                 # ⚠️ use an env var placeholder, never hardcode this
      base-url: https://integrate.api.nvidia.com/v1              # OpenAI-compatible endpoint, but hosted by NVIDIA NIM
      chat:
        model: nvidia/nemotron-3-ultra-550b-a55b                 # actual model requested at that endpoint
        max-tokens: 16384                                        # request parameter passed on every call
```

> ⚠️ **Security note:** Never commit real API keys into `application.yml`. Use `${NVIDIA_API_KEY}` and set the actual value via an environment variable, a `.env` file excluded from git, or a secrets manager. If a key has ever been pasted somewhere it shouldn't (chat, a public repo, a shared doc), the safest move is to revoke/rotate it immediately.

## 7. Official Documentation

📖 **ChatClient API reference:** https://docs.spring.io/spring-ai/reference/api/chatclient.html

📖 **ChatModel / provider API reference:** https://docs.spring.io/spring-ai/reference/api/index.html

📖 **OpenAI Chat integration docs:** https://docs.spring.io/spring-ai/reference/api/chat/openai-chat.html

📖 **Full reference index:** https://docs.spring.io/spring-ai/reference/index.html

---
*Personal learning notes — architecture deep dive on ChatClient/ChatModel binding and the auto-configuration bean journey.*
