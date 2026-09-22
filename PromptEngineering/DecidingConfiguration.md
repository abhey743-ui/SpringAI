# 9. Where Should Config Actually Live? Properties vs. `@Bean` vs. Runtime

This is the question that ties everything from files 1–8 together, so it deserves its own file. You've now seen the same thing configurable in what looks like three different places — `application.yml`, a `@Bean` method, and inline inside a controller/service method — and it's genuinely confusing until you see the pattern underneath it.

## The three levels, plainly

| Level | Where it lives | When it's decided | Changes without a redeploy? |
|---|---|---|---|
| **1. Properties** | `application.yml` / `application.properties` (or env vars feeding into them) | At application startup | Yes, if externalized (env var, config server) |
| **2. Bean defaults** | A `@Configuration` class, building the `ChatClient` (or `ChatMemory`, or an `Advisor`) once | At application startup | No — it's compiled into the running bean |
| **3. Runtime / business logic** | Inline in the fluent chain, inside a `@Service` or `@RestController` method | Per individual request | N/A — it's decided fresh every call |

The reason all three exist isn't redundancy — **each one answers a different question:**

- **Properties answer: "What's true for this *environment* (dev, staging, prod)?"** — API keys, base URLs, retry policy, which model to talk to by default.
- **Bean defaults answer: "What's true for every call this particular `ChatClient` makes, regardless of who's calling it?"** — its persona/system prompt, which advisors it always runs, its baseline temperature.
- **Runtime answers: "What's true for *this one request, from this one user, right now*?"** — which conversation ID, a document filter scoped to this tenant, a lower temperature specifically because this one call needs deterministic JSON.

**The decision rule that resolves 90% of the confusion:** ask *"does this value depend on who's calling, or on what they're asking for, right now?"* If yes → runtime. If no, but it's specific to *this application's role* (e.g. "this ChatClient is always our customer-support bot") → bean default. If it's about *infrastructure* (credentials, which environment you're in) → properties.

## How the three levels actually combine — it's a merge, not a replace

This is the part most tutorials skip, and it's important: Spring AI doesn't just let the "most specific" level win outright and throw the rest away. For `ChatOptions`, the framework **merges field by field** — runtime options are merged on top of the `ChatClient`'s bean-level defaults, which are themselves layered on top of the properties-driven defaults from the auto-configured `ChatModel`. If you only set `temperature` at runtime and leave `maxTokens` unset, the `maxTokens` value from the bean/properties level still applies — it doesn't get wiped out just because you touched a different field.

**One real gotcha worth knowing before you hit it yourself:** this clean merge behavior can break down for list-based fields like tool callbacks. There's a known, documented case where calling `.options(...)` inline on a specific request can unintentionally suppress tools that were registered as defaults on the `ChatClient` bean via `.defaultToolCallbacks(...)`, because the runtime options object didn't carry those tools forward. The lesson: **when you override options at runtime, be explicit about what you're overriding rather than assuming everything else is safely inherited** — especially for tools and advisors, verify the combined behavior with a test rather than assuming the merge "just works" in every case.

---

## Walking through every topic from files 1–8, level by level

### Connection credentials (API key, base URL)

**Always properties. Never bean, never runtime.** There's no legitimate reason for an API key to be decided per-request, and hardcoding it in a `@Bean` method means it's compiled into your source rather than swappable per environment.

```yaml
# application.yml — the only place this should live
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
```

### Which model to use, and baseline options (temperature, maxTokens, etc.)

**All three levels are legitimate here — they answer different questions.**

*Properties* — the environment-wide default (e.g., use the cheap model everywhere except where explicitly overridden):
```yaml
spring:
  ai:
    openai:
      chat:
        options:
          model: gpt-4o-mini
          temperature: 0.7
```

*Bean default* — this specific `ChatClient`'s persona always wants determinism (e.g., a JSON-extraction service):
```java
@Bean
ChatClient extractionClient(ChatClient.Builder builder) {
    return builder
        .defaultOptions(OpenAiChatOptions.builder().temperature(0.0).build())
        .build();
}
```

*Runtime* — one specific call needs something different from this client's usual behavior:
```java
@Service
public class SummaryService {
    private final ChatClient chatClient;

    public String summarize(String text, boolean creative) {
        return chatClient.prompt()
            .user(text)
            .options(OpenAiChatOptions.builder()
                .temperature(creative ? 0.9 : 0.2)
                .build())
            .call()
            .content();
    }
}
```

**Industry pattern:** properties set the safe, boring, org-wide default. Bean-level options define each `ChatClient`'s "personality" for its specific job (you'll typically have more than one `ChatClient` bean in a real app — one per distinct role). Runtime options are reserved for genuine per-request variability, like the `creative` flag above — not for things that are actually always true, which belong one level higher.

### The system prompt

*Bean default* — the normal, recommended home for this. A system prompt describes what this `ChatClient` *is*, which is an application-level fact, not a per-request one:
```java
@Bean
ChatClient supportBotClient(ChatClient.Builder builder) {
    return builder
        .defaultSystem("You are a support assistant for Acme Corp. Be concise and cite the docs you used.")
        .build();
}
```

*Runtime override* — legitimate only when a specific request genuinely needs a different persona than this client's default (rare — usually a sign you actually want a second `ChatClient` bean instead):
```java
chatClient.prompt()
    .system("For this one request, answer only in bullet points.")
    .user(userText)
    .call()
    .content();
```

**Industry pattern:** if you find yourself overriding the system prompt at runtime *often*, that's a signal to split it into two separate `ChatClient` beans with two separate default systems, rather than keep branching on flags inside one shared client. Keep the runtime override for genuinely rare, one-off cases.

### Advisors — this is the one that confuses people most

Advisors themselves (which ones are active) are a **bean-level decision**. Their **parameters for a specific call** are a **runtime decision**. Don't conflate the two.

*Bean default* — deciding *which* advisors this client always runs:
```java
@Bean
ChatClient chatClient(ChatClient.Builder builder, ChatMemory chatMemory, VectorStore vectorStore) {
    return builder
        .defaultAdvisors(
            MessageChatMemoryAdvisor.builder(chatMemory).build(),
            QuestionAnswerAdvisor.builder(vectorStore).build()
        )
        .build();
}
```

*Runtime* — supplying the per-request data those already-registered advisors need to do their job correctly:
```java
@Service
public class ChatService {
    private final ChatClient chatClient;

    public String ask(String userId, String question) {
        return chatClient.prompt()
            .user(question)
            .advisors(a -> a
                .param(ChatMemory.CONVERSATION_ID, userId)               // which conversation's memory to use
                .param(QuestionAnswerAdvisor.FILTER_EXPRESSION, "tenant == '" + userId + "'")) // scope the RAG search
            .call()
            .content();
    }
}
```

**Why it has to split this way:** the advisor *instance* (with its `ChatMemoryRepository`, its `VectorStore` connection) is expensive to construct and is genuinely the same for every user — building it once as a bean is correct. But the *conversation ID* and *filter expression* are obviously per-user, per-request facts that can't be known at startup. Registering the advisor at bean level and feeding it per-call parameters through `.advisors(a -> a.param(...))` is the only combination that makes sense — you'll rarely see a professional codebase constructing a brand-new `MessageChatMemoryAdvisor` instance inside a request handler.

*The one exception:* a custom advisor whose entire behavior genuinely depends on something only known per-request (e.g., which tenant's rules to enforce) can be constructed at runtime — but even then, the *pattern* is usually to make the advisor itself read from the shared per-request context (like the examples above) rather than build a whole new advisor object per call, since that's wasteful and harder to test.

### Retry and observability settings

**Properties, always.** These describe operational policy — how many times to retry, whether to log full prompts — and both are things that should differ *by environment* (you probably want `log-prompt: true` in a local dev profile and `false` in production) rather than by request or by which `ChatClient` bean is calling. There's no `.retry(...)` method in the fluent API for a reason — it's infrastructure-layer, not request-layer.

```yaml
# application-dev.yml
spring:
  ai:
    chat:
      client:
        observations:
          log-prompt: true

# application-prod.yml
spring:
  ai:
    chat:
      client:
        observations:
          log-prompt: false
```

---

## The cheat sheet

| Topic | Properties | Bean default | Runtime |
|---|---|---|---|
| API key / base URL | ✅ Always here | ❌ | ❌ |
| Default model / temperature for the whole app | ✅ Org-wide baseline | — | — |
| This client's "personality" options | — | ✅ Normal home | Only for genuine one-offs |
| System prompt | — | ✅ Normal home | Rare, one-off overrides only |
| Which advisors are active | — | ✅ Always here | ❌ (don't rebuild advisors per request) |
| Advisor parameters (conversation ID, filter expressions) | — | — | ✅ Always here |
| Retry policy, observability logging | ✅ Always here (per environment) | ❌ | ❌ |
| Per-call structured output type (`.entity(...)`) | — | — | ✅ Always here — it's inherently about *this* call's return type |

## The one-sentence version of this whole file

**Properties describe the environment you're running in. Bean defaults describe what a given `ChatClient` fundamentally *is*. Runtime code describes what *this specific request* needs.** Every confusing case resolves itself the moment you ask which of those three questions the value you're setting is actually answering.
