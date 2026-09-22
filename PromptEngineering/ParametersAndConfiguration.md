# 2. Parameters & Configuration

There are really two separate kinds of "configuration" people lump together, and keeping them mentally separate makes everything easier:

1. **Connection properties** — how to reach the provider at all (API key, base URL, which organization/project).
2. **Model/request options** — how the model should behave for a given call (temperature, max tokens, which specific model, etc.).

## Connection properties

For OpenAI (the pattern is nearly identical for Anthropic, Azure, etc. — just swap the prefix):

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      base-url: https://api.openai.com   # override only if using a proxy/gateway
      organization-id: org-xxxx           # optional, for multi-org accounts
      project-id: proj-xxxx               # optional
```

Never hardcode the raw key in `application.yml`. The standard pattern is to reference an environment variable via Spring's `${...}` placeholder syntax, and set the real value outside your codebase:

```bash
export OPENAI_API_KEY=sk-xxxxxxxx
```

You can also read it programmatically if you're pulling it from somewhere custom (like a secrets manager) instead of a plain env var:

```java
String apiKey = System.getenv("OPENAI_API_KEY");
```

In real production systems, that env var itself usually isn't sitting in a `.env` file on a server — it's injected at deploy time from something like AWS Secrets Manager, HashiCorp Vault, or a Kubernetes `Secret`, so the key never touches source control or plaintext config at all.

## Model/request options — the full list

These are the parameters that actually shape *how* the model generates text. Spring AI exposes a portable `ChatOptions` interface (works across providers) and a provider-specific extension (`OpenAiChatOptions`, `AnthropicChatOptions`, etc.) that adds provider-only knobs.

| Parameter | What it does | Typical default | Notes |
|---|---|---|---|
| `model` | Which specific model to call (e.g. `gpt-4o`, `gpt-4o-mini`) | provider default | The single biggest lever for cost — see file 5 |
| `temperature` | Randomness/creativity of output. Higher = more varied, lower = more focused and deterministic | ~0.7–0.8 | Don't tune this *and* `topP` together — their interaction is unpredictable |
| `topP` | Nucleus sampling — only consider tokens whose cumulative probability adds up to `topP` | 1.0 | Alternative to temperature, not a complement to it |
| `maxTokens` / `maxCompletionTokens` | Hard cap on how many tokens the response can contain | none (provider max) | `maxTokens` is deprecated in the OpenAI options in favor of `maxCompletionTokens`; total cost scales with this |
| `n` | How many completion choices to generate for one input | 1 | You're billed for every choice generated — keep at 1 unless you specifically need multiple candidates |
| `frequencyPenalty` | Penalizes tokens based on how often they've *already appeared* in the text so far (range -2.0 to 2.0) | 0.0 | Higher values discourage the model from repeating the same phrase verbatim |
| `presencePenalty` | Penalizes tokens that have appeared *at all*, regardless of frequency (range -2.0 to 2.0) | 0.0 | Higher values push the model toward introducing new topics |
| `stop` | A list of strings that, if generated, immediately stop the response | none | Useful for cutting off structured output cleanly (e.g. stop at `"###"`) |
| `seed` | Fixes the random seed for more reproducible output | none | Doesn't guarantee identical output every time, but reduces variance — useful for testing |
| `logitBias` | Manually boosts or suppresses the likelihood of specific tokens | none | Rarely used directly; mostly for very specific constraint problems |
| `responseFormat` | Requests `JSON_OBJECT` mode or full `JSON_SCHEMA` structured outputs | text | Covered in depth in file 4 — this is the provider-side guarantee that complements Spring AI's own output converters |
| `toolChoice` / `tools` | Which tools the model is allowed/forced to call | none | Covered in file 4 |

## Where to actually set these

**Option A — build a `Prompt` object with options attached**, when you need per-call control:

```java
Prompt prompt = new Prompt(
    userText,
    OpenAiChatOptions.builder()
        .model("gpt-4o-mini")
        .temperature(0.3)
        .maxTokens(300)
        .build()
);

String answer = chatClient.prompt(prompt).call().content();
```

**Option B — inline in the fluent chain**, when you're already using `.prompt().user(...)`:

```java
String answer = chatClient.prompt()
        .user("Summarize this contract")
        .options(OpenAiChatOptions.builder().temperature(0.2).build())
        .call()
        .content();
```

**Option C — set defaults once at the `ChatClient` bean level**, so every call inherits them unless overridden:

```java
@Bean
ChatClient chatClient(ChatClient.Builder builder) {
    return builder
        .defaultOptions(OpenAiChatOptions.builder().temperature(0.5).build())
        .build();
}
```

**Option D — set everything through `application.yml`**, which becomes the default for the auto-configured `ChatModel`:

```yaml
spring:
  ai:
    openai:
      chat:
        options:
          model: gpt-4o-mini
          temperature: 0.7
          max-completion-tokens: 500
          frequency-penalty: 0.0
          presence-penalty: 0.0
```

A good mental rule: **use properties files for the boring stable defaults** (which model, general temperature) and **use runtime options for anything that genuinely varies per request** (like a lower temperature specifically for a JSON-extraction call versus a higher one for a creative-writing call in the same app).

## Portable `ChatOptions` vs. provider-specific options

`ChatOptions` (the interface) covers the parameters that are common across nearly every provider — temperature, `topP`, `maxTokens`, stop sequences. If you build your prompt options against this interface instead of `OpenAiChatOptions` directly, swapping providers later (OpenAI → Anthropic → a local Ollama model) doesn't force you to rewrite your option-building code. You lose access to provider-only fields (like OpenAI's `logitBias`), but you gain real portability. This is the same tradeoff you'd make choosing a portable Spring Data repository interface over a vendor-specific one — pick portability unless you have a concrete reason not to.

## Retry configuration

Calls to any external LLM API can transiently fail (rate limits, network blips), so Spring AI wraps calls in a configurable retry policy:

```yaml
spring:
  ai:
    retry:
      max-attempts: 10
      backoff:
        initial-interval: 2s
        multiplier: 5
        max-interval: 3m
      on-client-errors: false     # if false, 4xx errors are NOT retried
      on-http-codes: []            # specific HTTP codes to force a retry on
      exclude-on-http-codes: []    # specific HTTP codes to never retry
```

The default backoff is exponential — it waits longer between each retry attempt — which is the right shape for rate-limit-style failures where hammering the API immediately again just makes things worse.

## Observability configuration

Spring AI integrates with Micrometer/OpenTelemetry, so every `ChatClient` call and every advisor in the chain can be traced. This needs `spring-boot-starter-actuator` on the classpath, plus:

```yaml
spring:
  ai:
    chat:
      client:
        observations:
          log-prompt: false      # true will log full prompt text — careful with sensitive data
          log-completion: false
```

These default to `false` on purpose — logging full prompts/completions is convenient for debugging but risks leaking sensitive user data into your logs, so it's an explicit opt-in. File 5 goes deeper into how this plugs into a full observability setup.
