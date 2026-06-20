<div align="center">

# 📝 AI RAG Research Assistant — Case Study

### What I built, the problems I hit, what I learned, and where it goes next

![Status](https://img.shields.io/badge/status-shipped-60f3a9)
![Domain](https://img.shields.io/badge/domain-RAG_·_LLM-9b5cff)
![Read_time](https://img.shields.io/badge/read-~6_min-65b8ff)

📦 **App repo:** [AI-RAG-Research-Assistant](https://github.com/Swarali603/AI-RAG-Research-Assistant) · 🧠 **System design:** [AI-RAG-System-Design](https://github.com/Swarali603/AI-RAG-System-Design)

</div>

---

## 🎯 The goal

Reading long PDFs is slow, keyword search misses context, and generic chatbots hallucinate without showing sources. I wanted to build an assistant that **answers questions from your own documents** and is **transparent about how** — every answer backed by citations, retrieval scores, and the exact prompt the model saw.

---

## 🛠️ What I built

A complete **Retrieval-Augmented Generation (RAG)** pipeline as a single, deploy-anywhere FastAPI service:

- **PDF ingestion** — upload one or more PDFs; text is extracted page-by-page and split into overlapping word-window chunks.
- **Hybrid retrieval** — relevant chunks are found by combining **semantic embeddings** (Gemini `text-embedding-004`) with **BM25 lexical scoring**, normalized and blended. Falls back to BM25-only if embeddings are unavailable, so it never hard-fails.
- **Grounded generation** — retrieved chunks are injected into a prompt and answered by `gemini-2.5-flash-lite`, **streamed token-by-token** over Server-Sent Events.
- **Cited answers** — every response references the source file, page, and chunk it drew from.
- **RAG Inspector** — a UI panel that shows query terms, retrieved chunks with similarity scores, matched terms, and the full prompt — so the retrieval is never a black box.
- **Two answer modes** — a professional *Research Mode* and a casual *Bestie Mode*.
- **Document caching** — a PDF is parsed and embedded **once per session** (keyed by content hash), not on every question.

**Stack:** Python · FastAPI · Google Gemini · pypdf · custom BM25 · pytest · Vercel.

---

## 🧱 Problems I faced (and how I solved them)

### 1. Retrieval was lexical-only — RAG that couldn't find synonyms
My first version scored chunks with cosine similarity over raw **keyword counts**. That meant a question about *"downsides"* wouldn't match a document that said *"limitations"* — the exact failure RAG is supposed to fix.
> **Fix:** moved to **hybrid retrieval** — real semantic embeddings for meaning, BM25 for exact terms and names, blended together. Pulled the ranking math into a pure, testable module.

### 2. Serverless killed my "obvious" tech choices
The textbook RAG stack is `sentence-transformers` + `FAISS`. On Vercel's serverless runtime that's a non-starter — huge model downloads and native build steps that don't fit the environment.
> **Fix:** used the **Gemini embedding API** instead of local models. No heavy dependencies, and it runs in a serverless function.

### 3. Every question re-processed the entire PDF
Each request re-extracted, re-chunked, and re-scored the whole document. Ask five questions about a 100-page PDF and it did the work five times.
> **Fix:** added an in-process **cache keyed by a hash of the file bytes**, so parsing and embedding happen once. FIFO-bounded so memory can't grow forever.

### 4. The repo drifted from the running app
The README described a Streamlit + FAISS app, but what actually shipped was FastAPI with different modes and retrieval. I also ended up with **two parallel implementations** living in the same repo, which confused anyone reading it.
> **Fix:** rewrote the docs to match reality and made the deployed app the single source of truth. (Lesson: docs are part of the product.)

### 5. "Tests" that couldn't fail
My early test files were `print()` scripts — they ran code but asserted nothing, so they could never actually catch a regression.
> **Fix:** wrote a real **pytest** suite for the retrieval logic and app helpers (runs fully offline, no API key), and wired up **GitHub Actions CI** to run it on every push.

### 6. Over-permissive defaults
CORS was wide open (`*` with credentials — which is actually invalid per spec), and uploads had no size limit.
> **Fix:** made CORS origins configurable, fixed the credentials/wildcard conflict, and added a per-file upload cap.

---

## 💡 Lessons learned

- **"RAG" is only as good as retrieval.** The LLM is the easy part; getting the *right* chunks in front of it is where the quality lives. Hybrid retrieval was the single biggest jump in answer quality.
- **Design for the deployment target early.** Half my rework came from assuming a stack that the serverless environment couldn't run. Knowing the constraints up front would have saved a rewrite.
- **Graceful degradation beats cleverness.** Making the embedder return `None` and fall back to BM25 meant a bad API key or rate limit produces a worse answer, not a crash.
- **Transparency is a feature.** The RAG Inspector started as a debugging tool and became the most compelling part of the demo — people trust answers they can audit.
- **Caching is a design decision, not an afterthought.** A content-hash cache turned a sluggish multi-question session into a fast one with a few lines of code.
- **A test that can't fail is documentation, not a test.** Real assertions + CI changed how confidently I could refactor.

---

## 🔭 Future improvements

| Area | Idea | Why it matters |
| --- | --- | --- |
| **Persistence** | Store embeddings in a vector DB (pgvector / Pinecone) keyed by document | Survive across sessions and cold starts; share docs between users |
| **Smarter chunking** | Semantic / heading-aware splitting instead of fixed windows | Cleaner context boundaries → better retrieval and citations |
| **Reranking** | Add a cross-encoder or LLM reranker over the top-k | Squeeze more precision out of retrieval before generation |
| **Evaluation** | A small labeled Q&A set + retrieval metrics (hit-rate, MRR) | Measure quality objectively instead of by vibe |
| **Multi-format** | Support DOCX, web pages, and images (OCR) | Broaden beyond PDFs |
| **Conversation memory** | Persist chat history and resolve follow-up references server-side | True multi-turn research sessions |
| **Observability** | Log latency, token usage, and fallback rate | Know what it costs and where it's slow |

---

<div align="center">

*Built as a learning project to go deep on retrieval, grounding, and shipping an LLM app end-to-end.*

</div>
