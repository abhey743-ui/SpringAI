# Spring AI — Full Learning Notes (Overview)

You showed a small `ChatClient` demo — a controller that builds a `ChatClient` from `ChatClient.Builder`, then tries a bunch of ways to call it: plain `.prompt(value)`, `.prompt().user().system()`, a manually built `Prompt` object with `OpenAiChatOptions` (temperature, maxTokens...), and finally `.entity(Prog.class)` / `.entity(new ParameterizedTypeReference<List<Prog>>(){})` to get typed Java objects back instead of raw strings.

That one file is actually touching **five separate topics** that are usually taught separately:

1. How `ChatClient` itself works (the fluent chain)
2. What parameters/options you can send to the model, and where they live
3. How to actually *write* a good prompt (prompt engineering)
4. How structured output, tools, and advisors work
5. How real companies run this in production without burning money or getting embarrassed by a bad output

So instead of one giant file, here are five focused files. Read them in this order if you're doing this properly for the first time:

| # | File | What it covers |
|---|------|-----------------|
| 1 | `01-chatclient-fundamentals.md` | The ChatClient fluent API itself — `prompt()`, `call()`, `stream()`, `content()`, message roles, PromptTemplate |
| 2 | `02-parameters-and-configuration.md` | Every knob you can turn (temperature, topP, maxTokens, etc.), and every place you can configure them — properties files, env vars, runtime overrides |
| 3 | `03-prompt-engineering-techniques.md` | The actual craft of prompting — zero-shot, few-shot, chain-of-thought, role prompting, and how production prompts are structured |
| 4 | `04-structured-output-tools-advisors.md` | How `.entity()` really works under the hood, tool/function calling, and Advisors (RAG, memory, logging) |
| 5 | `05-industry-practices-and-optimization.md` | How teams actually run this in production — evals, guardrails, observability, and cutting your token bill |

A quick mental model before you dive in, because this is the thing that makes everything else click:

```
Your Controller
      │
      ▼
ChatClient  (the thing you call — fluent, easy to use)
      │
      ├── Advisors (memory, RAG, logging — optional middleware)
      ├── PromptTemplate (fills in {placeholders})
      │
      ▼
ChatModel  (the actual interface — OpenAiChatModel, AnthropicChatModel, etc.)
      │
      ▼
The actual provider's HTTP API (OpenAI, Anthropic, Azure, Gemini, Ollama...)
```

`ChatClient` is what you touch every day. `ChatModel` is the thing underneath it that's specific to whichever provider you picked. This separation is the whole reason Spring AI exists — swapping OpenAI for Anthropic or a local Ollama model should mostly be a config change, not a rewrite.

One thing worth saying up front, because it trips almost everyone up the first week: in the fluent API, `.call()` (or `.stream()`) does **not** send the request. It just decides *how* the request will eventually be sent (blocking vs. streaming). The actual HTTP call only fires when you chain `.content()`, `.chatResponse()`, or `.entity()` after it. That's why you'll sometimes see people accidentally build three "requests" that never actually run.

Go read file 01 next.
