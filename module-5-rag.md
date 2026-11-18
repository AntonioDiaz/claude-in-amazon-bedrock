<h1>Module 5 — Retrieval Augmented Generation (RAG)</h1>

<h2>Contents</h2>

- [1. Introducing Retrieval Augmented Generation](#1-introducing-retrieval-augmented-generation)
  - [Two ways to give Claude the document](#two-ways-to-give-claude-the-document)
  - [How RAG works](#how-rag-works)
  - [Challenges](#challenges)
  - [When to use RAG](#when-to-use-rag)
- [2. Text chunking strategies](#2-text-chunking-strategies)
  - [Why it matters](#why-it-matters)
  - [The three strategies](#the-three-strategies)
  - [Size-based chunking and overlap](#size-based-chunking-and-overlap)
  - [Structure-based chunking](#structure-based-chunking)
  - [Implementations](#implementations)
  - [Choosing a strategy](#choosing-a-strategy)
- [3. Text embeddings](#3-text-embeddings)
  - [What an embedding is](#what-an-embedding-is)
  - [Why it matters for RAG](#why-it-matters-for-rag)
  - [Generating embeddings on Bedrock](#generating-embeddings-on-bedrock)
- [4. The full RAG flow](#4-the-full-rag-flow)
  - [Step 1 · Chunk the source text](#step-1--chunk-the-source-text)
  - [Step 2 · Embed and normalize](#step-2--embed-and-normalize)
  - [Step 3 · Store in a vector database](#step-3--store-in-a-vector-database)
  - [Step 4 · Embed the user query](#step-4--embed-the-user-query)
  - [Step 5 · Find the most similar chunks](#step-5--find-the-most-similar-chunks)
  - [Step 6 · Build the final prompt](#step-6--build-the-final-prompt)
  - [The complete flow](#the-complete-flow)
- [5. Implementing the RAG flow](#5-implementing-the-rag-flow)
  - [The `VectorIndex` class](#the-vectorindex-class)
  - [The five steps in code](#the-five-steps-in-code)
  - [Why store the text with the vector](#why-store-the-text-with-the-vector)
  - [Reading the results](#reading-the-results)
- [6. BM25 lexical search](#6-bm25-lexical-search)
  - [When semantic search falls short](#when-semantic-search-falls-short)
  - [Hybrid search](#hybrid-search)
  - [How BM25 works](#how-bm25-works)
  - [Using `BM25Index`](#using-bm25index)
- [7. A multi-search RAG pipeline](#7-a-multi-search-rag-pipeline)
- [8. Reranking results](#8-reranking-results)
- [9. Contextual retrieval](#9-contextual-retrieval)
- [Common pitfalls](#common-pitfalls)

---

## 1. Introducing Retrieval Augmented Generation

**RAG** is a way of working with documents that are too big to hand to Claude in one go: split the
document into small pieces ahead of time and, for each question, send Claude **only the pieces that
matter**.

Running example: an 800-page financial report and the question *"What risk factors does this company
have?"*. The answer is somewhere in the report; the problem is getting that part in front of Claude.

### Two ways to give Claude the document

| | Option 1 · Everything in the prompt | Option 2 · Chunks + retrieval (RAG) |
|---|---|---|
| **How** | Extract all the text and paste it into one prompt | Pre-split the document, find the relevant chunks, send only those |
| **Size limit** | Hard token limit: a very long document just doesn't fit | Scales to very large documents and whole collections |
| **Answer quality** | Drops as the prompt grows: too much to sift through | Claude concentrates on the relevant content |
| **Cost / latency** | Bigger prompt → more expensive and slower | Smaller prompt → cheaper and faster |
| **Complexity** | None | Preprocessing + a search mechanism |

### How RAG works

1. **Chunk** — split the document into smaller pieces (Strategy Outlook, Risk Factors, Balance Sheet…).
2. **Understand the question** — work out what the user is looking for.
3. **Retrieve** — find the chunks most relevant to that question.
4. **Generate** — put only those chunks in the prompt and let Claude answer.

```mermaid
flowchart LR
    subgraph PRE[Preprocessing, done once]
        D[Financial report] --> S[Split into chunks]
        S --> C1[Strategy Outlook]
        S --> C2[Risk Factors]
        S --> C3[Balance Sheet]
    end
    Q[User question: what are the risk factors?] --> F[Find relevant chunks]
    C1 -.-> F
    C2 --> F
    C3 -.-> F
    F --> P[Prompt = question + Risk Factors chunk]
    P --> CL[Claude]
    CL --> A[Answer]
```

For the risk question, only the "Risk Factors" chunk reaches Claude: focused context instead of 800
pages.

### Challenges

RAG isn't free; it moves the difficulty somewhere else:

- **Preprocessing step** — documents have to be chunked before any question can be answered.
- **Search mechanism** — you need a way to decide which chunks are relevant.
- **Missing context** — a retrieved chunk may not contain everything needed to answer.
- **Chunking strategy** — equal-sized portions, by headers/sections, by semantic meaning… each has
  trade-offs, and the right one depends on the use case.

### When to use RAG

| Use RAG when… | Stay with "everything in the prompt" when… |
|---|---|
| The document (or collection) doesn't fit, or barely fits, in a prompt | The content fits comfortably |
| Questions are specific and need only a part of the content | Questions need the document as a whole |
| You run many queries and care about cost and response time | Volume is low and simplicity matters more |

> RAG is a trade-off, not a default: chunking, search and retrieval are extra moving parts. Use it when
> that overhead buys you scale, speed or lower cost.

## 2. Text chunking strategies

Chunking is the step that decides the quality of the whole RAG pipeline: whatever lands in a chunk is
what Claude gets to see. Bad chunks put **irrelevant context** in the prompt, and Claude answers
confidently from the wrong material.

### Why it matters

Example: a report with a *medical research* section and a *software engineering* section. The user asks
*"How many bugs did engineers fix this year?"*. If the chunks are poorly cut, retrieval can return a
medical chunk just because it mentions a "bug" (in the biological sense), and the answer is wrong.

> Goal: chunks that are **semantically coherent** (one topic each) and that still make sense when read
> on their own.

### The three strategies

| Strategy | How it splits | Pros | Cons |
|---|---|---|---|
| **Size-based** | Fixed number of characters/words | Simple, works on any text | Cuts words and sentences, separates related content |
| **Structure-based** | Headers, paragraphs, sections | Chunks follow the document's own organization | Needs a guaranteed, consistent format (e.g. Markdown) |
| **Semantic-based** | Groups related sentences using NLP | Most meaningful chunks | Most complex and expensive to build |

### Size-based chunking and overlap

Split the text into pieces of roughly equal length. Reliable, but blind to meaning: a sentence can be
cut in half and its context lost.

**Overlap** mitigates that: each chunk repeats a few characters (or sentences) of its neighbour, so
information at a boundary appears complete in at least one chunk. It costs some duplication, which is
usually worth it.

```mermaid
flowchart LR
    T[Text] --> A[Chunk 1: chars 0 to 150]
    T --> B[Chunk 2: chars 130 to 280]
    T --> C[Chunk 3: chars 260 to 410]
    A -. 20 shared chars .- B
    B -. 20 shared chars .- C
```

### Structure-based chunking

If the documents have a reliable structure (Markdown with `##` headers, for example), split on it: each
chunk is a real section. Excellent results on well-formatted documents; unreliable on plain text or
inconsistent formatting.

### Implementations

**By characters**, with overlap:

```python
def chunk_by_char(text, chunk_size=150, chunk_overlap=20):
    chunks = []
    start_idx = 0

    while start_idx < len(text):
        end_idx = min(start_idx + chunk_size, len(text))
        chunks.append(text[start_idx:end_idx])

        # step back `chunk_overlap` chars, unless we already reached the end
        start_idx = end_idx - chunk_overlap if end_idx < len(text) else len(text)

    return chunks
```

**By sentences**, with sentence overlap:

```python
import re

def chunk_by_sentence(text, max_sentences_per_chunk=5, overlap_sentences=1):
    sentences = re.split(r"(?<=[.!?])\s+", text)
    chunks = []
    start_idx = 0

    while start_idx < len(sentences):
        end_idx = min(start_idx + max_sentences_per_chunk, len(sentences))
        chunks.append(" ".join(sentences[start_idx:end_idx]))
        start_idx += max_sentences_per_chunk - overlap_sentences

    return chunks
```

> ⚠️ If `overlap_sentences >= max_sentences_per_chunk`, `start_idx` never advances and the loop is
> infinite. The lesson's `if start_idx < 0` guard doesn't prevent this; validate
> `overlap_sentences < max_sentences_per_chunk` instead. The regex also splits on abbreviations and
> decimals ("e.g. ", "v2. "), so it's a poor fit for code or technical text.

**By sections** (Markdown `##` headers):

```python
import re

def chunk_by_section(document_text):
    return re.split(r"\n## ", document_text)
```

> Note: `re.split` removes the separator, so each chunk loses its `## ` prefix, and the first chunk is
> whatever comes before the first `##` (the document title). Use a lookahead, `re.split(r"\n(?=## )",
> document_text)`, to keep the headers.

### Choosing a strategy

| Strategy | Best for | Watch out for |
|---|---|---|
| Character-based | User-uploaded documents with no format guarantee; the safe fallback | Cut sentences; tune size and overlap |
| Sentence-based | Prose: good balance between context and meaning | Code and technical text with unexpected periods |
| Section-based | Well-structured internal docs (Markdown, consistent headers) | Breaks if the structure isn't there; sections can be too long |

Treat chunking as **iterative**: start simple, test with your real documents and questions, and refine
based on the quality of what retrieval returns.

> On AWS, **Bedrock Knowledge Bases** offers these strategies as managed options when you create a data
> source: fixed-size (with overlap), hierarchical, semantic, or no chunking.

📓 [001_chunking.ipynb](exercises/module_5/001_chunking.ipynb) · data: [report.md](exercises/module_5/report.md)

## 3. Text embeddings

With the document chunked, the next problem is **search**: given the user's question, which chunks are
related to it? Plain keyword matching isn't enough (remember the "bug" example), so the usual answer is
**semantic search**, built on **text embeddings**.

### What an embedding is

An **embedding** is a list of numbers that represents the *meaning* of a piece of text, so that a
computer can compare texts with math instead of words.

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/e4014a1a-17cd-463e-a7dc-6a53f4dc1d81" />

```mermaid
flowchart LR
    T[Text chunk] --> M[Embedding model]
    M --> V[Vector of 1024 numbers]
```

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/341b233f-d339-4deb-8e86-0837069b85eb" />

- The model returns a fixed-length vector (1024 numbers in this course).
- Each number can be thought of as a **score for some quality** of the text: "how happy is it?", "how
  much does it talk about oceans?"…
- In practice nobody knows what each position means: the model learns those features in training and
  they're **not human-interpretable**. The "score" idea is only a mental model.

> The lesson says the values range from -1 to +1. That holds for **normalized** vectors (length 1, as
> with `normalize=True` below); without normalization the values aren't bounded that way.

### Why it matters for RAG

**Similar meaning → similar vectors.** So you can embed the question, embed every chunk, and compare
them numerically to find the closest chunks, even when they don't share any words with the question.
That's what makes semantic search better than keyword matching for picking RAG context.

| | Keyword matching | Semantic search (embeddings) |
|---|---|---|
| Compares | Exact words | Meaning |
| "bug" in a medical vs software text | Same match | Different vectors |
| Synonyms / paraphrases | Missed | Found |

### Generating embeddings on Bedrock

Embedding models are called with **`InvokeModel`**, not `Converse` (Converse is for chat models). The
request and response body are model-specific; for **Amazon Titan Text Embeddings V2**:

```python
import json
import boto3

client = boto3.client("bedrock-runtime", region_name="us-west-2")

def generate_embedding(
    text,
    embedding_model_id="amazon.titan-embed-text-v2:0",
    dimensions=1024,
    normalize=True,
):
    request_body = {
        "inputText": text,
        "dimensions": dimensions,   # Titan V2: 256, 512 or 1024
        "normalize": normalize,     # unit-length vector, good for cosine similarity
    }

    response = client.invoke_model(
        modelId=embedding_model_id,
        body=json.dumps(request_body),
        accept="application/json",
        contentType="application/json",
    )

    response_body = json.loads(response["body"].read())
    return response_body["embedding"]   # list of `dimensions` floats
```

Running it on the first chunk of `report.md` returns 1024 floats such as
`[-0.0583, 0.0117, 0.0318, ...]`.

> - You may need to enable access to the Titan embeddings model in the Bedrock console first.
> - The lesson says V1 (`amazon.titan-embed-text-v1`) works just as well. It does, but **not with this
>   body**: V1 only accepts `inputText` (no `dimensions` or `normalize`) and always returns 1536 numbers.

📓 [002_embeddings.ipynb](exercises/module_5/002_embeddings.ipynb) · data: [report.md](exercises/module_5/report.md)

## 4. The full RAG flow

This lesson puts chunking and embeddings together and walks through the whole pipeline with a toy
example small enough to check by hand.

```mermaid
flowchart TB
    subgraph PRE[Preprocessing, ahead of time]
        A[1 · Chunk source text] --> B[2 · Embed + normalize each chunk]
        B --> C[3 · Store in vector database]
    end
    subgraph Q[At query time]
        D[4 · Embed the user query] --> E[5 · Find closest chunks: cosine similarity]
        E --> F[6 · Build prompt: question + chunks]
        F --> G[Claude answers]
    end
    C --> E
```

### Step 1 · Chunk the source text

Two chunks, both chosen to be ambiguous on purpose:

| Chunk | Text (summary) | Trap |
|---|---|---|
| **Medical research** | Progress understanding XDR-47, a "bug" never seen before | "bug" sounds like software |
| **Software engineering** | Studying "infection vectors" in distributed systems | "infection" sounds medical |

### Step 2 · Embed and normalize

Imagine a perfect embedding model that returns **2 numbers** with known meaning:
`[how medical, how software]`.

| Chunk | Raw embedding | Normalized (length 1) |
|---|---|---|
| Medical research | `[0.97, 0.34]` | `[0.944, 0.331]` |
| Software engineering | `[0.30, 0.97]` | `[0.295, 0.955]` |

**Normalization** scales each vector to magnitude 1.0, so every point lies on the unit circle and only
the *direction* matters. Embedding APIs usually do it for you (Titan's `normalize=True`).

### Step 3 · Store in a vector database

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/a0a028b1-6b74-4f64-8670-0f82a3f410bf" />

The normalized vectors go into a **vector database**: a store optimized to keep long lists of numbers
and search for the closest ones.

> Up to here everything is **preprocessing**, done once ahead of time. The rest happens per query.

### Step 4 · Embed the user query

Query: *"I'm curious about the company. In particular, what did the software engineering dept do this
year?"*. It goes through **the same embedding model** (vectors from different models can't be
compared): `[0.1, 0.89]` → normalized `[0.112, 0.994]`.

### Step 5 · Find the most similar chunks

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/6af8d05f-237f-481a-b121-153786c7e2ee" />

The vector database returns the stored vectors closest to the query vector, measured with **cosine
similarity**: the cosine of the angle between two vectors.

$$\cos(\theta) = \frac{A \cdot B}{\|A\| \, \|B\|}$$

| Cosine similarity | Meaning |
|---|---|
| ≈ 1 | Same direction → very similar |
| ≈ 0 | Perpendicular → unrelated |
| ≈ -1 | Opposite |

With normalized vectors both magnitudes are 1, so the cosine is just the **dot product**:

| Query vs… | Cosine similarity | |
|---|---|---|
| Software engineering | **0.982** | ✅ best match |
| Medical research | 0.434 | |

> The lesson gives 0.983 and **0.398**. Recomputing from its own vectors gives ≈ 0.982 and **0.434**
> (and the normalized query is `[0.112, 0.994]`, not `0.993`). The conclusion doesn't change.

```python
import math

def normalize(v):
    norm = math.sqrt(sum(x * x for x in v))
    return [x / norm for x in v]

def cosine_similarity(a, b):
    return sum(x * y for x, y in zip(a, b))   # a and b already normalized

medical  = normalize([0.97, 0.34])
software = normalize([0.30, 0.97])
query    = normalize([0.10, 0.89])

cosine_similarity(query, software)   # 0.982
cosine_similarity(query, medical)    # 0.434
```

**Cosine distance** = `1 - cosine similarity`. Vector databases often use it instead: **0 = identical**,
bigger = less similar (so you sort ascending).

| Measure | Most similar | Sort |
|---|---|---|
| Cosine similarity | Close to 1 | Descending |
| Cosine distance | Close to 0 | Ascending |

### Step 6 · Build the final prompt

Combine the user's question with the retrieved chunk(s) and send it to Claude, which now answers from
the document instead of from its general knowledge. In Converse format:

```python
prompt = f"""
Answer the user's question using the report excerpt below.

<user_question>
{user_question}
</user_question>

<report_excerpt>
{software_engineering_chunk}
</report_excerpt>
"""

messages = [{"role": "user", "content": [{"text": prompt}]}]
response = client.converse(modelId=model_id, messages=messages)
```

### The complete flow

1. Chunk the source documents.
2. Embed each chunk (normalized).
3. Store the embeddings in a vector database.
4. Embed the user's question with the same model.
5. Retrieve the most similar chunks (cosine similarity / distance).
6. Put the chunks and the question in the prompt.
7. Send it to Claude.

> Knowing the math behind steps 4–5 is what lets you debug a RAG system when it retrieves the wrong
> chunks.

## 5. Implementing the RAG flow

Now the flow from the previous lesson, in code, over `report.md`. Steps 1–3 are preprocessing; steps
4–5 run per question. The vector database is a small in-memory class, `VectorIndex`, so you can see
exactly what a real one does.

```mermaid
flowchart LR
    R[report.md] --> C[1 · chunk_by_section]
    C --> E[2 · generate_embedding per chunk]
    E --> S[3 · VectorIndex: vector + text]
    Q[User question] --> QE[4 · generate_embedding]
    QE --> SR[5 · store.search k=2]
    S --> SR
    SR --> RES[Top chunks + cosine distance]
```

### The `VectorIndex` class

A brute-force vector store: two parallel lists, and a search that compares the query against every
stored vector.

| Method | What it does |
|---|---|
| `add_vector(vector, document)` | Stores the vector and its document (a dict that must have `content`). Checks all vectors have the same dimension. |
| `add_document(document)` | Same, but computes the vector itself with the `embedding_fn` passed to the constructor. |
| `search(query, k)` | `query` can be a vector, or a string if there's an `embedding_fn`. Returns the `k` closest `(document, distance)` pairs, sorted by distance ascending. |
| `_cosine_distance` / `_euclidean_distance` | The metric, chosen with `distance_metric="cosine"` (default) or `"euclidean"`. |

The core of it:

```python
def _cosine_distance(self, vec1, vec2):
    mag1, mag2 = self._magnitude(vec1), self._magnitude(vec2)
    if mag1 == 0 and mag2 == 0:
        return 0.0
    if mag1 == 0 or mag2 == 0:
        return 1.0
    similarity = self._dot_product(vec1, vec2) / (mag1 * mag2)
    similarity = max(-1.0, min(1.0, similarity))   # guard against float rounding
    return 1.0 - similarity                        # cosine distance: 0 = identical

def search(self, query_vector, k=1):
    distances = [
        (self._cosine_distance(query_vector, v), doc)
        for v, doc in zip(self.vectors, self.documents)
    ]
    distances.sort(key=lambda item: item[0])       # smallest distance first
    return [(doc, dist) for dist, doc in distances[:k]]
```

> Simplified: the notebook's `search` also validates the input, accepts a string query and lets you
> pick the metric.

### The five steps in code

```python
# 1. Chunk the text by section
with open("./report.md", "r") as f:
    text = f.read()
chunks = chunk_by_section(text)

# 2. Embed every chunk (one Bedrock call per chunk, so it takes a while)
embeddings = [generate_embedding(chunk) for chunk in chunks]

# 3. Store each embedding together with its text
store = VectorIndex()
for embedding, chunk in zip(embeddings, chunks):
    store.add_vector(embedding, {"content": chunk})

# 4. Later, a user asks a question: embed it with the same model
user_embedding = generate_embedding("What did the software engineering dept do last year?")

# 5. Retrieve the 2 closest chunks
results = store.search(user_embedding, 2)
for doc, distance in results:
    print(distance, "\n", doc["content"][0:200], "\n")
```

`chunk_by_section` and `generate_embedding` are the functions from sections 2 and 3.

### Why store the text with the vector

The search compares numbers, but what you need afterwards is **the text** to put in Claude's prompt. A
vector on its own is useless, so each entry stores the embedding **plus metadata** (`{"content":
chunk}`, and in a real system also the source, section, page…).

### Reading the results

`search` returns `(document, cosine distance)` pairs: **lower distance = more relevant**.

| Query | 1st result | 2nd result |
|---|---|---|
| *"What did the software engineering dept do last year?"* (lesson) | Software Engineering · 0.71 | Methodology · 0.72 |
| *"what happened with INC-2023-Q4-011?"* (my notebook) | Cybersecurity (INC-2023-Q4-011) · 0.63 | Financial Analysis · 0.81 |

> The lesson and the completed notebook use different questions, so the numbers differ; the table shows
> both.

> In the lesson's run, the right section (0.71) and an unrelated one (0.72) are almost tied: semantic
> search alone isn't very decisive. That's what the next lessons improve (BM25, multi-search,
> reranking).

> The lesson stops at retrieval. To finish the RAG flow, the retrieved `doc["content"]` goes into the
> prompt with the question and is sent to Claude, as in [step 6 of the previous
> section](#step-6--build-the-final-prompt).

> `VectorIndex` is a learning tool: it compares against every vector (O(n)) and lives in memory. On AWS
> the managed options are Bedrock Knowledge Bases on top of a vector store such as OpenSearch
> Serverless, Aurora PostgreSQL (pgvector) or S3 Vectors.

📓 [003_vectordb.ipynb](exercises/module_5/003_vectordb.ipynb) (exercise) ·
[003_vectordb_completed.ipynb](exercises/module_5/003_vectordb_completed.ipynb) (solution) · data:
[report.md](exercises/module_5/report.md)

## 6. BM25 lexical search

Semantic search finds text with a similar *meaning*, but it can miss **exact terms**: IDs, codes,
product names. The fix is to add a classic **lexical** (keyword) search, BM25, and combine both.

### When semantic search falls short

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/b156585a-affe-4482-9649-14754004dccc" />

Query: *"What happened with INC-2023-Q4-011?"*. The incident ID appears literally in the sections that
discuss it, yet semantic search can rank sections that are only "similar in topic" above them, because
embeddings capture meaning, not exact strings.

> In my run of [section 5](#5-implementing-the-rag-flow) semantic search did put Cybersecurity first
> (0.63), but the 2nd result was Financial Analysis (0.81), which doesn't mention the incident, and it
> missed Software Engineering, which does.

### Hybrid search

Run both searches **in parallel** over the same chunks and **merge** the results (the merge is the next
lesson):

```mermaid
flowchart LR
    Q[User query] --> S[Semantic search: embeddings]
    Q --> L[Lexical search: BM25]
    S --> M[Merge results]
    L --> M
    M --> R[Final ranked chunks]
```

| | Semantic (embeddings) | Lexical (BM25) |
|---|---|---|
| Matches | Meaning, synonyms, paraphrases | Exact words |
| Strong at | Vague or conceptual questions | IDs, codes, names, rare terms |
| Weak at | Exact identifiers | Synonyms, different wording |
| Needs | Embedding model + vector store | Just the text (no model) |

### How BM25 works

**BM25** (*Best Match 25*) scores each chunk against the query:

1. **Tokenize** the query into terms.
2. **Count** two things: how often each term appears *in each chunk* (term frequency) and *in how many
   chunks* it appears (document frequency).
3. **Weight by rarity** (IDF): terms that appear in few chunks are worth a lot, terms that appear
   everywhere ("the", "what") are worth almost nothing.
4. **Score** each chunk: more occurrences of high-weight terms → higher score.

The formula used in the notebook ($N$ = number of chunks, $n_t$ = chunks containing term $t$,
$f_{t,d}$ = occurrences of $t$ in chunk $d$):

$$\text{IDF}(t) = \ln\left(\frac{N - n_t + 0.5}{n_t + 0.5} + 1\right)$$

$$\text{score}(q, d) = \sum_{t \in q} \text{IDF}(t) \cdot \frac{f_{t,d}\,(k_1 + 1)}{f_{t,d} + k_1\left(1 - b + b\,\frac{|d|}{\text{avgdl}}\right)}$$

| Parameter | Default | Effect |
|---|---|---|
| `k1` | 1.5 | Term-frequency saturation: the 10th occurrence adds much less than the 1st |
| `b` | 0.75 | Length normalization: long chunks don't win just for having more words |

### Using `BM25Index`

Same API as `VectorIndex` (`add_document` + `search`), so both stores can be used side by side. No
embeddings needed: it indexes the raw text.

```python
# 1. Chunk the text by section
chunks = chunk_by_section(text)

# 2. Create a BM25 store and add each chunk
store = BM25Index()          # k1=1.5, b=0.75
for chunk in chunks:
    store.add_document({"content": chunk})

# 3. Search
results = store.search("What happened with INC-2023-Q4-011?", 3)
for doc, score in results:
    print(score, "\n", doc["content"][0:200], "\n----\n")
```

Results in my notebook:

| # | Section | Score |
|---|---|---|
| 1 | Software Engineering (mentions the incident) | 0.27 |
| 2 | Cybersecurity, INC-2023-Q4-011 | 0.33 |
| 3 | Methodology | 0.94 |

Both sections that actually talk about the incident come first; Financial Analysis is gone.

> **The score is "lower is better".** Raw BM25 is "higher is better", but `search` converts it with
> `exp(-0.1 · raw_score)` so it behaves like a distance (≈ 0 = great match, ≈ 1 = poor), the same as
> `VectorIndex`. That makes the two lists easier to merge later.

> **Tokenization detail:** the default tokenizer lowercases and splits on non-word characters, so
> `INC-2023-Q4-011` becomes `inc`, `2023`, `q4`, `011`. It's not matched as one term; it works because
> those tokens are rare and appear together. A custom `tokenizer` can keep IDs whole.

📓 [004_bm25.ipynb](exercises/module_5/004_bm25.ipynb) (exercise) ·
[004_bm25_completed.ipynb](exercises/module_5/004_bm25_completed.ipynb) (solution) · data:
[report.md](exercises/module_5/report.md)

## 7. A multi-search RAG pipeline

_Pending._

## 8. Reranking results

_Pending._

## 9. Contextual retrieval

_Pending._

---

## Common pitfalls

_Pending._

---

[← Back to README](README.md)
