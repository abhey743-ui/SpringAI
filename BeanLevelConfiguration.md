# 07 — Bean-Level Configuration: ChatClient, ChatModel, and Options as Spring Beans

Everything so far showed options set inline, per-request. This file is about
the **other** place things get configured: `@Configuration` classes, where you
wire up `ChatClient`, `ChatModel`, and `ChatOptions` as proper Spring beans so
your controllers/services stay thin and your defaults live in one place.

## 1. The three things you can turn into beans

| Bean type | What it is | Why make it a bean |
|---|---|---|
| `ChatClient.Builder` | Auto-configured by Spring Boot (prototype scope) | You rarely define this yourself — you *consume* it to build `ChatClient` beans |
| `ChatClient` | The thing you actually inject into controllers/services | Bake in system prompt, default options, default tools, default advisors |
| `ChatModel` (e.g. `OpenAiChatModel`) | Lower-level model connection | Needed when you want multiple providers, custom HTTP client behavior, or a non-default `ApiKey` |
| `ChatOptions` / `OpenAiChatOptions` | The generation-parameter object itself | Reuse the same "preset" (e.g. "creative", "strict-json") across multiple call sites |

## 2. The simplest bean: one `ChatClient` with baked-in defaults

This is the pattern that replaces scattering `.system(...)` / `.options(...)`
across every controller method:

```java
@Configuration
public class ChatClientConfig {

    @Bean
    public ChatClient chatClient(ChatClient.Builder builder) {
        return builder
            .defaultSystem("You are a concise, helpful assistant. Answer in plain text.")
            .defaultOptions(ChatOptions.builder()
                .model("gpt-4o-mini")
                .temperature(0.4)
                .maxTokens(500)
                .build())
            .build();
    }
}
```

```java
@RestController
public class ChatController {

    private final ChatClient chatClient; // fully configured, ready to use

    public ChatController(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    @GetMapping("/chat")
    public String chat(@RequestParam String q) {
        return chatClient.prompt(q).call().content(); // system + options already applied
    }
}
```

Notice the constructor now takes `ChatClient` directly, not
`ChatClient.Builder` — because the `@Bean` method already called `.build()`.
This is the cleanest pattern for a single-purpose app.

## 3. `ChatOptions` as its own reusable bean

Useful when several different `ChatClient`s — or several different call sites
on the same client — should share one "preset."

```java
@Configuration
public class ChatOptionsConfig {

    @Bean
    public ChatOptions creativeOptions() {
        return ChatOptions.builder()
            .temperature(0.9)
            .maxTokens(1000)
            .build();
    }

    @Bean
    public ChatOptions strictJsonOptions() {
        return OpenAiChatOptions.builder()
            .temperature(0.0)
            .maxTokens(300)
            .responseFormat(new ResponseFormat(ResponseFormat.Type.JSON_OBJECT))
            .build();
    }
}
```

```java
@Service
public class ExtractionService {

    private final ChatClient chatClient;
    private final ChatOptions strictJsonOptions;

    public ExtractionService(ChatClient chatClient,
                              @Qualifier("strictJsonOptions") ChatOptions strictJsonOptions) {
        this.chatClient = chatClient;
        this.strictJsonOptions = strictJsonOptions;
    }

    public String extract(String text) {
        return chatClient.prompt(text)
            .options(strictJsonOptions)   // reused, defined once
            .call()
            .content();
    }
}
```

> Since both beans are of type `ChatOptions`, you need `@Qualifier("beanName")`
> (or field/param name matching) wherever you inject one specifically —
> otherwise Spring can't disambiguate.

## 4. Multiple `ChatClient` beans (different personas / tasks / models)

A very common real-world shape: one cheap/fast client for simple tasks, one
stronger client for hard ones, each pre-configured.

```java
@Configuration
public class MultiChatClientConfig {

    @Bean
    @Primary
    public ChatClient fastChatClient(ChatClient.Builder builder) {
        return builder
            .defaultOptions(ChatOptions.builder()
                .model("gpt-4o-mini")
                .temperature(0.2)
                .maxTokens(200)
                .build())
            .build();
    }

    @Bean
    public ChatClient reasoningChatClient(ChatClient.Builder builder) {
        return builder
            .defaultSystem("Think carefully step by step before answering.")
            .defaultOptions(OpenAiChatOptions.builder()
                .model("o1-preview")
                .maxCompletionTokens(2000) // reasoning models use this, not maxTokens
                .build())
            .build();
    }
}
```

```java
@Service
public class TriageService {

    private final ChatClient fastChatClient;
    private final ChatClient reasoningChatClient;

    public TriageService(ChatClient fastChatClient,
                          @Qualifier("reasoningChatClient") ChatClient reasoningChatClient) {
        this.fastChatClient = fastChatClient;       // @Primary, so unqualified injection works
        this.reasoningChatClient = reasoningChatClient;
    }
}
```

`@Primary` marks the default when a bean of that type is injected without a
qualifier; everything else needs `@Qualifier("beanName")`.

## 5. Multiple *providers* as beans (OpenAI + Anthropic side by side)

Once you have more than one `ChatModel` bean type in the context (e.g. both
`spring-ai-starter-model-openai` and `spring-ai-starter-model-anthropic` on
the classpath), building a `ChatClient` the naive way
(`ChatClient.builder(chatModel)`) **bypasses observability and any
`ChatClientBuilderCustomizer` beans**. Instead, inject
`ChatClientBuilderConfigurer` and route through it:

```java
@Configuration
public class MultiProviderChatClientConfig {

    @Bean
    @Primary
    public ChatClient openAiChatClient(OpenAiChatModel chatModel,
                                        ChatClientBuilderConfigurer configurer,
                                        ObjectProvider<ObservationRegistry> observationRegistry,
                                        ObjectProvider<ChatClientObservationConvention> convention,
                                        ObjectProvider<AdvisorObservationConvention> advisorConvention,
                                        ObjectProvider<ToolCallingAdvisor.Builder<?>> toolCallingAdvisorBuilder) {
        return build(chatModel, configurer, observationRegistry, convention, advisorConvention, toolCallingAdvisorBuilder);
    }

    @Bean
    public ChatClient anthropicChatClient(AnthropicChatModel chatModel,
                                           ChatClientBuilderConfigurer configurer,
                                           ObjectProvider<ObservationRegistry> observationRegistry,
                                           ObjectProvider<ChatClientObservationConvention> convention,
                                           ObjectProvider<AdvisorObservationConvention> advisorConvention,
                                           ObjectProvider<ToolCallingAdvisor.Builder<?>> toolCallingAdvisorBuilder) {
        return build(chatModel, configurer, observationRegistry, convention, advisorConvention, toolCallingAdvisorBuilder);
    }

    private ChatClient build(ChatModel chatModel, ChatClientBuilderConfigurer configurer,
                              ObjectProvider<ObservationRegistry> observationRegistry,
                              ObjectProvider<ChatClientObservationConvention> convention,
                              ObjectProvider<AdvisorObservationConvention> advisorConvention,
                              ObjectProvider<ToolCallingAdvisor.Builder<?>> toolCallingAdvisorBuilder) {
        ChatClient.Builder builder = ChatClient.builder(chatModel,
            observationRegistry.getIfUnique(() -> ObservationRegistry.NOOP),
            convention.getIfUnique(), advisorConvention.getIfUnique(),
            toolCallingAdvisorBuilder.getIfAvailable());
        return configurer.configure(builder).build();
    }
}
```

Then inject each by qualifier where you need a specific vendor, e.g. letting a
user pick their preferred model at runtime.

## 6. Customizing the underlying `ChatModel` bean itself

Sometimes the thing you need to configure isn't the `ChatClient`, it's the
model connection underneath it — custom `ApiKey`, custom `HttpClient`
behavior, observability wiring, connection-pool metrics, a custom executor for
virtual threads.

```java
@Configuration
public class OpenAiModelConfig {

    @Bean
    public OpenAiChatModel openAiChatModel(ObservationRegistry observationRegistry,
                                            MeterRegistry meterRegistry) {
        return OpenAiChatModel.builder()
            .options(OpenAiChatOptions.builder()
                .apiKey(System.getenv("OPENAI_API_KEY"))
                .model("gpt-4o")
                .temperature(0.4)
                .build())
            .observationRegistry(observationRegistry)
            .meterRegistry(meterRegistry)
            .dispatcherExecutor(Executors.newVirtualThreadPerTaskExecutor())
            .build();
    }
}
```

A rotating/secrets-manager-backed API key as its own bean:

```java
@Bean
public ApiKey openAiApiKey(SecretsClient secretsClient) {
    return () -> secretsClient.getCurrentSecret("openai-api-key");
}
```

### `OpenAiHttpClientBuilderCustomizer` beans

For intercepting the raw HTTP client (auth headers, logging interceptors,
proxy/SSL config) that's shared by *every* OpenAI model type (chat, embedding,
image, audio) in the app:

```java
@Bean
public OpenAiHttpClientBuilderCustomizer requestLoggingCustomizer() {
    return builder -> builder.addInterceptor(new LoggingInterceptor());
}
```

Multiple customizer beans are applied in `@Order`/`Ordered` order, after
Spring AI's own defaults — so your customization always wins if it conflicts.

## 7. `ChatClientBuilderCustomizer` — globally tweak every auto-configured builder

If you want *every* auto-configured `ChatClient.Builder` in the app to get a
default (e.g., always register a logging advisor) without redefining every
`ChatClient` bean by hand:

```java
@Bean
public ChatClientBuilderCustomizer loggingCustomizer() {
    return builder -> builder.defaultAdvisors(new SimpleLoggerAdvisor());
}
```

## 8. Advisors and tools as beans (so they're reusable and testable)

```java
@Configuration
public class AdvisorConfig {

    @Bean
    public ChatClient chatClient(ChatClient.Builder builder,
                                  ChatMemory chatMemory,
                                  VectorStore vectorStore) {
        return builder
            .defaultAdvisors(
                MessageChatMemoryAdvisor.builder(chatMemory).build(),
                QuestionAnswerAdvisor.builder(vectorStore).build(),
                new SimpleLoggerAdvisor()
            )
            .build();
    }

    @Bean
    public ChatMemory chatMemory(ChatMemoryRepository repository) {
        return MessageWindowChatMemory.builder()
            .chatMemoryRepository(repository)
            .maxMessages(20)
            .build();
    }

    @Bean
    public ChatMemoryRepository chatMemoryRepository() {
        return new InMemoryChatMemoryRepository(); // swap for Jdbc/Redis/Mongo/etc. in prod
    }
}
```

Registering default tools at the bean level so every request from this client
has them available without repeating `.tools(...)` everywhere:

```java
@Bean
public ChatClient chatClient(ChatClient.Builder builder, DateTimeTools dateTimeTools) {
    return builder.defaultTools(dateTimeTools).build();
}
```

### Customizing the auto-registered `ToolCallingAdvisor`

```java
@Bean
public ToolCallingAdvisor.Builder<?> toolCallingAdvisorBuilder(ToolCallingManager myToolCallingManager) {
    return ToolCallingAdvisor.builder()
        .toolCallingManager(myToolCallingManager)
        .advisorOrder(Ordered.LOWEST_PRECEDENCE);
}
```
Because Spring Boot's auto-configured bean is annotated
`@ConditionalOnMissingBean`, declaring your own `ToolCallingAdvisor.Builder<?>`
bean automatically takes precedence — no exclusions needed.

## 9. A realistic, complete `@Configuration` class

Putting it together — one config class that: sets a default persona, bakes in
a cost-conscious default options preset, wires chat memory, registers a
logging advisor only when a `debug` profile is active, and exposes a second,
"strict JSON" client for a specific extraction feature.

```java
@Configuration
public class AiConfig {

    @Bean
    public ChatMemory chatMemory() {
        return MessageWindowChatMemory.builder()
            .chatMemoryRepository(new InMemoryChatMemoryRepository())
            .maxMessages(20)
            .build();
    }

    @Bean
    @Primary
    public ChatClient chatClient(ChatClient.Builder builder, ChatMemory chatMemory) {
        return builder
            .defaultSystem("You are a helpful, concise product-support assistant.")
            .defaultOptions(ChatOptions.builder()
                .model("gpt-4o-mini")
                .temperature(0.4)
                .maxTokens(400)
                .build())
            .defaultAdvisors(MessageChatMemoryAdvisor.builder(chatMemory).build())
            .build();
    }

    @Bean
    @Profile("debug")
    public ChatClientBuilderCustomizer debugLoggingCustomizer() {
        return builder -> builder.defaultAdvisors(new SimpleLoggerAdvisor());
    }

    @Bean
    public ChatClient extractionChatClient(ChatClient.Builder builder) {
        return builder
            .defaultOptions(OpenAiChatOptions.builder()
                .model("gpt-4o-mini")
                .temperature(0.0)
                .maxTokens(300)
                .responseFormat(new ResponseFormat(ResponseFormat.Type.JSON_OBJECT))
                .build())
            .build();
    }
}
```

```java
@Service
public class SupportService {

    private final ChatClient chatClient;               // @Primary
    private final ChatClient extractionChatClient;      // @Qualifier by param name

    public SupportService(ChatClient chatClient,
                           @Qualifier("extractionChatClient") ChatClient extractionChatClient) {
        this.chatClient = chatClient;
        this.extractionChatClient = extractionChatClient;
    }
}
```

## 10. Rules of thumb for where configuration should live

| Put it here | When |
|---|---|
| `application.yml` property | Static, environment-dependent (API key, base URL, default model/temperature) |
| `ChatOptions`/`OpenAiChatOptions` bean | A reusable "preset" shared by multiple call sites or clients |
| `ChatClient.Builder.defaultOptions/defaultSystem/defaultAdvisors` in a `@Bean` | Persona/behavior that should apply to *every* call from one specific client |
| `.options(...)`/`.system(...)` inline on a request | A one-off override for a single call, e.g. user-selected "creative mode" toggle |
| `ChatModel` bean | Custom connection behavior: API key rotation, custom HTTP client, multiple providers |

This keeps the pattern from your original controller — six different inline
variants crammed into one method — out of runtime code entirely: the
"variants" become named beans (`fastChatClient`, `extractionChatClient`,
`reasoningChatClient`), and the controller just picks which one to call.
