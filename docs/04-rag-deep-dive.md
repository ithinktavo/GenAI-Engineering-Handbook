# RAG Deep Dive

## The Full Pipeline from Document to Answer

---

## What This Guide Covers

This is the engineering deep dive into Retrieval-Augmented Generation. If you've read [01 — From Prompt Engineering to Context Engineering](01-prompt-engineering-to-context-engineering.md), you saw the RAG pipeline at a high level. This guide takes you inside each component — chunking strategies, embedding models, vector databases, retrieval techniques, and reranking — with the decision frameworks you need to choose the right approach for production systems.

**The analogy:** RAG is a librarian. Not the kind that points you to the right shelf — the kind that reads every relevant page across every relevant book, pulls out the exact paragraphs you need, and hands them to you with citations. The AI then reads those paragraphs and writes your answer.

---

## The RAG Pipeline: End to End

There are two paths in every RAG system — the **indexing path** (one-time, when data changes) and the **query path** (every time a user asks a question):

```mermaid
graph TD
    subgraph INDEX["📥 INDEXING PATH — offline"]
        D1["📁 DOCUMENTS\nPDFs, DBs, APIs, wikis"]
        D2["✂️ CHUNKING\nSplit into small pieces"]
        D3["🔢 EMBEDDING\nConvert to vectors"]
        D4["🗄️ VECTOR DB\nStore and index"]
        D1 --> D2 --> D3 --> D4
    end

    subgraph QUERY["🔍 QUERY PATH — every question"]
        Q1["❓ USER QUESTION\n'What was Q3 revenue?'"]
        Q2["🔢 EMBEDDING\nConvert query to vector"]
        Q3["🔍 RETRIEVAL\nFind nearest vectors in DB"]
        Q4["🎯 RERANKING\nScore and reorder"]
        Q5["📋 PROMPT ASSEMBLY\nSystem prompt + context + query"]
        Q6["🤖 LLM\nGenerate answer grounded in data"]
        Q1 --> Q2 --> Q3 --> Q4 --> Q5 --> Q6
    end

    D4 -.->|similarity search| Q3

    style INDEX fill:#fff3cd,stroke:#ffc107,stroke-width:2px
    style QUERY fill:#f0f4ff,stroke:#2E86C1,stroke-width:2px
    style D1 fill:#fff3cd,stroke:#ffc107
    style D2 fill:#f0f4ff,stroke:#2E86C1
    style D3 fill:#f0f4ff,stroke:#2E86C1
    style D4 fill:#e2d5f1,stroke:#6f42c1
    style Q1 fill:#fff3cd,stroke:#ffc107
    style Q2 fill:#f0f4ff,stroke:#2E86C1
    style Q3 fill:#f0f4ff,stroke:#2E86C1
    style Q4 fill:#f0f4ff,stroke:#2E86C1
    style Q5 fill:#f0f4ff,stroke:#2E86C1
    style Q6 fill:#d4edda,stroke:#28a745
```

The indexing path runs when your data changes (daily, weekly, or on-demand). The query path runs in real-time for every user question. The key insight: **most of the engineering complexity is in the indexing path.** Get that right, and queries are fast and accurate.

---

## Step 1: Document Ingestion

### Where Data Comes From

In enterprise environments, RAG data rarely comes from a single clean source:

| Source Type | Examples | Preprocessing Needed |
|---|---|---|
| **Structured documents** | PDFs, Word docs, slide decks | Extract text, preserve tables, handle images |
| **Databases** | PostgreSQL, SQL Server, Snowflake | Export to text, include schema context |
| **APIs** | Salesforce, ServiceNow, JIRA | Serialize responses, handle pagination |
| **Collaboration tools** | SharePoint, Confluence, Teams | Extract content, strip formatting |
| **Emails & messages** | Outlook, Slack channels | Filter noise, preserve thread context |
| **Code repositories** | GitHub, Azure DevOps | Parse by function/class, include docstrings |

### Preprocessing Pipeline

Before chunking, raw data needs cleaning:

1. **Text extraction** — Convert PDFs, DOCX, HTML to plain text. Tools: `pypdf`, `unstructured`, Azure Document Intelligence.
2. **Deduplication** — Remove duplicate documents (common in SharePoint/Confluence environments).
3. **Metadata extraction** — Pull titles, authors, dates, categories. This metadata becomes critical for filtering during retrieval.
4. **Quality filtering** — Remove boilerplate (footers, headers, navigation text) that would pollute embeddings.

**Enterprise reality:** For the consulting firm's C-suite dashboard, the data pipeline pulls from 4 sources: the financial data warehouse (structured), practice area reports (PDFs), HR system (API), and client CRM (Salesforce via MCP). Each source requires a different extraction strategy.

---

## Step 2: Chunking Strategies

Chunking is where most RAG pipelines succeed or fail. The goal: create pieces small enough to be precise, but large enough to be meaningful.

### Strategy 1: Fixed-Size Chunking

Split text into chunks of N tokens with optional overlap.

```mermaid
graph TD
    DOC["📄 Document: 'Revenue reached $42M in Q3. The defense practice\ngrew 18%. Financial services slowed 5%. Health\nshowed strong recovery at 15% growth.'"]

    subgraph NO_OL["Fixed-size — 50 tokens, no overlap"]
        A1["✂️ Chunk 1: 'Revenue reached $42M in\nQ3. The defense practice grew 18%.'"]
        A2["✂️ Chunk 2: 'Financial services slowed\n5%. Health showed strong recovery\nat 15% growth.'"]
    end

    subgraph OL["Fixed-size — 50 tokens, 20% overlap"]
        B1["✂️ Chunk 1: 'Revenue reached $42M in\nQ3. The defense practice grew 18%.'"]
        B2["🔄 Chunk 2: 'The defense practice grew\n18%. Financial services slowed 5%.\nHealth showed strong recovery...'"]
    end

    DOC --> NO_OL
    DOC --> OL

    style DOC fill:#fff3cd,stroke:#ffc107
    style NO_OL fill:#f0f4ff,stroke:#2E86C1,stroke-width:2px
    style OL fill:#f0f4ff,stroke:#2E86C1,stroke-width:2px
    style A1 fill:#f0f4ff,stroke:#2E86C1
    style A2 fill:#f0f4ff,stroke:#2E86C1
    style B1 fill:#f0f4ff,stroke:#2E86C1
    style B2 fill:#e2d5f1,stroke:#6f42c1
```

**Pros:** Simple, predictable chunk sizes, easy to implement.
**Cons:** Can split sentences mid-thought, ignores document structure.

### Strategy 2: Semantic Chunking

Split at natural boundaries — paragraphs, sections, topic shifts.

**Pros:** Preserves meaning within chunks, respects document structure.
**Cons:** Uneven chunk sizes (some too small, some too large), harder to implement.

### Strategy 3: Overlapping Chunking

Any strategy combined with overlap at boundaries, so context isn't lost between chunks.

**Pros:** Catches information that falls at boundaries.
**Cons:** Increases storage and indexing time (10-20% more chunks).

### Strategy 4: Hierarchical (Parent-Child)

Two levels: parent chunks (full sections) and child chunks (paragraphs within sections). Retrieve the child, include the parent for context.

```mermaid
graph TD
    P["📁 Parent chunk: 'Q3 2025 Financial Performance Summary'\nEntire 3-paragraph section"]
    C1["📄 Child 1: 'Total revenue for Q3 reached $42.3M...'"]
    C2["🎯 Child 2: 'Revenue by practice area: Defense...'"]
    C3["📄 Child 3: 'Gross margin improved to 38.5%...'"]
    R["✅ Return Child 2 + Parent for full context"]

    P --> C1
    P --> C2
    P --> C3
    C2 -->|"🔍 Search matches"| R

    style P fill:#fff3cd,stroke:#ffc107,stroke-width:2px
    style C1 fill:#f0f4ff,stroke:#2E86C1
    style C2 fill:#d4edda,stroke:#28a745,stroke-width:2px
    style C3 fill:#f0f4ff,stroke:#2E86C1
    style R fill:#d4edda,stroke:#28a745
```

**Pros:** Best of both worlds — precise retrieval with broad context.
**Cons:** Complex to implement, requires careful parent-child linking.

### Strategy 5: Sentence-Window

Each chunk is a single sentence, but retrieval returns the sentence plus surrounding sentences (a "window").

**Pros:** Maximum precision in matching. Great for factual lookups.
**Cons:** Tiny chunks can lack context, window expansion adds latency.

### Decision Framework

| Factor | Fixed-Size | Semantic | Overlapping | Hierarchical | Sentence-Window |
|---|---|---|---|---|---|
| **Implementation complexity** | Low | Medium | Low | High | Medium |
| **Retrieval precision** | Medium | High | Medium | Very High | Very High |
| **Context preservation** | Low | High | Medium | Very High | Medium |
| **Storage overhead** | Low | Low | Medium | High | Low |
| **Best for** | Quick prototypes | Most production use | Boundary-sensitive data | Complex docs with sections | Factual Q&A |

> **Recommendation for most enterprise RAG:** Start with semantic chunking + 10-15% overlap. Move to hierarchical if your documents have clear section structure (which most reports do).

For hands-on comparison, see [Notebook 03 — Chunking Strategies Compared](../notebooks/03-chunking-strategies.ipynb).

---

## Step 3: Embedding Models

The embedding model determines how well your system understands meaning. It converts text into high-dimensional vectors where semantic similarity maps to vector proximity.

### How Embeddings Work

```mermaid
graph LR
    T1["📝 'Q3 revenue was $42 million'"] -->|Embed| V1["🔢 [0.023, -0.156, 0.891, ..., 0.034]\n384 to 3,072 dimensions"]
    T2["📝 'Third quarter sales figures'"] -->|Embed| V2["🔢 [0.019, -0.148, 0.887, ..., 0.031]\nSimilar values = similar meaning"]

    V1 <-.->|"✅ High similarity"| V2

    style T1 fill:#fff3cd,stroke:#ffc107
    style T2 fill:#fff3cd,stroke:#ffc107
    style V1 fill:#e2d5f1,stroke:#6f42c1
    style V2 fill:#e2d5f1,stroke:#6f42c1
```

### Model Comparison (2026)

| Model | Dimensions | Cost | Self-Hostable | Best For |
|---|---|---|---|---|
| **OpenAI text-embedding-3-large** | 3,072 | ~$0.13/1M tokens | No | General-purpose, highest quality |
| **OpenAI text-embedding-3-small** | 1,536 | ~$0.02/1M tokens | No | Cost-sensitive production |
| **Cohere embed-v3** | 1,024 | ~$0.10/1M tokens | No | Multilingual, strong retrieval |
| **all-MiniLM-L6-v2** | 384 | Free | Yes | Prototyping, air-gapped environments |
| **bge-large-en-v1.5** | 1,024 | Free | Yes | Production-grade open-source |
| **nomic-embed-text** | 768 | Free | Yes | Long documents (8K context) |

**Enterprise decision factors:**
- **FedRAMP/air-gapped:** Must self-host → open-source models only
- **Cost at scale:** 1M+ chunks → consider open-source or embedding-3-small
- **Multilingual:** Global consulting → Cohere embed-v3
- **Maximum quality:** Budget allows cloud APIs → embedding-3-large

To visualize how embeddings capture meaning, see [Notebook 04 — Embeddings Explorer](../notebooks/04-embeddings-explorer.ipynb).

---

## Step 4: Vector Databases

Vector databases are purpose-built for storing embeddings and running similarity search at scale. They're not general-purpose databases — they're optimized for the specific operation: "given this query vector, find the K nearest vectors."

### How Similarity Search Works

```mermaid
graph TD
    Q["🔍 Query vector: [0.023, -0.156, 0.891, ...]"]

    subgraph VDB["🗄️ VECTOR DATABASE"]
        C1["📄 Chunk 1: [0.019, ...]\ndistance: 0.04 ✅ close"]
        C2["📄 Chunk 2: [0.812, ...]\ndistance: 0.73 ❌ far"]
        C3["📄 Chunk 3: [0.025, ...]\ndistance: 0.03 ✅ closest"]
        C4["📄 Chunk 4: [0.567, ...]\ndistance: 0.51 ❌ far"]
        C5["📄 Chunk 5: [0.021, ...]\ndistance: 0.05 ✅ close"]
    end

    R["🎯 Top 3 results: Chunk 3, Chunk 1, Chunk 5"]

    Q --> VDB
    C3 --> R
    C1 --> R
    C5 --> R

    style Q fill:#fff3cd,stroke:#ffc107,stroke-width:2px
    style VDB fill:#e2d5f1,stroke:#6f42c1,stroke-width:2px
    style C1 fill:#d4edda,stroke:#28a745
    style C2 fill:#f8d7da,stroke:#dc3545
    style C3 fill:#d4edda,stroke:#28a745,stroke-width:2px
    style C4 fill:#f8d7da,stroke:#dc3545
    style C5 fill:#d4edda,stroke:#28a745
    style R fill:#d4edda,stroke:#28a745,stroke-width:2px
```

### Database Comparison

| Database | Type | Hybrid Search | Metadata Filtering | Scale | FedRAMP | Best For |
|---|---|---|---|---|---|---|
| **Azure AI Search** | Managed | ✅ Vector + keyword | ✅ Rich filters | Billions | ✅ | Microsoft enterprise shops |
| **Pinecone** | Managed | ✅ | ✅ | Billions | ❌ | Startups, fast prototyping |
| **Weaviate** | Open-source | ✅ | ✅ | Millions | Self-host | Flexible, self-hosted |
| **Qdrant** | Open-source | ✅ | ✅ Rich | Billions | Self-host | High performance, self-hosted |
| **ChromaDB** | Open-source | ❌ | ✅ Basic | Thousands | Self-host | Learning, prototyping |
| **pgvector** | Extension | ❌ | ✅ Full SQL | Millions | Depends | Already using PostgreSQL |

> **For regulated enterprise environments:** Azure AI Search if you're in the Microsoft ecosystem (FedRAMP-authorized). Qdrant or Weaviate self-hosted if you need on-premise control. ChromaDB is excellent for learning (see our notebooks) but not for production at scale.

---

## Step 5: Retrieval and Reranking

Retrieval is where the query meets the data. The goal: find the most relevant chunks for the user's question.

### Retrieval Methods

**1. Vector-only (semantic search):**
Convert query to vector, find nearest neighbors. Works well for conceptual questions ("How is employee morale?") but can miss exact matches ("What was the Q3 book-to-bill ratio?").

**2. Keyword-only (BM25/TF-IDF):**
Traditional full-text search. Great for exact terms and technical jargon, but misses semantic similarity.

**3. Hybrid search (vector + keyword):**
Run both searches, combine results. This is the production standard — it catches both conceptual and exact matches.

```mermaid
graph TD
    Q["❓ User query: 'Q3 book-to-bill ratio'"]

    Q --> VS
    Q --> KS

    subgraph VS["🔢 Vector Search"]
        V1["✅ 'New bookings in Q3 totaled $38M'"]
        V2["✅ 'Revenue reached $42.3 million'"]
        V3["✅ 'Backlog stands at $127M'"]
    end

    subgraph KS["🔤 Keyword Search"]
        K1["✅ 'book-to-bill ratio of 0.90'"]
        K2["❌ only exact keyword matches"]
    end

    VS --> H["🎯 Hybrid: combines both\nBest results"]
    KS --> H

    style Q fill:#fff3cd,stroke:#ffc107,stroke-width:2px
    style VS fill:#e2d5f1,stroke:#6f42c1,stroke-width:2px
    style KS fill:#f0f4ff,stroke:#2E86C1,stroke-width:2px
    style H fill:#d4edda,stroke:#28a745,stroke-width:2px
    style V1 fill:#d4edda,stroke:#28a745
    style V2 fill:#d4edda,stroke:#28a745
    style V3 fill:#d4edda,stroke:#28a745
    style K1 fill:#d4edda,stroke:#28a745
    style K2 fill:#f8d7da,stroke:#dc3545
```

### Reranking: The Second Filter

Initial retrieval casts a wide net — top 10-20 results. A **reranker** then scores each result for relevance to the specific question and reorders them.

```mermaid
graph LR
    subgraph BEFORE["🔍 Initial Retrieval — top 5"]
        B1["1. Revenue data — 0.82"]
        B2["2. Headcount data — 0.78"]
        B3["3. Book-to-bill ratio — 0.75"]
        B4["4. Client satisfaction — 0.73"]
        B5["5. Backlog data — 0.71"]
    end

    BEFORE -->|"🎯 Rerank"| AFTER

    subgraph AFTER["✅ After Reranking"]
        A1["1. Book-to-bill ratio — 0.95 ⬆️"]
        A2["2. Revenue data — 0.88"]
        A3["3. Backlog data — 0.82 ⬆️"]
        A4["4. Headcount data — 0.45 ⬇️"]
        A5["5. Client satisfaction — 0.31 ⬇️"]
    end

    style BEFORE fill:#fff3cd,stroke:#ffc107,stroke-width:2px
    style AFTER fill:#d4edda,stroke:#28a745,stroke-width:2px
    style A1 fill:#d4edda,stroke:#28a745
    style A2 fill:#d4edda,stroke:#28a745
    style A3 fill:#d4edda,stroke:#28a745
    style A4 fill:#f8d7da,stroke:#dc3545
    style A5 fill:#f8d7da,stroke:#dc3545
```

**Why reranking matters:** Initial retrieval uses fast but approximate similarity (bi-encoder). Reranking uses slower but more precise cross-attention (cross-encoder) that reads the query and document together. This catches cases where a chunk is similar in topic but not actually relevant to the specific question.

**Common rerankers:** Cohere Rerank, `bge-reranker-v2`, `ms-marco-MiniLM-L-6-v2` (open-source).

---

## Step 6: Prompt Assembly

The final step before the LLM: assembling all retrieved context into a well-structured prompt.

### The Anatomy of a RAG Prompt

```mermaid
graph TD
    subgraph PROMPT["📋 RAG Prompt Assembly"]
        S["🛡️ SYSTEM PROMPT\n'You are a senior analyst. Answer based ONLY on\nthe provided context. Cite sources. If the data\nisn't in the context, say I don't have that.'"]
        C["📁 RETRIEVED CONTEXT\n[Source: Q3 Financial Summary]\nTotal revenue for Q3 reached $42.3 million...\n\n[Source: Q3 Client Report]\nThe firm onboarded 12 new clients in Q3..."]
        U["❓ USER QUESTION\n'How did financial performance relate to new\nclient acquisition in Q3?'"]

        S --> C --> U
    end

    style PROMPT fill:#f0f4ff,stroke:#2E86C1,stroke-width:2px
    style S fill:#f8d7da,stroke:#dc3545
    style C fill:#e2d5f1,stroke:#6f42c1
    style U fill:#fff3cd,stroke:#ffc107
```

### Token Budget Management

The context window has a fixed size. You need to allocate it wisely:

```mermaid
graph TD
    subgraph BUDGET["🧠 Context Window Budget — 128K tokens"]
        A["📋 System prompt\n~500 tokens — fixed"]
        B["📁 Retrieved context\n~2,000 tokens — variable"]
        C["❓ User question\n~100 tokens — small"]
        D["🤖 Response space\n~1,000 tokens — reserved"]
        E["✅ Available for context\n~124,400 tokens — plenty of room"]

        A --> B --> C --> D --> E
    end

    style BUDGET fill:#f0f4ff,stroke:#2E86C1,stroke-width:2px
    style A fill:#f8d7da,stroke:#dc3545
    style B fill:#e2d5f1,stroke:#6f42c1
    style C fill:#fff3cd,stroke:#ffc107
    style D fill:#d4edda,stroke:#28a745
    style E fill:#d4edda,stroke:#28a745,stroke-width:2px
```

At 128K tokens, budget is rarely an issue. But at 4K-8K tokens (older models, cheaper tiers), you must be strategic: fewer chunks, shorter system prompts, compressed context.

---

## Evaluating RAG Quality

Building the pipeline is half the work. Measuring whether it actually works well is the other half.

### The Three Dimensions of RAG Quality

| Dimension | Question It Answers | How to Measure |
|---|---|---|
| **Retrieval quality** | Did we find the right chunks? | Recall@K, precision@K, MRR |
| **Faithfulness** | Does the answer match the retrieved context? | LLM-as-judge, NLI models |
| **Answer correctness** | Is the final answer actually right? | Human evaluation, ground truth comparison |

### Common Failure Modes

1. **Retrieval miss:** The right information exists in your data but the retrieval pipeline doesn't find it. Fix: better chunking, hybrid search, reranking.
2. **Hallucination despite context:** The LLM ignores the retrieved data and makes something up. Fix: stronger system prompt guardrails, lower temperature.
3. **Wrong context, right format:** Retrieval returns plausible-looking but wrong chunks, and the LLM confidently answers from them. Fix: reranking, metadata filtering.
4. **Boundary split:** The answer spans two chunks that got split apart during chunking. Fix: overlapping chunks, hierarchical chunking.

---

## Decision Framework: Should You Build RAG?

```mermaid
graph TD
    Q1{"🤔 Need AI to answer questions\nabout private/current data?"}
    Q1 -->|No| A1["✅ Standard LLM with\ngood prompting is sufficient"]
    Q1 -->|Yes| Q2{"📏 Data < 100 docs\nand < 100K tokens?"}

    Q2 -->|Yes| A2["✅ Consider long context\nfirst — simpler"]
    Q2 -->|No| A3["🎯 RAG is the right choice"]

    Q1 -->|Yes| Q3{"🔄 Data changes\nfrequently — daily/weekly?"}
    Q3 -->|Yes| A4["🎯 RAG — update index\nincrementally"]

    Q1 -->|Yes| Q4{"⚡ Need sub-second\nresponse times?"}
    Q4 -->|Yes| A5["🎯 RAG — long context\n= 30–60s at 1M tokens"]

    Q1 -->|Yes| Q5{"💾 Data measured\nin terabytes?"}
    Q5 -->|Yes| A6["🎯 RAG is the\nonly viable option"]

    style Q1 fill:#fff3cd,stroke:#ffc107,stroke-width:2px
    style Q2 fill:#fff3cd,stroke:#ffc107
    style Q3 fill:#fff3cd,stroke:#ffc107
    style Q4 fill:#fff3cd,stroke:#ffc107
    style Q5 fill:#fff3cd,stroke:#ffc107
    style A1 fill:#f0f4ff,stroke:#2E86C1
    style A2 fill:#f0f4ff,stroke:#2E86C1
    style A3 fill:#d4edda,stroke:#28a745
    style A4 fill:#d4edda,stroke:#28a745
    style A5 fill:#d4edda,stroke:#28a745
    style A6 fill:#d4edda,stroke:#28a745
```

---

## Key Takeaways

1. **RAG has two paths — indexing and querying.** The indexing path (chunk, embed, store) runs offline. The query path (embed question, search, rerank, generate) runs in real-time. Most engineering effort goes into the indexing path.

2. **Chunking strategy is the #1 lever.** Bad chunking produces bad embeddings, which produce bad retrieval, which produces bad answers. Start with semantic chunking + overlap for most use cases.

3. **Hybrid search (vector + keyword) is the production standard.** Vector-only misses exact terms. Keyword-only misses semantic meaning. Hybrid catches both.

4. **Reranking is cheap insurance.** A cross-encoder reranker after initial retrieval catches false positives and dramatically improves precision. Always add one in production.

5. **Measure what matters.** Retrieval quality, faithfulness, and answer correctness are three separate dimensions. A system can have perfect retrieval but still hallucinate, or perfect generation but retrieve the wrong chunks.

---

### Related Content
- **[RAG vs Long Context](05-rag-vs-long-context.md)** — When to use RAG, when long context wins
- **[Notebook 01 — Semantic Search Visualized](../notebooks/01-semantic-search-visualized.ipynb)** — See how embeddings and similarity work
- **[Notebook 02 — Build a RAG Pipeline](../notebooks/02-build-a-rag-pipeline.ipynb)** — Build a working RAG system step by step
- **[Notebook 03 — Chunking Strategies Compared](../notebooks/03-chunking-strategies.ipynb)** — Hands-on comparison of chunking approaches
