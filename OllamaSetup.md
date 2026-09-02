# Spring AI + Ollama — End-to-End Setup & Configuration Guide

> Note: I wasn't able to pull the transcript from the YouTube video directly (YouTube rate-limited the fetch), so this guide is built straight from the **official Spring AI reference docs** for Ollama — which is the same material tutorials like that one walk through. Every property and step below is verified against the current docs, not guessed.

## 1. What Is Ollama, and Why Pair It With Spring AI?

Ollama is a tool that lets you **run open-source LLMs (Llama, Mistral, Gemma, DeepSeek, Qwen, etc.) locally on your own machine**, and exposes them through a simple local HTTP API. Spring AI has a dedicated integration for it — `OllamaChatModel` — so you can plug a locally running model into your Spring Boot app the exact same way you'd plug in OpenAI or Anthropic, just pointed at `localhost` instead of a paid cloud API.

**Why people use it:** free, private (nothing leaves your machine), great for development/testing without burning API credits.

## 2. Step 1 — Install & Run Ollama Itself (Outside of Spring)

Before touching any Java code, you need a running Ollama server.

1. **Download Ollama** from https://ollama.com/download (Windows/Mac/Linux all supported).
2. After installing, Ollama runs as a background service on `http://localhost:11434` by default.
3. **Pull a model** from the command line:
   ```bash
   ollama pull llama3.2
   ```
   You can pull any model from the Ollama library: https://ollama.com/library — common choices are `llama3.2`, `mistral`, `gemma`, `qwen2.5`, `deepseek-r1`.
4. (Optional) You can also run any GGUF model hosted on Hugging Face directly:
   ```bash
   ollama pull hf.co/<username>/<model-repository>
   ```
5. Verify it's working by testing it directly in the terminal:
   ```bash
   ollama run llama3.2
   ```
   If you get a prompt where you can chat with it, Ollama itself is working correctly — now it's time to wire up Spring.

> Alternative: you can run Ollama via **Testcontainers** instead of installing it locally, which is useful for automated integration tests.

## 3. Step 2 — Add the Spring AI Ollama Starter Dependency

Maven (`pom.xml`):

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-ollama</artifactId>
</dependency>
```

Gradle (`build.gradle`):

```groovy
dependencies {
    implementation 'org.springframework.ai:spring-ai-starter-model-ollama'
}
```

> Don't forget the Spring AI **BOM** (Bill of Materials) in your dependency management section — it keeps all Spring AI module versions consistent, the same as you'd do for Spring Cloud.

This single starter is what triggers the whole auto-configuration journey you already learned: it puts `OllamaChatModel` and its auto-configuration class on the classpath, which Spring Boot then activates.

## 4. Step 3 — Configure `application.yml`

Minimal working config:

```yaml
spring:
  ai:
    ollama:
      base-url: http://localhost:11434
      chat:
        options:
          model: llama3.2
          temperature: 0.7
```

That's genuinely enough to get a working `ChatModel` bean. Compare this to your NVIDIA/OpenAI config from earlier — **notice there's no `api-key` property at all.** That's the single biggest structural difference: Ollama has nothing to authenticate against, because you're the one running the server.

## 5. Full Property Reference — What Each One Actually Does

### a) Base connection properties (prefix: `spring.ai.ollama`)

| Property | What it does | Default |
|---|---|---|
| `spring.ai.ollama.base-url` | The URL where your Ollama server is running | `http://localhost:11434` |

Only change this if Ollama is running on a different host/port (e.g., a remote server, a Docker container, or Testcontainers).

### b) Model auto-pull properties (prefix: `spring.ai.ollama.init`)

These control whether Spring Boot should automatically download a model at application startup if it isn't already present locally — genuinely useful so you don't have to manually run `ollama pull` before every environment setup.

| Property | What it does | Default |
|---|---|---|
| `spring.ai.ollama.init.pull-model-strategy` | Whether/how to pull models at startup. Options: `always`, `when_missing`, `never` | `never` |
| `spring.ai.ollama.init.timeout` | How long to wait for the pull to finish before giving up | `5m` |
| `spring.ai.ollama.init.max-retries` | Retry attempts if the pull fails | `0` |
| `spring.ai.ollama.init.chat.include` | Whether chat models are included in this auto-pull behavior | `true` |
| `spring.ai.ollama.init.chat.additional-models` | Extra models to pull besides your configured default (useful if you switch models at runtime) | `[]` |

Example — always ensure the latest version of the model is pulled at startup, with a longer timeout:

```yaml
spring:
  ai:
    ollama:
      init:
        pull-model-strategy: always
        timeout: 60s
        max-retries: 1
        chat:
          additional-models:
            - llama3.2
            - qwen2.5
```

> ⚠️ **Production note (straight from the docs):** auto-pulling is discouraged in production because downloading a multi-GB model at startup can seriously delay boot time. Pre-download models manually in production and reserve `always`/`when_missing` for local dev.

### c) Core chat request properties (prefix: `spring.ai.ollama.chat`)

| Property | What it does | Default |
|---|---|---|
| `spring.ai.ollama.chat.model` | Which model to use for chat | `mistral` |
| `spring.ai.ollama.chat.format` | Force the response into `"json"` or a JSON Schema object | *(none)* |
| `spring.ai.ollama.chat.keep_alive` | How long the model stays loaded in memory after a request (avoids reload delay on the next call) | `5m` |
| `spring.ai.ollama.chat.think` | Whether the model should emit its internal reasoning before the final answer (for reasoning models) | *(none)* |

> There used to be `spring.ai.ollama.chat.enabled` to toggle the chat model on/off — this has been **replaced** by the top-level `spring.ai.model.chat` property (same one you already used for OpenAI): set `spring.ai.model.chat=ollama` to enable it (default) or `spring.ai.model.chat=none` to disable it — this exists specifically to support multi-model setups.

### d) Fine-grained model behavior options (also under `spring.ai.ollama.chat.*`)

These map directly to Ollama's own model runtime parameters — this is the part that's genuinely unique to running a model locally, since a cloud API would never expose this level of control:

| Property | What it does | Default |
|---|---|---|
| `temperature` | Creativity/randomness of output | `0.8` |
| `top-k` | Limits token choice pool — lower = more conservative | `40` |
| `top-p` | Works with top-k for diversity control | `0.9` |
| `num-ctx` | Size of the context window (how much text the model can "remember" per request) | `2048` |
| `num-predict` | Max tokens to generate (`-1` = unlimited) | `-1` |
| `num-gpu` | How many model layers to offload to GPU | `-1` (auto) |
| `num-thread` | CPU threads used for inference | `0` (auto-detect) |
| `seed` | Fixed seed = reproducible output for the same prompt | `-1` (random) |
| `repeat-penalty` | Penalizes the model for repeating itself | `1.1` |
| `stop` | Sequences that make generation stop immediately | *(none)* |
| `mirostat` | Advanced sampling algorithm for controlling output "perplexity" (`0`=off, `1`/`2`=variants) | `0` |

Example — a more deterministic, tightly-controlled local setup:

```yaml
spring:
  ai:
    ollama:
      chat:
        options:
          model: llama3.2
          temperature: 0.2
          top-p: 0.8
          num-ctx: 4096
          seed: 42
          repeat-penalty: 1.2
```

**When you'd touch these:** `num-ctx` when you need longer conversations/documents in context, `num-gpu`/`num-thread` when tuning performance for your specific hardware, `seed` when you need reproducible test output, `temperature`/`top-p`/`top-k` for creativity control (same idea as any other provider).

## 6. Step 4 — Inject and Use It in Code

Following the same `ChatClient.Builder` pattern from before:

```java
@RestController
public class ChatController {

    private final ChatClient chatClient;

    public ChatController(ChatClient.Builder chatClientBuilder) {
        this.chatClient = chatClientBuilder.build();
    }

    @GetMapping("/ai/generate")
    public String generate(@RequestParam String message) {
        return chatClient.prompt()
                .user(message)
                .call()
                .content();
    }
}
```

Or, if you want the lower-level `ChatModel` directly (useful for streaming or custom control):

```java
@RestController
public class ChatController {

    private final OllamaChatModel chatModel;

    public ChatController(OllamaChatModel chatModel) {
        this.chatModel = chatModel;
    }

    @GetMapping("/ai/generate")
    public Map<String, String> generate(
            @RequestParam(defaultValue = "Tell me a joke") String message) {
        return Map.of("generation", chatModel.call(message));
    }

    @GetMapping(value = "/ai/generateStream")
    public Flux<ChatResponse> generateStream(
            @RequestParam(defaultValue = "Tell me a joke") String message) {
        return chatModel.stream(new Prompt(new UserMessage(message)));
    }
}
```

Both work — `ChatClient` if you want the fluent, advisor-friendly API; `OllamaChatModel` directly if you want raw access (e.g., for streaming with `Flux`).

## 7. Bonus — Using the OpenAI Client Against Ollama Instead

Here's a neat trick worth knowing since you already understand the OpenAI-compatibility pattern from your NVIDIA setup: **Ollama itself exposes an OpenAI-compatible endpoint**, so you can skip the Ollama starter entirely and use the OpenAI starter pointed at your local Ollama server:

```yaml
spring:
  ai:
    openai:
      base-url: http://localhost:11434/v1
      api-key: ollama        # Ollama doesn't check this, but the property still expects a non-empty value
      chat:
        options:
          model: mistral
```

This is exactly the same trick as your NVIDIA config — different server, same underlying compatibility mechanism. The upside: you get access to `extraBody` for passing Ollama-specific params (`top_k`, `repeat_penalty`, `num_predict`) through the OpenAI client. The downside: you lose Ollama-native features like the dedicated `think` boolean and some of the low-level tuning properties listed above (unless passed manually via `extraBody`).

## 8. Multimodal (Image) Input

Some Ollama models (like `llava`) support images. Spring AI's `Media` type handles this:

```java
var imageResource = new ClassPathResource("/photo.png");

var userMessage = new UserMessage(
    "Explain what do you see on this picture?",
    new Media(MimeTypeUtils.IMAGE_PNG, imageResource));

ChatResponse response = chatModel.call(new Prompt(userMessage,
    OllamaChatOptions.builder().model(OllamaModel.LLAVA).build()));
```

## 9. Thinking Mode (Reasoning Models)

Models like `deepseek-r1` and `qwen3` can expose their reasoning process before the final answer:

```java
ChatResponse response = chatModel.call(
    new Prompt("How many letter 'r' are in the word 'strawberry'?",
        OllamaChatOptions.builder()
            .model("deepseek-r1")
            .enableThinking()
            .build()));

String thinking = response.getResult().getMetadata().get("thinking");
String answer = response.getResult().getOutput().getText();
```

## 10. Quick Troubleshooting Checklist

- **Connection refused** → Ollama isn't running, or `base-url` doesn't match the port it's actually running on. Run `ollama serve` or check the Ollama tray icon/service.
- **Model not found** → You haven't pulled it yet (`ollama pull <model>`), or `pull-model-strategy` is `never` and it can't auto-fetch it.
- **Very slow first response** → Normal — Ollama loads the model into memory on first use. `keep_alive` controls how long it stays loaded afterward so subsequent calls are fast.
- **Out of memory / crashes** → The model is too large for your RAM/VRAM; try a smaller model (e.g., a `-mini` or smaller parameter-count variant) or reduce `num-ctx`.

## 11. Official Documentation

📖 **Ollama Chat (Spring AI reference):** https://docs.spring.io/spring-ai/reference/api/chat/ollama-chat.html

📖 **Ollama download:** https://ollama.com/download

📖 **Ollama model library:** https://ollama.com/library

📖 **Getting Started with Spring AI:** https://docs.spring.io/spring-ai/reference/getting-started.html

---
*Personal learning notes — end-to-end Ollama setup with Spring AI, covering install → dependency → properties → code → troubleshooting.*
