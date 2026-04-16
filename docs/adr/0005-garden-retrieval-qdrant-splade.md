# 0005 — Garden Retrieval: Qdrant + SPLADE + Quarkus Native

Date: 2026-04-16
Status: Proposed

## Context and Problem Statement

The current retrieval algorithm (Python, 3-tier: domain filter → INDEX.md match → git grep)
is a hand-rolled approximation of full-text search with no relevance ranking and no semantic
understanding. It does not scale to community-garden volumes (100K–1M entries). A
replacement is needed that scales without architectural changes, supports two consumer modes
(LLM downstream and human downstream), and operates within a federated multi-garden model
where each garden runs independently.

## Decision Drivers

* Scales identically from 100 to 1,000,000 entries — no migration path needed
* Two retrieval modes: recall (LLM reads all results) and precision (human sees top result)
* Federated: each garden is an independent stack; child gardens query parent gardens via API
* Open-source and self-hosted — no managed cloud services
* Java/Quarkus preferred — team has LangChain4J experience
* Distributable as a native binary for developer laptop deployment

## Considered Options

* **Option A** — Qdrant + SPLADE (ONNX) + Ollama + Quarkus native (chosen)
* **Option B** — Meilisearch + Ollama + Quarkus
* **Option C** — Custom BM25 (Java) + Qdrant + Ollama + Quarkus
* **Option D** — SQLite FTS5 (no vector search)
* **Option E** — DJL local inference only (no Ollama)

## Decision Outcome

Chosen option: **Option A — Qdrant + SPLADE + Ollama + Quarkus native**.

Each garden deploys an independent stack: git repo (source of truth) + Qdrant instance
(vector index) + `garden-retrieval` Quarkus native service (REST API + MCP server).

Sparse vectors generated client-side via SPLADE (`prithivida/Splade_PP_en_v1`, Apache 2.0)
using ONNX Runtime Java binding. Dense vectors via Ollama (`nomic-embed-text`). Qdrant
stores both vector types and handles hybrid search (Reciprocal Rank Fusion), payload
filtering, and IDF weighting via `"modifier": "idf"`.

Two search modes:
- **Recall:** 5–8 full entries, no reranker (LLM downstream reranks)
- **Precision:** 1–3 card-format entries, ONNX cross-encoder reranker (human downstream)

### Positive Consequences

* SPLADE: no corpus statistics in Java — scales to 1M entries with zero code changes
* SPLADE semantic term expansion: "flush" retrieves "lifecycle", "callback", "JPA" — beyond exact BM25 matching
* Qdrant HNSW: O(log n) query latency regardless of collection size
* Per-garden Qdrant: federation is service-to-service HTTP, not git-repo cloning
* Native image: fast startup, low memory, distributable binary
* LLM reranker (recall mode): better than any ONNX model at selecting the relevant entry

### Negative Consequences / Tradeoffs

* Qdrant server-side inference is Cloud-only: Java client must compute vectors client-side
* SPLADE ONNX + Quarkus native image: JNI complications on some platforms (needs validation)
* LangChain4J Qdrant integration does not expose sparse vectors (Issue #1600): requires
  direct Qdrant Java client alongside LangChain4J for sparse vector upsert
* Ollama required as a separate process for dense embeddings (not a pure Java stack)
* Qdrant restart at 1M entries takes 3–8 minutes to deserialise HNSW from disk

## Pros and Cons of the Options

### Option A — Qdrant + SPLADE + Ollama + Quarkus

* ✅ SPLADE scales to 1M+ without corpus statistics or re-indexing
* ✅ Hybrid search (dense + sparse) catches both exact terms and semantic similarity
* ✅ ONNX reranker precision mode — local, no API
* ✅ Per-garden Qdrant enables clean federated service model
* ❌ ONNX Runtime JNI in native image needs platform validation
* ❌ LangChain4J sparse vector gap requires dual-client workaround
* ❌ Ollama is a separate process

### Option B — Meilisearch + Ollama + Quarkus

* ✅ Simpler hybrid search setup (BM25 + vector built in)
* ✅ Good developer experience
* ❌ BM25 only (no learned sparse) — weaker semantic recall than SPLADE
* ❌ Separate search service alongside the Java service
* ❌ IDF corpus statistics recalculated on change (scaling concern)

### Option C — Custom BM25 (Java) + Qdrant + Ollama + Quarkus

* ✅ No ONNX Runtime for sparse vectors
* ✅ Simple implementation
* ❌ Corpus-wide IDF statistics must be maintained in Java (scaling problem at 1M entries)
* ❌ Exact term matching only — no semantic expansion
* ❌ Requires periodic re-indexing as corpus grows

### Option D — SQLite FTS5

* ✅ No external services for search
* ✅ stdlib only
* ❌ No semantic search — pure keyword
* ❌ Degrades significantly at 100K+ entries
* ❌ No vector search capability

### Option E — DJL local inference only (no Ollama)

* ✅ Truly self-contained — no external embedding service
* ❌ CPU-only inference for dense embeddings (no GPU acceleration)
* ❌ Bulk import of 1M entries: hours vs minutes with GPU
* ❌ More complex native image (ONNX Runtime JNI for both dense and sparse)

## Links

* Design spec: `spec/docs/design/2026-04-16-garden-retrieval-service.md`
* Review brief: `spec/docs/design/2026-04-16-garden-retrieval-review-brief.md`
* Supersedes: hand-rolled 3-tier Python retrieval in `soredium/scripts/mcp_garden_search.py`
* Relates to: ADR-0004 (SQLite aggregate state — CHECKED.md replacement)
* Critical validations required before implementation — see review brief
