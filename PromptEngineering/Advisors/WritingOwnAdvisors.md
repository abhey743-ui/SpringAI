# 8. Writing Your Own Advisor — A Professional, End-to-End Build

Files 6 and 7 covered the theory and the built-ins. This file builds a real custom advisor from nothing, the way you'd actually do it on a team: pick a genuine need, implement it properly for both blocking and streaming, order it correctly against the other advisors in your chain, test it without hitting a real model, and wire it into observability.

## Step 0 — Pick a real reason to write one

You write a custom advisor when a built-in doesn't cover something you need. Realistic examples:
- Redacting PII (emails, phone numbers, card numbers) from a response before it ever reaches the client.
- Enforcing a per-tenant or per-user rate limit before spending money on a model call.
- Injecting request-scoped context (a tenant ID, a correlation ID) into every prompt automatically, so callers don't have to remember to do it themselves.
- A second, smarter layer of content safety beyond what `SafeGuardAdvisor`'s keyword list can catch (file 7 flagged this gap explicitly).

We'll build the PII-redaction example end to end, since it exercises everything: reading the response, modifying it, and making a real decision about ordering.

## Step 1 — Dependencies

Nothing extra. Custom advisors are built entirely on `spring-ai-client-chat`, which you already have from any model starter. No new artifact needed.

## Step 2 — Implement both `CallAdvisor` and `StreamAdvisor`

Always implement both, even if your app is blocking-only today — someone will add a streaming endpoint eventually, and an advisor that silently does nothing in streaming mode is a bug waiting to be found in production.

```java
package com.example.aiapp.advisor;

import org.springframework.ai.chat.client.ChatClientRequest;
import org.springframework.ai.chat.client.ChatClientResponse;
import org.springframework.ai.chat.client.advisor.api.CallAdvisor;
import org.springframework.ai.chat.client.advisor.api.CallAdvisorChain;
import org.springframework.ai.chat.client.advisor.api.StreamAdvisor;
import org.springframework.ai.chat.client.advisor.api.StreamAdvisorChain;
import org.springframework.ai.chat.messages.AssistantMessage;
import org.springframework.ai.chat.model.ChatResponse;
import org.springframework.ai.chat.model.Generation;
import reactor.core.publisher.Flux;

import java.util.List;
import java.util.regex.Pattern;

public class PiiRedactionAdvisor implements CallAdvisor, StreamAdvisor {

    private static final Pattern EMAIL = Pattern.compile("[\\w.+-]+@[\\w-]+\\.[\\w.-]+");
    private static final Pattern PHONE = Pattern.compile("\\b\\d{3}[-.\\s]?\\d{3}[-.\\s]?\\d{4}\\b");

    private final int order;

    public PiiRedactionAdvisor(int order) {
        this.order = order;
    }

    @Override
    public String getName() {
        return "PiiRedactionAdvisor";
    }

    @Override
    public int getOrder() {
        return this.order;
    }

    @Override
    public ChatClientResponse adviseCall(ChatClientRequest request, CallAdvisorChain chain) {
        ChatClientResponse response = chain.nextCall(request);
        return redact(response);
    }

    @Override
    public Flux<ChatClientResponse> adviseStream(ChatClientRequest request, StreamAdvisorChain chain) {
        // Redacting token-by-token would risk splitting a match across chunks,
        // so we buffer the full response, then redact once, then re-emit it.
        return chain.nextStream(request)
                .collectList()
                .flatMapMany(responses -> {
                    if (responses.isEmpty()) {
                        return Flux.empty();
                    }
                    ChatClientResponse last = responses.get(responses.size() - 1);
                    return Flux.just(redact(last));
                });
    }

    private ChatClientResponse redact(ChatClientResponse response) {
        ChatResponse chatResponse = response.chatResponse();
        if (chatResponse == null) {
            return response;
        }

        List<Generation> cleaned = chatResponse.getResults().stream()
                .map(this::redactGeneration)
                .toList();

        ChatResponse newChatResponse = ChatResponse.builder()
                .from(chatResponse)
                .generations(cleaned)
                .build();

        return ChatClientResponse.builder()
                .from(response)
                .chatResponse(newChatResponse)
                .build();
    }

    private Generation redactGeneration(Generation generation) {
        if (!(generation.getOutput() instanceof AssistantMessage message)) {
            return generation;
        }
        String text = message.getText();
        String scrubbed = EMAIL.matcher(text).replaceAll("[redacted-email]");
        scrubbed = PHONE.matcher(scrubbed).replaceAll("[redacted-phone]");
        return new Generation(new AssistantMessage(scrubbed), generation.getMetadata());
    }
}
```

A few deliberate choices worth calling out, because they're the difference between a demo and something you'd actually ship:

- **The order is a constructor parameter, not hardcoded.** This lets you place this exact advisor differently in different `ChatClient` beans if you ever need to, without touching the class itself.
- **Streaming buffers instead of scanning chunk-by-chunk.** A naive per-token regex would happily let `user@examp` through as one chunk and `le.com` through as the next, missing the match entirely. Buffering the full stream before scrubbing is the only reliable approach for pattern-based redaction — it costs you true token-by-token streaming to the client for this one advisor's position in the chain, which is a real, known tradeoff to document, not hide.
- **It never blocks the request outright** — it only ever transforms the response. Compare this to how a guardrail advisor would be written (next section) which *can* stop the chain entirely.

## Step 3 — A blocking-style advisor: stopping the request outright

For contrast, here's the shape a request-blocking advisor takes — the key difference is it doesn't unconditionally call `chain.nextCall(...)`:

```java
@Override
public ChatClientResponse adviseCall(ChatClientRequest request, CallAdvisorChain chain) {
    String text = request.prompt().getContents();
    if (containsDisallowedPattern(text)) {
        AssistantMessage refusal = new AssistantMessage(
            "I can't help with that request.");
        ChatResponse blocked = ChatResponse.builder()
            .generations(List.of(new Generation(refusal)))
            .build();
        return ChatClientResponse.builder()
            .chatResponse(blocked)
            .context(request.context())
            .build();
    }
    return chain.nextCall(request); // only reached if it passed the check
}
```

Notice the model is never called at all in the blocked branch — no tokens spent, no latency from the provider, and the caller still gets back a well-formed `ChatClientResponse` rather than an exception or a null value (file 7 flagged that `SafeGuardAdvisor` historically had bugs here — returning a real, well-formed response instead of an empty one is exactly the lesson to take from that).

## Step 4 — Get the ordering right, deliberately, not by accident

This is the step people skip, and it's where subtle bugs live. Think through where your advisor needs to sit relative to the others in the same chain:

- **Safety/blocking checks that should see the raw user input** → very low order (run first, before memory or RAG have modified the request). This is exactly the `SafeGuardAdvisor` ordering issue from file 7 — if it ran after a memory advisor injected old history, it could trip on stale content.
- **Response redaction, like the PII example above** → should typically be the *outermost* layer on the way out, meaning a low order number, so it's the very last thing to touch the response before it reaches your controller — you want it downstream of everything else that might add sensitive content (memory summaries, tool outputs), not upstream of it.
- **An advisor that needs to see every intermediate step of a tool-calling loop** (say, a cost-tracking advisor that sums tokens across every tool round-trip) needs to sit *inside* `ToolCallingAdvisor`'s position (recall from file 4 that its default order is `Ordered.HIGHEST_PRECEDENCE + 300`) — otherwise it only ever sees the final, post-loop result, not each individual round trip.

Write down, in a comment on the `@Configuration` class where you register these, *why* each one is ordered where it is. Six months from now, nobody — including you — will remember the reasoning otherwise, and someone will "simplify" the ordering and quietly reintroduce the bug you just avoided.

## Step 5 — Registering it properly

```java
@Configuration
public class AiConfig {

    @Bean
    ChatClient chatClient(ChatClient.Builder builder, ChatMemory chatMemory) {
        return builder
            .defaultAdvisors(
                new SafeGuardAdvisor(...),              // order: very low — sees raw input first
                MessageChatMemoryAdvisor.builder(chatMemory).build(),
                new PiiRedactionAdvisor(Ordered.LOWEST_PRECEDENCE) // last to touch the response
            )
            .build();
    }
}
```

## Step 6 — Testing it without spending real API money

Because `CallAdvisorChain` is just an interface, you can fake it in a plain unit test — no Spring context, no real model call, no cost:

```java
@Test
void redactsEmailFromResponse() {
    PiiRedactionAdvisor advisor = new PiiRedactionAdvisor(0);

    ChatClientRequest request = ChatClientRequest.builder()
        .prompt(new Prompt("What's my account email?"))
        .build();

    CallAdvisorChain fakeChain = mock(CallAdvisorChain.class);
    ChatResponse rawResponse = ChatResponse.builder()
        .generations(List.of(new Generation(
            new AssistantMessage("Your email on file is jane.doe@example.com"))))
        .build();
    when(fakeChain.nextCall(any())).thenReturn(
        ChatClientResponse.builder().chatResponse(rawResponse).build());

    ChatClientResponse result = advisor.adviseCall(request, fakeChain);

    assertThat(result.chatResponse().getResult().getOutput().getText())
        .contains("[redacted-email]")
        .doesNotContain("jane.doe@example.com");
}
```

This is the professional standard: **an advisor's own logic should be testable in complete isolation from the LLM.** If you find yourself needing an actual model call to test an advisor, that's usually a sign the advisor is doing too much, or that some of its logic belongs in a plain helper class the advisor merely calls.

## Step 7 — Observability comes free, but name things well

Because advisors participate in Spring AI's observability stack automatically (file 6), the one thing you have to get right yourself is `getName()` — return something specific and stable (`"PiiRedactionAdvisor"`, not `"Advisor1"` or the raw class name of an anonymous inner class), since that's the label that will show up in every trace and metric for this advisor's execution. A vague name here is the equivalent of an unlabeled span in normal APM tracing — technically present, practically useless when you're trying to find it during an incident.

## A professional checklist before you ship a custom advisor

- [ ] Implements both `CallAdvisor` and `StreamAdvisor`, even if only one is used today
- [ ] `getName()` returns something specific and greppable in logs/traces
- [ ] `getOrder()` is set deliberately, with a comment explaining *why*, relative to the other advisors in the same chain
- [ ] Never returns `null` or an empty result on the "blocked" path — always a well-formed `ChatClientResponse`
- [ ] Core logic is extracted into a plain, framework-free method/class that can be unit tested without a fake chain or a real model
- [ ] Streaming behavior has been deliberately thought through (buffer vs. true per-chunk processing), not just copy-pasted from the blocking version
- [ ] Doesn't silently swallow exceptions — if something inside the advisor fails, that failure should be visible (thrown, logged, or turned into an explicit error response), not hidden
- [ ] If it touches sensitive data (PII, secrets), that data is never accidentally written to logs by the advisor itself, or by `SimpleLoggerAdvisor`/observability running alongside it (recall from file 2 that `log-prompt`/`log-completion` default to `false` for exactly this reason)

That's the complete loop — from "what problem am I solving" through implementation, ordering, testing, and the operational details that separate a tutorial snippet from something a team would actually trust in production.
