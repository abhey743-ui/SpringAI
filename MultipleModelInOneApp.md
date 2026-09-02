# Using Multiple AI Models at the Same Time in Spring AI (Ollama + OpenAI + Others)

## 1. Why You'd Even Want Multiple Models in One App

Before the "how," the "why" — because this isn't just a party trick, it solves real problems:

- **Different tasks need different strength/cost tradeoffs.** A powerful, expensive model (GPT-4o, Claude Sonnet) for complex reasoning or customer-facing answers; a cheap/free local model (Ollama) for simple, high-volume, low-stakes tasks like classification or formatting.
- **Fallback / reliability.** If your primary cloud provider is down or rate-limited, automatically fall back to a secondary provider so your app doesn't fully break.
- **Cost control.** Route cheap/bulk requests to a free local Ollama model, and only pay for the expensive API when it's actually needed.
- **Privacy-sensitive routing.** Keep sensitive data on a local Ollama model, send only non-sensitive queries to a cloud provider.
- **Comparing outputs.** Run the same prompt against two models side by side (useful during development, or for building your own quality-evaluation pipeline).
- **Multimodal specialization.** One model for text, a different specialized model for vision/image understanding, another for embeddings.

This is genuinely a common production pattern — most serious AI applications end up using more than one model.

## 2. The Core Problem You Have to Solve

Here's the thing that trips people up, straight from Spring AI's own team: **Spring Boot doesn't natively support auto-configuring multiple beans of the same type out of the box.** Normally, Spring AI auto-configures exactly *one* `ChatModel` bean and *one* `ChatClient.Builder` bean. If you add two model starters (say, OpenAI + Ollama) without doing anything else, Spring Boot has no way to know which one should be "the" default — you have to be explicit.

So the entire "how" of this file boils down to: **you manually define one bean per model, and use `@Qualifier` to tell Spring exactly which one you want injected where.**

## 3. Step-by-Step Setup

### Step 1 — Add BOTH starter dependencies

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-ollama</artifactId>
</dependency>
```

Adding both means Spring Boot now has *two* auto-configuration candidates on the classpath — `OpenAiChatModel` and `OllamaChatModel` — each fully valid `ChatModel` implementations.

### Step 2 — Configure both providers' properties, side by side

```yaml
spring:
  ai:
    model:
      chat: none        # IMPORTANT — explained in step 3
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o
          temperature: 0.7
    ollama:
      base-url: http://localhost:11434
      chat:
        options:
          model: llama3.2
          temperature: 0.7
```

Notice — each provider still uses its own separate property prefix (`spring.ai.openai.*` and `spring.ai.ollama.*`), exactly like you learned in the providers-comparison file. Nothing changes there; you're just configuring *both* at once instead of picking one.

### Step 3 — Disable the ambiguous default auto-configuration

```yaml
spring:
  ai:
    model:
      chat: none
```

**Why this matters:** you already learned that `spring.ai.model.chat` picks *which* provider becomes the default `ChatModel`/`ChatClient.Builder` when there's ambiguity. With two providers on the classpath and no explicit choice, Spring AI can't guess for you. Setting it to `none` disables the automatic single-`ChatClient.Builder` behavior entirely, so you take **full manual control** instead — which is exactly what you want when running multiple models side by side.

You'll also want to disable the default `ChatClient.Builder` auto-configuration itself, since you're going to build your own `ChatClient` beans manually:

```yaml
spring:
  ai:
    chat:
      client:
        enabled: false
```

### Step 4 — Manually define one `ChatClient` bean per model

This is the actual heart of the setup — a small `@Configuration` class where you explicitly wire each model into its own named bean:

```java
@Configuration
public class MultiModelConfig {

    @Bean
    @Primary   // this one gets injected by default when no @Qualifier is specified
    public ChatClient openAiChatClient(OpenAiChatModel chatModel) {
        return ChatClient.create(chatModel);
    }

    @Bean
    public ChatClient ollamaChatClient(OllamaChatModel chatModel) {
        return ChatClient.create(chatModel);
    }
}
```

What's happening here: Spring Boot's auto-configuration already created the underlying `OpenAiChatModel` and `OllamaChatModel` beans for you (that part still works automatically, because each is a distinct class/type — the ambiguity only existed at the `ChatClient`/`ChatClient.Builder` level). You're just taking those two `ChatModel` beans and wrapping each into its own explicitly-named `ChatClient` bean.

- `@Primary` marks one bean as the default choice when something injects `ChatClient` **without** specifying which one — useful so your "main" model doesn't require a qualifier everywhere.
- The bean method name (`openAiChatClient`, `ollamaChatClient`) automatically becomes that bean's qualifier name, which is what you reference next.

### Step 5 — Inject the specific model you need, using `@Qualifier`

```java
@RestController
public class OpenAiChatController {

    private final ChatClient chatClient;

    public OpenAiChatController(@Qualifier("openAiChatClient") ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    @GetMapping("/openai/chat")
    public String chat(@RequestParam String message) {
        return chatClient.prompt().user(message).call().content();
    }
}
```

```java
@RestController
public class OllamaChatController {

    private final ChatClient chatClient;

    public OllamaChatController(@Qualifier("ollamaChatClient") ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    @GetMapping("/local/chat")
    public String chat(@RequestParam String message) {
        return chatClient.prompt().user(message).call().content();
    }
}
```

Each controller now talks to a **completely different model**, with zero ambiguity — `@Qualifier("...")` tells Spring exactly which bean to hand over, matching the bean method name from Step 4.

## 4. Bonus Pattern — Multiple Models From the SAME Provider

You don't need two different providers to want multiple models — sometimes you want, say, two different Anthropic models (a fast/cheap one and a powerful one) at once. Since both would be the exact same `ChatModel` *type* (`AnthropicChatModel`), you can't just rely on the auto-configured bean twice — you construct the extra one manually, reusing the already-configured API client underneath:

```java
@Configuration
public class MultiAnthropicConfig {

    @Bean
    public ChatModel fastChatModel(
            AnthropicApi anthropicApi,
            AnthropicChatModel defaultChatModel,
            @Value("${spring.ai.anthropic.chat.options.fast-model}") String fastModelName) {

        AnthropicChatOptions options = defaultChatModel.getDefaultOptions().copy();
        options.setModel(fastModelName);

        return AnthropicChatModel.builder()
                .anthropicApi(anthropicApi)
                .defaultOptions(options)
                .build();
    }

    @Bean
    public ChatClient fastChatClient(@Qualifier("fastChatModel") ChatModel fastChatModel) {
        return ChatClient.create(fastChatModel);
    }
}
```

```yaml
spring:
  ai:
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        options:
          model: claude-sonnet-4-20250514        # the "default"/powerful one
          fast-model: claude-haiku-4-5-20251001   # your custom extra property, read via @Value
```

The trick here: reuse the already auto-configured `AnthropicApi` bean (handles auth/connection), copy the *default* options object, override just the `model` field, and build a second `ChatModel` from that — giving you two models from one provider without duplicating your API key config.

## 5. Bonus Pattern — Routing Logic (Deciding at Runtime Which Model to Use)

Sometimes you don't want two fixed endpoints — you want your app to **pick a model dynamically** based on some condition (task complexity, user tier, cost budget, whether it's a sensitive query, etc.):

```java
@Service
public class ModelRouterService {

    private final ChatClient powerfulClient;
    private final ChatClient localClient;

    public ModelRouterService(
            @Qualifier("openAiChatClient") ChatClient powerfulClient,
            @Qualifier("ollamaChatClient") ChatClient localClient) {
        this.powerfulClient = powerfulClient;
        this.localClient = localClient;
    }

    public String answer(String message, boolean isComplexTask) {
        ChatClient chosenClient = isComplexTask ? powerfulClient : localClient;
        return chosenClient.prompt().user(message).call().content();
    }
}
```

This is exactly how the "different tasks need different models" and "cost control" reasons from Section 1 get implemented in real code — your Java logic decides, request by request, which underlying model actually handles it. Remember from the RAG file: **the model never decides this itself** — it's always your application code making the routing choice before the model ever sees anything.

## 6. Bonus Pattern — Fallback (Reliability)

A simple try/catch pattern for automatic failover if your primary provider errors out:

```java
@Service
public class ResilientChatService {

    private final ChatClient primaryClient;
    private final ChatClient fallbackClient;

    public ResilientChatService(
            @Qualifier("openAiChatClient") ChatClient primaryClient,
            @Qualifier("ollamaChatClient") ChatClient fallbackClient) {
        this.primaryClient = primaryClient;
        this.fallbackClient = fallbackClient;
    }

    public String answer(String message) {
        try {
            return primaryClient.prompt().user(message).call().content();
        } catch (Exception e) {
            // primary provider down, rate-limited, or erroring — fall back to local model
            return fallbackClient.prompt().user(message).call().content();
        }
    }
}
```

## 7. Full Config Recap (Everything Together)

```yaml
spring:
  application:
    name: MultiModelApplication
  ai:
    model:
      chat: none                    # disable ambiguous default auto-config
    chat:
      client:
        enabled: false              # disable the single default ChatClient.Builder
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o
          temperature: 0.7
    ollama:
      base-url: http://localhost:11434
      chat:
        options:
          model: llama3.2
          temperature: 0.7
```

```java
@Configuration
public class MultiModelConfig {

    @Bean
    @Primary
    public ChatClient openAiChatClient(OpenAiChatModel chatModel) {
        return ChatClient.create(chatModel);
    }

    @Bean
    public ChatClient ollamaChatClient(OllamaChatModel chatModel) {
        return ChatClient.create(chatModel);
    }
}
```

That's genuinely the whole mechanism — two starters, two property blocks, one config class wiring each `ChatModel` into its own named `ChatClient` bean, and `@Qualifier` everywhere you need to pick one explicitly.

## 8. Quick Checklist / Common Mistakes

- ❌ Forgetting to set `spring.ai.model.chat: none` when using two+ starters → Spring may still try to pick a single ambiguous default and error at startup.
- ❌ Forgetting `@Qualifier` when injecting `ChatClient` in a class with multiple `ChatClient` beans defined → Spring throws a `NoUniqueBeanDefinitionException` unless one bean is marked `@Primary`.
- ✅ Mark exactly one `ChatClient` bean `@Primary` if you want a sensible default for places that don't specify a qualifier.
- ✅ Keep each provider's properties under its own prefix (`spring.ai.openai.*`, `spring.ai.ollama.*`, `spring.ai.anthropic.*`) — they don't conflict with each other even when configured simultaneously.
- ✅ For same-provider multi-model setups, reuse the auto-configured `*Api` bean rather than re-declaring API keys/connections from scratch.

## 9. Official Documentation

📖 **Chat Client API — Multiple Chat Models section:** https://docs.spring.io/spring-ai/reference/api/chatclient.html

📖 **Chat Models overview:** https://docs.spring.io/spring-ai/reference/api/chatmodel.html

📖 **OpenAI Chat:** https://docs.spring.io/spring-ai/reference/api/chat/openai-chat.html

📖 **Ollama Chat:** https://docs.spring.io/spring-ai/reference/api/chat/ollama-chat.html

📖 **Anthropic Chat:** https://docs.spring.io/spring-ai/reference/api/chat/anthropic-chat.html

---
*Personal learning notes — configuring and using multiple AI models (OpenAI, Ollama, and same-provider variants) side by side in one Spring Boot application, with routing and fallback patterns.*
