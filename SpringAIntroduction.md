# Spring AI — Explained Simply

## 1. What is Spring AI?

Spring AI is a framework built by the Spring team (the same people who made Spring Boot, Spring Data, etc.) that lets Java developers add AI features into their applications — things like chatbots, document search, question-answering over your own data, and AI agents — **without having to learn a totally new way of coding.**

Think of it like this: you already know how Spring Boot lets you plug in a database, a message queue, or a REST client using simple annotations and starters. Spring AI does the exact same thing, but for AI models (like GPT, Claude, Gemini, Llama, etc.). Instead of writing raw HTTP calls to OpenAI's API or Anthropic's API yourself, you use familiar Spring patterns — dependency injection, auto-configuration, builders — and Spring AI handles the messy plumbing underneath.

In short:

> **Spring AI = Spring-style building blocks for talking to AI models.**

## 2. Why does it exist? What problem does it solve?

If you tried to add AI to a Java app *without* Spring AI, you'd run into these headaches:

- Every AI provider (OpenAI, Anthropic, Google, Amazon Bedrock, Ollama, Mistral, DeepSeek...) has a **different API format**. Switching providers means rewriting a lot of code.
- You'd have to manually manage things like retries, streaming responses, prompt templates, and converting AI responses into Java objects.
- Adding memory (so the AI remembers earlier parts of a conversation) or connecting the AI to your own data (RAG) would mean building a lot of custom infrastructure yourself.

Spring AI's core idea is to give you **one consistent API** that works across almost any AI provider, so you can:

- Swap models/providers with minimal code changes (portability)
- Use Spring Boot's auto-configuration and starters to wire things up fast
- Build AI features the same clean, POJO-based way you already build regular Spring apps

## 3. Key Building Blocks (the main things you'll actually use)

### a) ChatClient
This is the main API you'll interact with. It's a **fluent builder**, very similar to Spring's `WebClient` or `RestClient` if you've used those. You basically say: "here's my prompt, here's the model, give me back a response."

```java
String response = chatClient.prompt()
    .user("Explain Spring AI in one sentence")
    .call()
    .content();
```

That's it — no manual HTTP requests, no manual JSON parsing.

### b) Advisors API
Advisors let you plug in **cross-cutting behavior** around your AI calls — similar to interceptors/filters in web development. Common uses:
- **Conversation memory** (so the AI remembers past messages)
- **RAG** (Retrieval-Augmented Generation — feeding the AI your own documents/data before it answers)
- **Tool calling** (letting the AI call your Java methods)

You attach advisors to your `ChatClient` and they automatically wrap every request/response.

### c) Models & Providers
Spring AI supports many AI providers through a consistent interface: **Anthropic (Claude), OpenAI, Google, Amazon Bedrock, Ollama, Mistral AI, DeepSeek**, and more. You pick your provider via a Spring Boot **starter** dependency and some config properties — the actual Java code calling the model barely changes even if you switch providers later.

### d) Spring Boot Auto-Configuration & Starters
Just like `spring-boot-starter-data-jpa` gives you a database connection with almost no setup, Spring AI has starters (e.g. for OpenAI, Anthropic, etc.) that auto-configure everything for you. You typically pick your model/vector-store on **start.spring.io** just like any other Spring Boot project.

### e) Vector Stores & RAG (Retrieval-Augmented Generation)
"RAG" means: instead of the AI answering purely from what it was trained on, you feed it *your own* documents/data at question time so it can give accurate, up-to-date answers about *your* content (e.g., "Chat with your documentation").

To do this, Spring AI includes:
- An **ETL (Extract-Transform-Load) framework** for loading your documents (PDFs, text, etc.) into a **vector database**
- Support for many popular vector stores
- Built-in retrieval logic so the AI can search that data before answering

### f) Tool Calling (Function Calling)
You can let the AI call your own Java methods when it needs extra information or needs to *do* something (e.g., "look up this order status," "send this email"). Spring AI handles the back-and-forth of the AI deciding to call a tool, executing your code, and feeding the result back to the model.

### g) MCP (Model Context Protocol)
Spring AI integrates with **MCP**, an open protocol for connecting AI applications to external tools/data sources. This lets your Spring app either **consume** MCP servers (use external AI tools) or **expose** its own services as MCP servers (so other AI systems/agents can use them).

### h) Chat Memory
Keeps track of conversation history so a chatbot can have a coherent, multi-turn conversation instead of forgetting everything after each message.

### i) Observability & Evaluation
- **Observability**: built-in insights/metrics into what your AI calls are doing (useful for debugging and monitoring in production).
- **Evaluation**: utilities to help you check whether the AI's responses are accurate and to help guard against "hallucinations" (the AI making things up).

## 4. What can you actually *build* with it?

- Chatbots / customer support assistants
- "Chat with your documentation" or "Q&A over your own data" systems (RAG)
- AI agents that can call tools/APIs to take actions, not just answer questions
- Content generation features baked into your app (summaries, translations, etc.)
- Multi-agent systems, using it alongside MCP

## 5. How it fits with what you already know (Spring Boot / Spring Cloud Streams)

Since you're already working with **Spring Boot, Spring Cloud Streams, and Debezium** (per your RabbitMQ project), Spring AI will feel very familiar structurally:

| Concept you know | Spring AI equivalent |
|---|---|
| `spring-boot-starter-amqp` (RabbitMQ starter) | `spring-ai-starter-model-openai` (or anthropic, etc.) |
| `application.yml` config for RabbitMQ | `application.yml` config for your AI provider's API key/model |
| `@Bean` for a `RabbitTemplate` | Auto-configured `ChatClient` bean, ready to inject |
| Message listeners/consumers | Advisors wrapping the chat calls |

It genuinely reuses the same mental model — starters, auto-config, and injected clients — just pointed at an AI provider instead of a broker.

## 6. Requirements / Notes

- Requires **Java 17+** (the upcoming Spring AI 2.0 line is expected to target Java 21+)
- It lives inside a normal Spring Boot application — you don't need any special runtime
- The project is actively evolving — as of writing, the stable version is in the **1.1.x / 1.0.x line**, with a **2.0.x** line also progressing. Always check the docs for the latest version before starting a new project.

## 7. Official Documentation

📖 **Main reference docs:** https://docs.spring.io/spring-ai/reference/index.html

📖 **Getting Started guide:** https://docs.spring.io/spring-ai/reference/getting-started.html

📖 **Project homepage:** https://spring.io/projects/spring-ai/

📖 **GitHub repo:** https://github.com/spring-projects/spring-ai

---
*Notes file generated for personal learning reference — feel free to expand with your own code snippets and experiments as you go through the docs.*
