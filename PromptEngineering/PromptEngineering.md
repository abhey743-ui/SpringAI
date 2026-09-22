
# 3. Prompt Engineering — The Actual Craft

Everything in file 2 was about the *dial settings*. This file is about the *words* — which, honestly, matters more than any parameter you tune.

A useful framing that shows up a lot in current writing on this topic: prompt engineering in 2026 has really split into two different skills.

- **Casual prompting** — getting a good answer once, in a chat window. Modern frontier models are forgiving of sloppy phrasing here.
- **Production prompting** — getting a *reliable* answer, thousands of times a day, across many users, languages, and edge cases, with nobody watching each individual response. This is the harder skill, and it's the one that actually matters when you're writing a Spring AI service.

This file focuses on production prompting, since that's the context your `ChatClient` code lives in.

## The core techniques, in order of how often you'll actually reach for them

### 1. Zero-shot prompting
Just a clear instruction, no examples. The model relies entirely on what it already learned during training.

```java
chatClient.prompt()
    .system("You are a precise sentiment classifier. Reply with exactly one word: Positive, Neutral, or Negative.")
    .user(reviewText)
    .call()
    .content();
```

This works well for tasks that are common and unambiguous — summarization, basic classification, simple extraction. It fails when the desired output *format* or reasoning style isn't obvious from the instruction alone — the model will still answer confidently, it'll just miss what you actually wanted.

### 2. Few-shot prompting
You give the model 2–5 example input/output pairs before the real task. This doesn't teach the model anything new — it narrows the *interpretation* of an otherwise-ambiguous instruction by showing exactly what "correct" looks like.

```java
String template = """
    Classify sentiment as Positive, Neutral, or Negative.

    Input: "The staff was incredibly helpful and friendly."
    Output: Positive

    Input: "The food was okay, nothing special."
    Output: Neutral

    Input: "My order was wrong and the waiter was rude."
    Output: Negative

    Input: "{review}"
    Output:
    """;
```

Use few-shot when:
- The model has never reliably seen your exact domain pattern before.
- Output *format* is important and hard to describe in words (a weird CSV layout, a specific JSON shape, your company's tone of voice).
- You've actually observed the zero-shot version getting it wrong in a consistent way.

Keep it to 2–5 examples — more than that has diminishing returns and just burns tokens (which is real money, see file 5). And keep the examples up to date as new edge cases show up in production; a few-shot block that was written once and never revisited quietly goes stale.

### 3. Role / persona prompting
Telling the model *who it is* before it answers shapes tone, vocabulary, and default assumptions more than you'd expect.

```java
.system("You are a senior tax accountant reviewing a client's return for errors.")
```

The trap here is being too theatrical about it. "You are a wise old wizard of finance" wastes tokens and adds nothing — keep the role functional and specific to the actual task.

### 4. Chain-of-thought (CoT)
Asking the model to reason step-by-step before giving a final answer. Historically this was done by literally appending "think step by step" to the prompt.

Important 2026 nuance: modern **reasoning models** (the "thinking" tier of models) already do internal step-by-step reasoning automatically. Forcing an explicit "think step by step" instruction on top of that is often redundant, and can occasionally *hurt* accuracy rather than help it. Explicit CoT still earns its keep on non-reasoning/cheaper models, or any time you want the reasoning trace to actually be visible and inspectable (for debugging or for showing your work to a user).

A practical version — asking for the reasoning and the final answer as separate structured fields (ties directly into file 4's structured output):

```java
record ReasonedAnswer(String reasoning, String finalAnswer) {}

ReasonedAnswer result = chatClient.prompt()
    .user("A train travels 60 km in 45 minutes. What's its speed in km/h? Show your reasoning.")
    .call()
    .entity(ReasonedAnswer.class);
```

### 5. Self-consistency
Run the same prompt multiple times (often with a nonzero temperature) and take the majority answer, or use a second call to have the model check its own first answer. This trades extra API calls (and cost) for higher reliability on genuinely hard reasoning tasks. Reserve this for the small slice of high-stakes queries where the extra cost is justified — not your whole traffic.

### 6. Task decomposition / prompt chaining
Instead of one giant prompt trying to do five things at once, break it into a sequence of smaller `ChatClient` calls, where each one's output feeds the next one's input. This is the same instinct as breaking a large method into smaller ones — each individual prompt becomes easier to test, easier to reason about, and easier to swap for a cheaper model if that particular step doesn't need a frontier model. The real risk: errors from an early step compound into later ones, so validate intermediate outputs rather than blindly trusting them.

### 7. ReAct-style / tool-augmented prompting
The model is told it has tools available and reasons about *when* to call them versus answering directly. In Spring AI this isn't something you write by hand in the prompt text — it's built into the `.tools(...)` mechanism and the `ToolCallingAdvisor`, covered fully in file 4. Conceptually, though, it's still a prompting pattern: the system prompt describes what the tools are for and when to prefer them.

### 8. Meta-prompting
Using the model itself to help write or refine a prompt — "here's my current prompt and three examples where it failed, suggest a better version." Genuinely useful during development; not something you run live in production traffic.

## What a real production prompt actually looks like

A single "prompt" in a production system usually isn't one string — it's layered, and each layer maps directly onto something you already have in the Spring AI fluent API:

1. **System prompt** — persona, tone, hard constraints. → `.system(...)`
2. **Few-shot examples** (if needed) — folded into the system or user text via `PromptTemplate` variables.
3. **Tool definitions** — → `.tools(...)`
4. **Retrieved context** (RAG) — → handled by an advisor like `QuestionAnswerAdvisor` (file 4), not hand-written into the prompt each time.
5. **Conversation history** — → handled by a memory advisor (file 4).
6. **The actual user input** — → `.user(...)`

Production system prompts at serious AI companies routinely run into the thousands of tokens once you account for edge-case handling and explicit behavior rules — that's normal, not a sign you're doing something wrong. What matters is that it's a deliberate, reviewed, versioned piece of text — not something assembled fresh and unreviewed on every request.

## A rule of thumb for choosing a technique

| If your problem is... | Reach for... |
|---|---|
| The task is common and the model already "gets it" | Zero-shot |
| The output format/style is specific and hard to describe in words | Few-shot |
| The model needs a consistent voice/perspective | Role prompting |
| The task involves multi-step reasoning and you're on a non-reasoning model | Explicit chain-of-thought |
| The task is high-stakes and worth the extra cost of double-checking | Self-consistency |
| The task is actually five smaller tasks glued together | Decomposition / chaining |
| The answer depends on live data or an action needs to happen | Tools |
| The answer depends on your own private documents | RAG (an advisor, see file 4) |

Whatever technique you pick, the same production discipline applies: don't trust it because it "felt right" in one manual test. File 5 covers how teams actually verify a prompt is good *before* shipping it.
