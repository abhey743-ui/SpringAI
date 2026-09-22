# 10. Vectors & Embeddings — What They Actually Are, and Why RAG Needs Them

Files 4 and 7 kept mentioning "the vector store" whenever `QuestionAnswerAdvisor` or `VectorStoreChatMemoryAdvisor` came up, without really explaining what's happening underneath. This file fills that gap — starting from "what even is a vector" and building all the way up to a real, working setup.

## The problem this is solving, first

An LLM only knows two things: what it learned during training, and whatever text you put directly in the prompt. It has no access to your company's internal documents, your database, or anything written after its training cutoff — unless you physically paste it into the prompt yourself.

The naive fix — "just paste all our documents into the prompt" — falls apart fast: prompts have a size limit (the context window), sending huge amounts of text on every single call is expensive (file 5's cost math applies directly), and most of what you'd paste in is irrelevant to any given question anyway.

What you actually want is: **given a question, find the small handful of documents that are actually relevant, and only send those.** That's the whole job. Vectors are simply the technique that makes "find the relevant ones" possible at scale, based on *meaning* rather than exact keyword matches.

## What a vector (embedding) actually is

A vector, in this context, is just **a list of numbers that represents the meaning of a piece of text.**

```
"The cat sat on the mat"  →  [0.021, -0.184, 0.402, ..., 0.077]   (often 384, 768, or 1536 numbers long)
```

That list of numbers is called an **embedding**. It's produced by a specially trained model (an **embedding model**, separate from a chat model) whose entire job is: read text in, output a fixed-length list of numbers that captures what the text *means*.

The property that makes this useful: **texts with similar meaning end up with similar numbers.** "The cat sat on the mat" and "A feline rested on the rug" will produce two embeddings that are numerically close to each other, even though they don't share a single word. Meanwhile "The stock market fell sharply today" will produce an embedding that's numerically far away from both. This is the entire trick — you've converted "does this mean the same thing" into "are these two lists of numbers close to each other," which is something a computer can calculate directly and extremely fast, even across millions of pieces of text.

Each number in the vector doesn't correspond to something human-readable like "positivity" or "topic" in any clean, labeled way — it's a compressed, learned representation. You don't need to interpret the individual numbers; you only need the *distance between two vectors* to be meaningful, and that's what the embedding model is trained to guarantee.

## How "closeness" is actually measured

A few standard math methods for comparing two vectors, from most to least commonly used in this context:

- **Cosine similarity** — measures the *angle* between two vectors, ignoring their length. This is the default almost everywhere in RAG systems, because it cares about *direction* (meaning) and not magnitude. Result ranges from -1 to 1, where 1 means "pointing in exactly the same direction" (same meaning).
- **Euclidean distance** — the straight-line distance between two points, the way you'd measure distance on a map. Smaller = more similar.
- **Dot product** — related to cosine similarity but also affected by vector length; some embedding models are specifically tuned to work best with this metric instead.

You don't typically implement any of this math yourself — the vector database does it internally, and you just tell it which metric to use (Spring AI defaults to cosine similarity almost everywhere, and it's the right default to leave alone unless you have a specific reason not to).

## The full pipeline, end to end

There are genuinely two separate phases, and keeping them mentally separate avoids a lot of confusion:

### Phase 1 — Ingestion (happens once, or whenever your source data changes)

```
Raw documents (PDFs, Markdown, database rows...)
        │
        ▼
  DocumentReader        ── "Extract": load raw content into Spring AI's Document objects
        │
        ▼
  DocumentTransformer    ── "Transform": split large documents into smaller chunks
   (e.g. TokenTextSplitter)
        │
        ▼
  EmbeddingModel          ── convert each chunk's text into a vector
        │
        ▼
  VectorStore.add(...)    ── "Load": store the chunk's text + its vector + any metadata
```

This Extract → Transform → Load shape is exactly Spring AI's own **ETL pipeline** terminology for this process, and it maps directly onto three interfaces: `DocumentReader`, `DocumentTransformer`, `DocumentWriter` (which `VectorStore` itself implements).

**Why chunking matters:** you don't embed a whole 50-page PDF as one vector — a single embedding can only represent so much meaning before it becomes a mushy average of everything in the document, and it becomes far less useful for finding a specific answer to a specific question. Instead, you split documents into smaller pieces (a paragraph, a few hundred tokens) using something like `TokenTextSplitter`, and embed *each chunk* separately. This is also why the chunk size is a real design decision: too small and you lose context around the answer; too large and the embedding is diluted and less precise, and you spend more on embedding + retrieval tokens than you need to.

### Phase 2 — Retrieval (happens on every single user question)

```
User's question: "What's our refund policy?"
        │
        ▼
  EmbeddingModel   ── embed the question itself, using the SAME embedding model used at ingestion time
        │
        ▼
  VectorStore.similaritySearch(...)  ── find the closest stored chunks
        │
        ▼
  Top-K matching chunks, as Document objects
        │
        ▼
  Stuffed into the prompt as context   ── this is exactly what QuestionAnswerAdvisor does automatically (file 7)
        │
        ▼
  Sent to the chat model, which now has the actual relevant text to answer from
```

**The one rule you cannot break:** the embedding model used to embed your documents at ingestion time and the embedding model used to embed the incoming question at query time **must be the same model** (or at least produce vectors of the same dimensionality using the same underlying representation). Swap embedding models later, and every previously stored vector becomes meaningless relative to new queries — you have to re-embed and re-ingest everything from scratch. This is a real operational gotcha worth planning around before you pick a model, not after.

## Spring AI's actual interfaces

**`Document`** — the unit everything is built around:
```java
new Document("Spring AI rocks!", Map.of("source", "readme.md", "category", "docs"));
```
Content plus a free-form metadata map — that metadata is what powers filtering later (the `FILTER_EXPRESSION` you saw on `QuestionAnswerAdvisor` in file 7 filters on exactly these fields).

**`EmbeddingModel`** — turns text into vectors:
```java
public interface EmbeddingModel extends Model<EmbeddingRequest, EmbeddingResponse> {
    float[] embed(Document document);
    default float[] embed(String text) { ... }
    default List<float[]> embed(List<String> texts) { ... }
}
```
You'll rarely call `embed()` directly yourself in application code — the `VectorStore` calls it internally on your behalf when you `add()` documents or run a similarity search. It's useful to know it's there, though, for debugging or if you ever need raw vectors for something custom.

**`VectorStore`** — storage plus search:
```java
public interface VectorStore extends DocumentWriter {
    void add(List<Document> documents);
    void delete(List<String> ids);
    void delete(FilterExpression filterExpression);
    List<Document> similaritySearch(SearchRequest request);
}
```

**`SearchRequest`** — how you tune a single query:
```java
SearchRequest request = SearchRequest.builder()
    .query("What's our refund policy?")
    .topK(5)                          // how many matching chunks to return
    .similarityThreshold(0.75)        // discard weak matches below this score
    .filterExpression("category == 'policies'")  // restrict which documents are even eligible
    .build();

List<Document> matches = vectorStore.similaritySearch(request);
```

Notice this is exactly the same `SearchRequest` shape you configured on `QuestionAnswerAdvisor` back in file 7 — the advisor is really just a thin wrapper that runs this search for you automatically and stitches the results into the prompt.

## A full working setup — PGVector

PGVector is a pragmatic first choice for most Java/Spring shops specifically because you probably already run PostgreSQL — no new infrastructure to stand up, and your relational data and your vector data live in the same database.

**Dependencies:**
```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-vector-store-pgvector</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId>  <!-- provides the EmbeddingModel -->
</dependency>
```

**Configuration:**
```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: myuser
    password: mypassword
  ai:
    vectorstore:
      pgvector:
        initialize-schema: true       # let Spring AI create the table/extension for you
        index-type: HNSW
        distance-type: COSINE_DISTANCE
        dimensions: 1536              # must match your embedding model's output size
```

`dimensions` has to match whatever your chosen `EmbeddingModel` actually produces — OpenAI's common embedding models default to 1536, but this varies by model and provider, so check the specific model you're using rather than assuming.

**Ingesting documents:**
```java
@Component
public class DocumentLoader {
    private final VectorStore vectorStore;

    public DocumentLoader(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }

    public void load(Resource pdfResource) {
        var reader = new PagePdfDocumentReader(pdfResource);
        var splitter = new TokenTextSplitter();  // sensible default chunk size out of the box
        List<Document> chunks = splitter.apply(reader.get());
        vectorStore.add(chunks);
    }
}
```

**Querying it** — either directly, or (far more commonly) through `QuestionAnswerAdvisor` as shown in file 7:
```java
List<Document> matches = vectorStore.similaritySearch(
    SearchRequest.builder().query("refund policy").topK(3).build());
```

## Practical judgment calls you'll actually have to make

- **Chunk size** — a few hundred tokens per chunk is a common, reasonable starting point. Tune based on your content: dense technical docs often want smaller chunks; narrative/conversational content can tolerate larger ones. Change this by testing retrieval quality on real questions, not by guessing once and never revisiting it.
- **`topK` and `similarityThreshold` together** — `topK` caps *how many* results come back; `similarityThreshold` filters out weak matches even within that top K. A low threshold with a high `topK` risks stuffing the prompt with marginally-relevant chunks (hurting both quality and your token bill, per file 5); too strict a threshold risks retrieving nothing at all for a valid question phrased unusually. Tune both together against real examples.
- **Embedding cost is real but usually small relative to chat costs** — embedding models are typically far cheaper per token than chat models, but if you're re-ingesting large document sets frequently, it's still worth tracking as its own line item.
- **Local, cost-free embedding models exist** — options like locally-run ONNX-based embedding models (e.g., via `spring-ai-transformers`) let you generate embeddings without an external API call at all, trading a small amount of setup complexity for zero marginal embedding cost and no data leaving your infrastructure. Worth it once volume justifies it.

## How this ties back to everything else you've already learned

- **File 4 / file 7's `QuestionAnswerAdvisor` and `RetrievalAugmentationAdvisor`** are entirely built on the `VectorStore.similaritySearch(...)` call described here — they just automate "embed the question, search, inject results" so you don't write that loop by hand.
- **File 7's `VectorStoreChatMemoryAdvisor`** uses the exact same mechanism, just storing *conversation turns* as the documents instead of your knowledge base — which is why it can retrieve "the most relevant past exchange," not just "the most recent one."
- **File 3's prompt engineering** still applies once the retrieved context is in hand — how you tell the model to use that context (cite it, only answer from it, say "I don't know" if it's not there) is a prompting decision layered on top of the retrieval mechanism this file describes.

Vectors and vector stores aren't really a separate "AI feature" from everything you've already learned — they're the plumbing that makes the RAG advisors from file 7 possible in the first place.
