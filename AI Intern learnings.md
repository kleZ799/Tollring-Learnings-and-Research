# RAG Learning Journey — My Notes

## Milestone 1: RAG Foundation

### What I Built
I built a basic end-to-end RAG system: I take raw documents, preprocess and chunk them, embed those chunks into vectors, store them in a vector database, retrieve the most relevant chunks for a user question, and then let an LLM generate an answer grounded in those retrieved chunks. In practice this means: I can drop in PDFs/Markdown/text, the pipeline chunks and embeds them, and then at query time I get back both an answer and the supporting passages.

### Core Concepts I Learned

#### RAG architecture (indexing → retrieval → generation)
- **What it is (in my words)**: RAG is a pipeline, not just a single LLM call. First I **index** documents (clean them, chunk them, embed them, and store them). Then, when a question comes in, I **retrieve** the most relevant chunks from that index. Finally, I pass those chunks plus the question into the LLM to **generate** an answer.
- **Why it matters**: It forces me to separate "knowledge storage and search" from "language generation." Instead of hoping the model magically knows everything, I explicitly control what information it can look at.
- **What problem it solves**: A pure LLM tends to hallucinate or miss recent/secret information. RAG lets me plug the model into my own data and keep the knowledge base up-to-date without retraining the model.

#### Chunking strategies (fixed, recursive, semantic) and the tradeoffs between them
- **What it is**:
  - **Fixed chunking**: split text into equal-sized pieces (e.g. 512 tokens) with optional overlap.
  - **Recursive chunking**: split text by structure (headings → subheadings → paragraphs → sentences) until chunks fit a size limit.
  - **Semantic chunking**: split based on meaning or topic boundaries (e.g., using embeddings or a model to find natural breakpoints).
- **Why it matters**: The quality of retrieved context is heavily determined by how I chunk. Bad chunking means the right facts are either missing, buried with irrelevant text, or spread across chunks.
- **What problem it solves**:
  - Fixed chunking is simple and fast, but can cut important sections in half and mix unrelated topics.
  - Recursive chunking preserves document structure, so a "section" stays together where possible.
  - Semantic chunking tries to keep each chunk about one coherent idea, which often makes retrieval and reading easier for the model.
  - The tradeoff is simplicity vs. quality vs. compute cost. Fixed is easiest but crude; semantic is smartest but more expensive and complex.

#### Embeddings and vector similarity (what they are, how cosine similarity works)
- **What it is**: An **embedding** is a numerical representation (vector) of a piece of text, where similar meanings end up as points that are close together in a high-dimensional space. **Cosine similarity** measures how similar two vectors are by the angle between them: 1 means they point in the same direction (very similar), 0 means orthogonal (unrelated), -1 means opposite.
- **Why it matters**: Instead of matching exact words, I can do **semantic search**. Queries like "How do I terminate my agreement?" can match docs talking about "cancelling a contract" even if the wording is different.
- **What problem it solves**: Exact keyword search fails when the question and documents use different phrasing. Embeddings plus cosine similarity bridge that gap by ranking text by meaning rather than literal string match.

#### Vector databases (why they're different from normal databases)
- **What it is**: A vector database is designed to store high-dimensional vectors and perform fast approximate nearest-neighbor (ANN) search over them. Instead of indexing columns and rows, it organizes vectors so we can quickly find the closest ones to a query vector.
- **Why it matters**: For RAG, every chunk becomes a vector. I need to search across potentially millions of these vectors in real-time. Traditional relational databases aren’t optimized for this kind of similarity search.
- **What problem it solves**: It makes semantic search at scale practical. Without a vector DB (or a library with similar indexing), retrieval would be too slow or too memory-heavy when the corpus gets large.

#### Top-k vs MMR retrieval (why "most similar" isn't always "most useful")
- **What it is**:
  - **Top-k retrieval**: take the k chunks with the highest similarity scores to the query.
  - **MMR (Maximal Marginal Relevance)**: balances **relevance** to the query with **diversity** among the retrieved chunks, penalizing near-duplicates.
- **Why it matters**: If I only use top-k similarity, I often get several very similar chunks saying almost the same thing (e.g., the same section repeated or multiple paragraphs from one narrow part of a document).
- **What problem it solves**: MMR reduces redundancy. It tries to cover multiple aspects of a question by picking chunks that are each relevant but not too similar to one another. This helps the LLM see a broader slice of the knowledge base instead of reading the same thing five times.

#### Why grounding the LLM in retrieved context prevents hallucination
- **What it is**: "Grounding" means giving the LLM concrete retrieved passages to base its answer on and, ideally, asking it to quote or reference them explicitly.
- **Why it matters**: LLMs are very good at producing fluent text, even when it’s wrong. By grounding, I constrain the model to answer *from* the provided context instead of free-associating from its pretraining.
- **What problem it solves**: It reduces hallucinations and makes answers more auditable. If the model says something, I can check whether that statement is actually supported by the retrieved chunks. If not, that’s a signal to improve retrieval instead of blaming "the model" in the abstract.

### Key Takeaway
Milestone 1 taught me that RAG is mostly about **good retrieval**, not just fancy prompting. The model can only be as trustworthy as the chunks I feed into it.

---

## Milestone 2: Advanced RAG Optimization

### What I Built
I extended the basic RAG pipeline into a more advanced, production-style system. This version adds **query expansion**, **hybrid search** (BM25 + vector search with score fusion), a **reranking** step using a stronger model, and automatic **RAGAS evaluation** of the answers. Instead of just "retrieve some chunks and hope," I now have a pipeline that actively boosts recall, improves ranking quality, and measures how well the system is doing.

### Core Concepts I Learned

#### Query expansion (why one phrasing of a question isn't enough)
- **What it is**: Taking the user’s question and generating alternative versions or related queries (synonyms, sub-questions, reformulations) to use during retrieval.
- **Why it matters**: Users often write short, vague, or oddly-phrased questions. A single query can easily miss relevant documents that use different terminology or focus on one sub-aspect of the question.
- **What problem it solves**: Query expansion improves **recall**. For example, a user might ask "How do I get a refund?" while the docs use "reimbursement policy" and "chargeback." By expanding the query with these related phrases, I give the retrieval system more chances to pull the right chunks.

#### Hybrid search (BM25 keyword search + vector search, and why combining them matters — score fusion and normalization)
- **What it is**: **Hybrid search** combines traditional keyword-based search (e.g., BM25) with vector-based semantic search. Each side produces its own ranking and scores, which I then normalize and **fuse** (for example, a weighted sum) into a single ranked list.
- **Why it matters**:
  - BM25 is great at exact term matching and handling term frequency and document length.
  - Vector search is great at semantic similarity even when wording differs.
  - Each has blind spots: BM25 is "exact-term obsessed" and can miss synonyms; pure vector search can over-rank passages that are semantically close but miss a critical keyword (like a product name or version number).
- **What problem it solves**: By normalizing both score types and combining them, hybrid search covers both **exact matches** and **semantic matches**. For example, if a user includes a specific error code, BM25 makes sure chunks containing that exact code rank highly, while vector search still surfaces semantically related explanations.

#### Reranking (bi-encoder vs cross-encoder — the actual architectural difference, and why we use both in a two-stage retrieve-then-rerank pattern instead of just one)
- **What it is**:
  - A **bi-encoder** encodes queries and documents **separately** into vectors. Similarity is computed with a fast function (e.g., dot product or cosine). This is what I use for the initial retrieval because it’s efficient.
  - A **cross-encoder** takes the query and a candidate document chunk **together** as input and runs them through a model that attends across both sequences, outputting a relevance score.
- **Why it matters**: Bi-encoders scale well (I can precompute document embeddings and index them), but their scoring is relatively coarse. Cross-encoders are much more expressive because they see the **full interaction** between query and text, but they are much slower and can’t be precomputed the same way.
- **What problem it solves**: The two-stage pattern — **retrieve (bi-encoder) → rerank (cross-encoder)** — lets me get the best of both worlds. I first use the bi-encoder to quickly narrow down to, say, the top 50 candidates. Then I pass those to the cross-encoder, which re-scores them with a deeper understanding and returns a better-ordered top N. This fixes cases where the initial retrieval got "the right neighborhood" but mis-ordered the best chunks.

#### RAGAS evaluation (the four metrics — faithfulness, answer relevance, context precision, context recall — and why you need all four together, not just one aggregate score)
- **What it is**: RAGAS is a framework for evaluating RAG systems using four main metrics:
  - **Faithfulness**: Does the answer stay true to the provided context, or is it hallucinating?
  - **Answer relevance**: Does the answer actually address the user’s question?
  - **Context precision**: Are the retrieved chunks mostly relevant, or is there a lot of noise?
  - **Context recall**: Did we retrieve the key pieces of information needed to answer the question?
- **Why it matters**: A single "overall score" hides what’s really going wrong. RAG systems can fail in different ways, and I need to know **which part** (retrieval vs. generation, precision vs. recall, etc.) to fix.
- **What problem it solves**:
  - Low **faithfulness** means the LLM is inventing details not supported by the context → fix prompting or add stricter grounding.
  - Low **answer relevance** means the model isn’t really answering the question → fix instruction design or retrieval focus.
  - Low **context precision** means I’m pulling in too much irrelevant text → refine chunking, filtering, or ranking.
  - Low **context recall** means key facts never even make it into the context window → improve retrieval, query expansion, or indexing.
  Using all four together turns "the RAG system feels off" into a concrete debugging checklist.

### Key Takeaway
Milestone 2 taught me to think of RAG as an **optimizable system**, not a black box. With techniques like hybrid search, reranking, and RAGAS, I can systematically diagnose where it fails and make targeted improvements.

---

## The Meta-Lesson

Across both milestones, I see a clear pattern: every "advanced" technique is really a response to a specific failure of a simpler approach.
- **Fixed chunking** often cuts ideas in half or mixes topics → **recursive/semantic chunking** preserve structure and meaning.
- Plain **top-k retrieval** returns redundant chunks → **MMR** enforces diversity so more aspects of the question are covered.
- Pure **vector search** can ignore critical exact terms → **hybrid search** with BM25 fixes "exact-term blindness."
- A single-pass **bi-encoder retrieval** can mis-rank borderline cases → adding a **cross-encoder reranker** fixes imprecise ranking.
- **Manual spot-checking** of answers is subjective and doesn’t scale → **RAGAS** gives me metric-based, automated feedback.

The transferable skill here is not memorizing each individual trick, but recognizing the **failure mode → targeted fix** pattern. When something goes wrong (missing facts, noisy context, hallucinations, wrong ranking), I now think: "Which stage of the pipeline is failing, and what specific technique exists to address exactly this kind of problem?"

---

## What's Next

Next up is **Milestone 3: Knowledge Graphs & Entity Extraction with Neo4j**. I’ll be focusing on pulling structured entities and relationships out of unstructured text and storing them in a graph so I can answer more complex, multi-hop questions. As I work through that milestone, I’ll come back and extend these notes with what I build and what I learn.