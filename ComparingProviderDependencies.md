# Spring AI — Comparing Provider Dependencies & Their Properties

You already know how the **OpenAI** provider works (starter dependency → auto-configuration → `ChatModel` bean → properties bound from `spring.ai.openai.*`). Every other provider follows the **exact same pattern** — the only things that change are:

1. Which starter dependency you add
2. Which property prefix (`spring.ai.<provider>.*`) you configure
3. Whether the provider needs a real API key, a self-hosted URL, or cloud credentials

This file compares the major providers side by side so you can see what's the same and what's different.

## 1. The Pattern That Never Changes

No matter which provider you pick, the flow is identical:

```
Add starter dependency
        │
Provider's ChatModel class + AutoConfiguration land on classpath
        │
Spring Boot binds spring.ai.<provider>.* properties into a Properties object
        │
<Provider>ChatModel bean created & registered
        │
ChatClientAutoConfiguration wraps it into ChatClient.Builder
        │
You inject ChatClient.Builder → .build() → ready to use
```

Only the **prefix** and **shape** of the properties differ between providers — the mechanism is 100% reused.

## 2. OpenAI (what you're already using)

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId>
</dependency>
```

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      base-url: https://api.openai.com/v1     # default — override for OpenAI-compatible endpoints
      chat:
        options:
          model: gpt-4o
          temperature: 0.7
          max-tokens: 1024
```

**When to use it:** Real OpenAI models, OR any provider that exposes an OpenAI-compatible API (NVIDIA NIM, Groq, Together AI, DeepInfra, LM Studio, vLLM, Azure via compatibility mode, etc. — you already saw this trick with NVIDIA).

## 3. Anthropic (Claude)

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-anthropic</artifactId>
</dependency>
```

```yaml
spring:
  ai:
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      base-url: https://api.anthropic.com     # default
      version: 2023-06-01                     # Anthropic API version header
      chat:
        options:
          model: claude-sonnet-4-20250514
          temperature: 0.7
          max-tokens: 1024
```

**Key differences from OpenAI:**
- Uses the **official Anthropic Java SDK** under the hood (not a hand-rolled HTTP client like some others)
- Has an extra `version` property — Anthropic's API is versioned via a request header, unlike OpenAI
- Options live under `chat.options.*` (model/temperature/max-tokens), same nesting style as OpenAI

**When to use it:** You want Claude models directly from Anthropic (not through Bedrock).

## 4. Ollama (local models)

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-ollama</artifactId>
</dependency>
```

```yaml
spring:
  ai:
    ollama:
      base-url: http://localhost:11434        # your local Ollama server, not a cloud endpoint
      chat:
        options:
          model: llama3.2
          temperature: 0.7
      init:
        pull-model-strategy: always            # auto-download the model on startup if missing
```

**Key differences from OpenAI/Anthropic:**
- **No `api-key` property at all** — there's nothing to authenticate, because you're talking to a model running on your own machine (or your own server) via Ollama's local server.
- `base-url` points at `localhost` by default (or wherever your Ollama instance runs) instead of a public cloud endpoint.
- Has a unique `init.pull-model-strategy` property — Ollama can pull the model automatically when your app starts, since models are files sitting on that machine, not something a cloud provider already has ready.
- Has a huge list of low-level model-loading options (`num-ctx`, `num-gpu`, `num-thread`, `seed`, etc.) because you're the one controlling the actual model runtime — a cloud API would never expose these, but a local inference engine does.

**When to use it:** Local development, privacy-sensitive workloads (data never leaves your machine), avoiding API costs while prototyping.

## 5. Azure OpenAI

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-azure-openai</artifactId>
</dependency>
```

```yaml
spring:
  ai:
    azure:
      openai:
        api-key: ${AZURE_OPENAI_API_KEY}
        endpoint: https://your-resource.openai.azure.com
        chat:
          options:
            deployment-name: gpt-4o            # NOTE: "deployment-name", not "model"
```

**Key differences from plain OpenAI:**
- Property prefix is `spring.ai.azure.openai.*`, a separate namespace from `spring.ai.openai.*` — these are two distinct starters/beans, even though the underlying model family is the same.
- Uses **`endpoint`** (your specific Azure resource URL) instead of a shared public `base-url`.
- You configure a **`deployment-name`** instead of a raw model name — because in Azure, you first "deploy" a model under a name of your choosing inside your Azure resource, and that deployment name is what you call, not the model ID directly.
- Auth is via Azure-style API keys or Azure AD/Entra ID credentials (enterprise identity integration), which OpenAI's own API doesn't offer.

**When to use it:** Enterprise environments already standardized on Microsoft Azure, needing Azure's compliance/security/billing integration.

## 6. Amazon Bedrock (multi-model gateway, incl. Claude/Titan/Llama)

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-bedrock-converse</artifactId>
</dependency>
```

```yaml
spring:
  ai:
    bedrock:
      aws:
        region: us-east-1
        access-key: ${AWS_ACCESS_KEY_ID}
        secret-key: ${AWS_SECRET_ACCESS_KEY}
      converse:
        chat:
          options:
            model: anthropic.claude-3-5-sonnet-20241022-v2:0
```

**Key differences:**
- No single `api-key` — instead uses **AWS credentials** (access key/secret key, or IAM roles), since Bedrock is an AWS service, not an independent AI company's API.
- Requires a **`region`** property — AWS services are region-scoped.
- One Bedrock integration gives you access to **many underlying model families** (Anthropic, Meta Llama, Amazon Titan, Mistral, etc.) — the `model` property is what picks which one, all through the same AWS-billed pipeline.

**When to use it:** Already on AWS, want unified billing/IAM/compliance across multiple model vendors without managing separate API keys for each.

## 7. Mistral AI

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-mistral-ai</artifactId>
</dependency>
```

```yaml
spring:
  ai:
    mistralai:
      api-key: ${MISTRAL_API_KEY}
      chat:
        options:
          model: mistral-large-latest
          temperature: 0.7
```

**Key differences:** Structurally almost identical to OpenAI/Anthropic (api-key + chat.options.model) — Mistral's own hosted API is a fairly standard REST API, so the integration shape mirrors OpenAI closely.

**When to use it:** Want Mistral's own models (efficient, often cheaper) directly from Mistral's cloud rather than through a gateway like Bedrock.

## 8. Google Vertex AI (Gemini)

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-vertex-ai-gemini</artifactId>
</dependency>
```

```yaml
spring:
  ai:
    vertex:
      ai:
        gemini:
          project-id: your-gcp-project-id
          location: us-central1
          chat:
            options:
              model: gemini-1.5-pro
```

**Key differences:**
- Auth is via **Google Cloud credentials** (a service account / Application Default Credentials), not a simple `api-key` string.
- Requires **`project-id`** and **`location`** — same idea as AWS's `region`, because Vertex AI is scoped to your GCP project and a specific data-center region.

**When to use it:** Already on Google Cloud, want Gemini models with GCP-native IAM/billing.

## 9. Any "OpenAI-Compatible" Provider (NVIDIA, Groq, Together AI, DeepInfra, etc.)

You don't get a dedicated starter for most of these — you reuse the **OpenAI starter** and just override `base-url`, exactly like you did with NVIDIA:

```yaml
spring:
  ai:
    openai:
      api-key: ${PROVIDER_API_KEY}
      base-url: https://<provider-specific-openai-compatible-endpoint>
      chat:
        options:
          model: <provider-specific-model-name>
```

**Why this works:** These providers deliberately mimic OpenAI's `/chat/completions` request/response format so existing OpenAI-client tooling (Spring AI, LangChain, etc.) works against them with zero extra integration code — only the URL, key, and model name change.

## 10. Side-by-Side Property Comparison

| Provider | Auth property | Extra required property | Model property location |
|---|---|---|---|
| OpenAI | `api-key` | — | `chat.options.model` |
| Anthropic | `api-key` | `version` (API version header) | `chat.options.model` |
| Ollama | *(none — local)* | `base-url` (points to local server) | `chat.options.model` |
| Azure OpenAI | `api-key` (or Azure AD) | `endpoint` | `chat.options.deployment-name` |
| Amazon Bedrock | AWS `access-key`/`secret-key` (or IAM) | `region` | `converse.chat.options.model` |
| Mistral AI | `api-key` | — | `chat.options.model` |
| Vertex AI Gemini | GCP service account credentials | `project-id`, `location` | `chat.options.model` |
| NVIDIA / Groq / etc. (via OpenAI starter) | `api-key` | `base-url` (provider endpoint) | `chat.options.model` |

## 11. How to Decide Which One to Use

- **Prototyping locally / no cost / privacy-sensitive data** → Ollama
- **Want Claude directly, simplest setup** → Anthropic starter
- **Already deep in AWS, want multiple model vendors under one IAM/billing setup** → Bedrock
- **Already deep in Azure/enterprise Microsoft shop** → Azure OpenAI
- **Already deep in Google Cloud** → Vertex AI Gemini
- **Want a specific niche/cheap/fast provider (NVIDIA NIM, Groq, Together, etc.)** → OpenAI starter + custom `base-url`, if that provider is OpenAI-compatible (most are)

## 12. Official Documentation Links

📖 **All chat model integrations index:** https://docs.spring.io/spring-ai/reference/api/chat/comparison.html

📖 **OpenAI Chat:** https://docs.spring.io/spring-ai/reference/api/chat/openai-chat.html

📖 **Anthropic Chat:** https://docs.spring.io/spring-ai/reference/api/chat/anthropic-chat.html

📖 **Ollama Chat:** https://docs.spring.io/spring-ai/reference/api/chat/ollama-chat.html

📖 **Azure OpenAI Chat:** https://docs.spring.io/spring-ai/reference/api/chat/azure-openai-chat.html

📖 **Amazon Bedrock Converse:** https://docs.spring.io/spring-ai/reference/api/chat/bedrock-converse.html

📖 **Mistral AI Chat:** https://docs.spring.io/spring-ai/reference/api/chat/mistralai-chat.html

📖 **Vertex AI Gemini Chat:** https://docs.spring.io/spring-ai/reference/api/chat/vertexai-gemini-chat.html

---
*Personal learning notes — provider comparison, covering property differences and when to reach for each one.*
