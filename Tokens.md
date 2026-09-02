# Tokens — The Most Important Concept in AI You Need to Understand

You're right to flag this — **tokens are the single unit that everything in AI is built around.** Cost, speed, memory limits, context length, even *why* the model sometimes "forgets" earlier parts of a conversation — all of it traces back to tokens. Let's go deep, in plain language.

## 1. What Exactly Is a Token?

A token is **a small chunk of text** — not quite a word, not quite a letter. It's the basic unit an AI model actually reads and writes in.

Here's the key thing that surprises most people: **the model doesn't see "words" at all.** Before your text ever reaches the model, it gets broken down (tokenized) into these chunks, and the model only ever works with numbers representing those chunks.

Rough intuition (actual splits vary by model/tokenizer):

```
"I love programming" → ["I", " love", " program", "ming"]  → 4 tokens
"Hello"               → ["Hello"]                            → 1 token
"unbelievable"         → ["un", "believ", "able"]             → 3 tokens
"Spring AI"            → ["Spring", " AI"]                    → 2 tokens
```

A useful rule of thumb for English text: **1 token ≈ 4 characters ≈ ¾ of a word.** So 100 tokens is roughly 75 words.

Tokens aren't always whole words — common words often get their own token, but rarer or longer words get split into pieces (called "subwords"). This is deliberate: it lets the model handle *any* word, even ones it's never seen before, by composing it from familiar sub-pieces, and it keeps the model's vocabulary size manageable (usually 50,000–200,000 possible tokens, instead of needing a token for literally every possible word).

## 2. Why Tokens Exist At All (The "Why" Behind the Design)

Neural networks can't process raw text — they process **numbers**. So every AI provider maintains a **tokenizer**: a fixed lookup table that converts text ↔ numbers in both directions.

```
Your text  →  Tokenizer  →  [1212, 1842, 8622, ...]  →  Model processes numbers  →  Model outputs numbers  →  Tokenizer  →  Text back to you
```

Different providers use **different tokenizers** — this is a big deal, because it means the *same sentence* can cost a different number of tokens depending on which model you send it to:

- OpenAI models use a tokenizer called **tiktoken** (specifically, encodings like `cl100k_base` / `o200k_base` depending on model generation).
- Anthropic's Claude models use their own tokenizer.
- Ollama/open-source models each ship with their own tokenizer, usually tied to the specific model family (Llama, Mistral, Qwen, etc. all tokenize slightly differently).

This is exactly why you can't reliably estimate token counts by "word count" alone across providers — the real number depends on which model's tokenizer is doing the counting.

## 3. The Three Token Numbers You'll See Constantly

Every AI response comes back with token accounting, broken into three numbers:

| Term | What it means |
|---|---|
| **Prompt tokens** (a.k.a. input tokens) | Everything you sent — your message, system prompt, conversation history, any documents/RAG content injected |
| **Completion tokens** (a.k.a. output tokens) | Everything the model generated back to you |
| **Total tokens** | Prompt + completion combined |

This distinction matters a lot because **providers usually charge different prices for input vs. output tokens** (output is typically more expensive per token than input, since generating is more computationally expensive than reading).

## 4. How AI Providers Manage Tokens (Their Side of the Equation)

Providers care about tokens for three separate reasons, and it helps to keep them separate in your head:

### a) Context Window (a hard ceiling)
Every model has a maximum number of tokens it can "see" at once — this is called the **context window**. It includes *everything*: your system prompt + conversation history + your current message + the space reserved for the model's response. If you exceed it, the request either fails outright or older parts of the conversation get silently truncated/dropped (behavior depends on the client/provider).

This is *why* long conversations eventually "forget" early messages — they literally fall outside the context window once you accumulate too many tokens.

### b) Pricing (what you get billed for)
Nearly every commercial provider (OpenAI, Anthropic, etc.) bills **per token**, typically quoted as a price per 1 million tokens, separately for input and output. This is why:
- Long system prompts cost you money on *every single request*, not just once.
- A chatty back-and-forth conversation gets progressively more expensive per turn — because each new message re-sends the *entire* conversation history as prompt tokens (the model has no memory of its own between calls; your app has to resend everything each time).

### c) Rate Limits (how fast you can send)
Providers also cap **tokens-per-minute** (TPM) and **requests-per-minute** (RPM), separately from your total quota. This protects their infrastructure from being overwhelmed, and it's tied to your account tier/plan. Hit the limit, and you get throttled or rejected until it resets.

## 5. How This Flows Through a Spring Boot App (Your Side of the Equation)

This is where it connects to everything you've learned about `ChatClient`/`ChatModel`. Here's the actual request lifecycle, token-by-token:

```
1. Your code builds a Prompt (system message + user message + history)
        │
2. ChatClient/ChatModel sends this to the provider's tokenizer internally
        │  (this happens on the PROVIDER's servers — not in your JVM)
        │
3. Provider tokenizes your entire prompt → counts "prompt tokens"
        │
4. Provider checks: does this fit inside the model's context window?
        │  NO → error response (context length exceeded)
        │  YES → continue
        │
5. Model generates a response, token by token, until:
        │  - it naturally finishes (hits a stop token)
        │  - it hits your configured max-tokens limit
        │  - it hits a custom `stop` sequence you defined
        │
6. Provider counts "completion tokens" generated
        │
7. Response comes back to your Spring app WITH token usage metadata attached
```

### The property you already used: `max-tokens`

Remember your NVIDIA config?

```yaml
spring:
  ai:
    openai:
      chat:
        max-tokens: 16384
```

Now you know exactly what this does: **it caps how many completion tokens the model is allowed to generate in a single response.** It does *not* limit your prompt size — only the output side. If the model would naturally want to keep generating past that limit, it gets cut off there. This is a safety/cost control — without it, a model could theoretically ramble on (and cost you) indefinitely.

### Reading token usage back out in Spring AI

Every `ChatResponse` carries a `Usage` object with exactly the three numbers from Section 3:

```java
ChatResponse response = chatClient.prompt()
        .user("Explain gravity")
        .call()
        .chatResponse();

Usage usage = response.getMetadata().getUsage();

Integer promptTokens     = usage.getPromptTokens();
Integer completionTokens = usage.getCompletionTokens();
Integer totalTokens      = usage.getTotalTokens();
```

Spring AI also exposes **rate limit metadata** the same way — how many tokens/requests you have left before hitting the provider's limit:

```java
RateLimit rateLimit = response.getMetadata().getRateLimit();

Long tokensRemaining = rateLimit.getTokensRemaining();
Duration tokensReset = rateLimit.getTokensReset();
```

This is genuinely useful in production: you can log this, expose it on a dashboard, or even build your own internal rate-limiting/cost-alerting logic on top of it — Spring AI is just surfacing what the provider already sends back in the raw API response.

## 6. A Subtlety Worth Knowing: `ChatClient` Can Silently Inflate Your Token Count

This is a real, documented gotcha: because `ChatClient` wraps your `ChatModel` with **Advisors** (memory, RAG, default system messages, tool definitions, etc.), the *actual* prompt sent to the model can be much bigger than the text you literally typed. Developers have reported prompt-token counts 10–20x higher than expected when using `ChatClient` versus calling the provider's SDK directly — usually traced back to things like a large embedded image, accumulated chat memory, or tool schemas being silently included in every request.

**Practical takeaway:** if your token usage looks suspiciously high, check what your Advisors are silently attaching to every prompt — memory, RAG-injected documents, and tool definitions all count as prompt tokens even though you didn't type them yourself.

## 7. Why This Matters "For Today's World" (Your Question)

You're right that this is central to how the entire industry works right now:

- **Pricing models** for every major AI product (ChatGPT Plus, Claude Pro, API pricing, enterprise contracts) are fundamentally token-based, even when hidden behind a flat subscription fee — the provider is still tracking your token consumption behind the scenes.
- **Model comparisons** you see online (e.g., "128K context window" vs "1M context window") are literally just comparing how many tokens each model can hold in memory at once — a bigger context window means it can handle longer documents, longer conversations, more RAG content, before truncating.
- **Prompt engineering** as a discipline exists partly *because* of tokens — a concise, well-structured prompt costs fewer tokens (cheaper, faster) than a rambling one that says the same thing.
- **RAG (Retrieval-Augmented Generation)**, which you learned about earlier, exists specifically to work around context-window limits — instead of stuffing your entire knowledge base into every prompt (impossible — it wouldn't fit, and would be absurdly expensive), you retrieve only the most relevant chunks and inject just those as tokens.
- **Agent/tool-calling systems** (also from your earlier notes) burn extra tokens on every single request because the tool definitions/schemas themselves have to be sent as part of the prompt every time — more tools registered = more baseline tokens per call, even before your actual question.

## 8. Quick Mental Model to Keep

```
Tokens are the "currency" of AI:
- They're what you're billed in (input + output priced separately)
- They're what defines how much the model can "remember" at once (context window)
- They're what rate limits are measured in (tokens-per-minute)
- They're what max-tokens controls (caps the OUTPUT side only)
- They're counted by the PROVIDER, using THEIR tokenizer — not something your JVM computes
- Spring AI just reads back the usage numbers the provider already calculated, via ChatResponse.getMetadata().getUsage()
```

## 9. Official Documentation

📖 **Spring AI — AI Metadata (Usage & Rate Limits):** https://docs.spring.io/spring-ai/reference/api/aimetadata.html

📖 **OpenAI — Tokenizer concept & tool:** https://platform.openai.com/tokenizer

📖 **Anthropic — Token counting:** https://docs.anthropic.com/en/docs/build-with-claude/token-counting

📖 **Spring AI — Chat Models overview:** https://docs.spring.io/spring-ai/reference/api/chatmodel.html

---
*Personal learning notes — deep dive on tokens: what they are, why they exist, how providers manage them, and how they flow through a Spring AI application.*
