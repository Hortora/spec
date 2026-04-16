# Garden Retrieval Service — Review Brief

A self-contained summary for independent architectural review. Seeking alternatives,
challenges, and gaps we may have missed.

---

## The Problem

**Two distinct gardens, same infrastructure:**

**Garden 1 — Knowledge garden:** Short markdown entries (~50–200 lines) capturing
non-obvious developer knowledge — gotchas, techniques, undocumented behaviours. Editorial
bar: only what would genuinely surprise a skilled developer. Prose-centric.

**Garden 2 — Code examples garden:** Minimal, use-case-driven code snippets (~20–50
lines). Entry point: "I want to support multiple databases for my persistence layer" →
return 3–5 working minimal examples to copy. Code-centric. No editorial bar — any valid
working minimal example qualifies. Populated primarily by a **code miner** (see below).

**Code Miner:** A CLI tool that points at a codebase, uses JavaParser (AST analysis) to
discover pattern instances (builders, multi-datasource configs, test patterns, etc.), then
uses an LLM (Claude API or Ollama) to rewrite each instance as a minimal teachable
example. Optional human review gate before submission. Handles duplicate detection by
querying the retrieval API before submitting.

Both gardens use the same Qdrant + `garden-retrieval` Quarkus service infrastructure —
different Qdrant collections, potentially different embedding models.

---

## The Retrieval Problem

**Knowledge garden:** given a natural language symptom description ("my Hibernate
callback isn't firing when I expect it to"), find the 3–8 most relevant entries. Two
consumer modes:

- **LLM downstream (recall):** An AI assistant (Claude, Copilot, Cursor) reads all
  returned entries and selects the relevant one. Return 5–8 full entries. Optimise for
  recall — the right entry must be in the set. No reranker needed; the LLM reranks.

- **Human downstream (precision):** A developer is using a web UI or CLI. Return 1–3
  entries with the best result at position 1. Optimise for precision. A reranker is needed
  because there's no LLM downstream.

Scale target: grows from hundreds to potentially 1 million entries. Must work identically
at every scale with no architectural changes.

---

## The Stack

**Language / framework:** Quarkus native image (GraalVM). Deployed as a distributable
binary. Fast startup (<500ms), low memory (~80–150MB), runs as a developer laptop sidecar
or server service.

**Vector database:** Qdrant (open-source, self-hosted). Handles hybrid search (dense +
sparse vectors), metadata payload filtering, HNSW index, Reciprocal Rank Fusion. Each
garden runs its own Qdrant instance.

**Dense embeddings:** Ollama running `nomic-embed-text` (768-dim, Apache 2.0) locally.
GPU-capable when available. The Java service calls Ollama's HTTP API — no embedding code
in the Java service.

**Sparse embeddings:** `prithivida/Splade_PP_en_v1` (Apache 2.0 SPLADE++ model) run
via ONNX Runtime Java binding (`com.microsoft.onnxruntime`, MIT). Computed client-side
in the Java service, stored in Qdrant as sparse vectors. Qdrant applies IDF weighting at
query time via `"modifier": "idf"` on the collection — no corpus statistics maintained
in Java.

**Reranker (precision mode only):** ONNX cross-encoder (`cross-encoder/ms-marco-MiniLM-L6-v2`)
via LangChain4J's ONNX reranking pipeline. Local, no API key.

**RAG framework:** LangChain4J (Quarkus extension). Provides: Qdrant integration, ONNX
reranker pipeline, MCP server declarative API. Justification: team has LangChain4J
experience; Quarkus Dev Services provides zero-config local Qdrant + Ollama for development.

**MCP server:** Exposed via Quarkus LangChain4J MCP extension. Stdio transport for
Claude Code / Cursor / Copilot. Both `garden_search` (recall mode) and `garden_status`
tools. Any MCP-compatible AI assistant can query the garden directly.

---

## What the Java Service Does

Almost nothing — the intelligence is entirely in Qdrant and the models:

1. **Parse YAML frontmatter** (~30 lines, SnakeYAML). Extract title, domain, type, score,
   tags, stack, submitted, body.

2. **Index** (on new entry merged):
   - Call Ollama → dense vector (768-dim)
   - Run SPLADE ONNX model → sparse vector (term → weight map)
   - Qdrant upsert: dense + sparse + metadata payload

3. **Search** (REST or MCP):
   - Qdrant hybrid query: dense prefetch + sparse prefetch + RRF fusion
   - Pre-filter by metadata: domain, type, score ≥ N
   - Recall mode: return 5–8 full entries
   - Precision mode: rerank top 20 via ONNX cross-encoder, return top 3 as cards

4. **Expose MCP server** for AI assistant integration.

---

## Federation Model

Each garden is an independent stack: git repo + Qdrant + garden-retrieval service.

```
garden/
  ├── git repo          ← entries, SCHEMA.md (source of truth)
  ├── Qdrant            ← vector index (this garden's entries only)
  └── garden-retrieval  ← REST API + MCP server
```

Garden relationships (declared in SCHEMA.md):
- **Canonical** — root authority, no upstream
- **Child** — extends a parent; strict non-duplication contract; retrieval walks upstream
- **Peer** — equals sharing knowledge without hierarchy

Client config (`~/.claude/garden-config.toml`):
```toml
[[gardens]]
name = "jvm-garden"
path = "~/.hortora/jvm-garden"
retrieval_url = "https://api.hortora.io/jvm"

[[gardens]]
name = "my-private-garden"
path = "~/work/my-garden"
retrieval_url = "http://localhost:8090"
```

**Upstream chain walk:** child's retrieval service searches its local Qdrant first; on
upstream query, HTTP call to parent's `/api/search`. Semantic SPLADE search at the parent —
no local clone of parent entries needed.

**Non-duplication on submission:** when a PR arrives at a child garden, call parent's
retrieval API with the new entry's text. SPLADE similarity catches semantic duplicates
that Jaccard title/tag matching misses.

`init_garden.py` generates a `docker-compose.yml` for the full local stack (Qdrant +
garden-retrieval service).

---

## Key Decisions and Rationale

| Decision | Rationale | Alternative considered |
|----------|-----------|----------------------|
| Qdrant over Meilisearch | Qdrant/bm25 + SPLADE in one store; ACORN filtered HNSW | Meilisearch — hybrid search but no SPLADE, would still need ONNX client-side |
| SPLADE over custom BM25 | No corpus statistics; scales to 1M+ with zero changes; semantic term expansion | Custom 50-line BM25 — simpler but requires IDF management, degrades at scale |
| Ollama for dense embeddings | GPU-capable, simple HTTP API, widely familiar | DJL local ONNX (pure Java) — eliminates Ollama but CPU-only, complex native image |
| No LLM reranker in recall mode | LLM downstream is a better reranker than any ONNX model | ONNX reranker in both modes — adds latency, worse than the LLM that reads anyway |
| LangChain4J over raw clients | MCP server + ONNX reranker pipeline; team experience | Pure Qdrant Java client + manual MCP implementation — more code |
| Service per garden | Clean isolation; parent serves upstream queries via API | Shared Qdrant with namespacing — coupling, harder to run canonical garden as public service |
| Quarkus native image | Fast startup for developer sidecar; distributable binary | JVM jar — simpler build, but slower startup, larger footprint |

---

## Performance Estimates (beefy server: 64-core, 256GB RAM, A100 GPU)

| Metric | Estimate |
|--------|---------|
| 1000 entries bulk import | 10–30 seconds (GPU-batched SPLADE + dense) |
| Restart, 1M entries, indexes preserved | 3–8 minutes (Qdrant loads from disk) |
| Restart, 1M entries, from Qdrant snapshot | 10–20 minutes |
| Restart, 1M entries, full rebuild | 1–2 hours (re-embed + HNSW rebuild) |
| Search latency, 1M entries, recall mode | 15–40ms |
| Search latency, 1M entries, precision mode | 70–150ms |

---

## What Needs Validation Before Building

**Critical (blockers if wrong):**

1. **SPLADE ONNX in Quarkus native image.** ONNX Runtime uses JNI. Quarkus native image
   + JNI has known complications on macOS ARM. If SPLADE can't run in native image, the
   "distributable binary with no external dependencies" goal fails for sparse embeddings.
   The fallback is: SPLADE runs in JVM mode only, native image skips it and falls back
   to TF-only sparse vectors (degraded quality). *Needs: test on target platforms.*

2. **LangChain4J Qdrant + sparse vectors.** LangChain4J's Qdrant integration does not
   natively expose sparse vector upsert (Issue #1600 open). The plan uses the Qdrant Java
   client directly for sparse vectors alongside LangChain4J for dense. Verify this
   dual-client pattern works without state conflicts. *Needs: prototype.*

3. **Qdrant `"modifier": "idf"` + SPLADE sparse vectors.** The IDF modifier is designed
   for BM25 TF vectors. Using it with SPLADE output (which already incorporates semantic
   weighting) may double-weight IDF or interact unexpectedly. Verify that the combination
   is semantically correct, or disable the IDF modifier when using SPLADE.
   *Needs: review Qdrant documentation + test retrieval quality.*

**Important (affect quality but not blocking):**

4. **Embedding model selection.** `nomic-embed-text` (768-dim) is the assumed dense model.
   For technical developer content (short entries, specific library names, error messages),
   a code-specific model (CodeBERT, `Salesforce/SFR-Embedding-Code`) may outperform a
   general-purpose sentence transformer. *Needs: quality benchmark on garden entries.*

5. **SPLADE vs BM25 quality on short technical entries.** SPLADE is superior on
   long documents. For 50–200 line entries with highly specific technical vocabulary
   (library names, error codes), BM25 exact matching may be competitive or superior.
   *Needs: A/B test on garden corpus.*

6. **Reranker in Quarkus native image.** Same JNI concern as SPLADE but specifically
   for the cross-encoder reranker in precision mode. LangChain4J's ONNX reranker support
   in native image needs verification. *Needs: test on target platforms.*

7. **Performance numbers.** All figures above are component-based estimates, not benchmarks
   of this specific stack. Actual throughput depends on Ollama/SPLADE batch efficiency,
   Qdrant version, hardware. *Needs: benchmark on target infrastructure.*

---

## Open Questions

1. **Flat folder vs domain directories.** The retrieval service is indifferent — domain
   is a Qdrant payload field, not a directory. But domain directories improve GitHub
   browsability and CI routing. Not resolved.

2. **FastEmbed4J.** A Java port of Qdrant's FastEmbed Python library (ONNX Runtime +
   HuggingFace Tokenizers + BM25/SPLADE) would eliminate the need for Ollama entirely,
   giving a truly self-contained native binary. Does not exist yet. Worth building as
   a separate open-source library or raising with Qdrant.

3. **Qdrant snapshot strategy.** At 1M entries, catastrophic rebuild is 1–2 hours.
   Qdrant has built-in snapshot/restore (minutes). Snapshot schedule and retention
   policy not yet designed.

4. **Peer garden search.** Current design handles canonical/child chain. Peer gardens
   (parallel canonical gardens, independent curation) — how does a child garden query
   across multiple peer canonicals? Not yet designed.

5. **garden-retrieval as an MCP proxy.** The child garden's retrieval service could
   transparently proxy MCP calls to the parent's retrieval service, making federation
   invisible to the end user. Not yet designed.

6. **Embedding model for code examples garden.** `nomic-embed-text` embeds NL use-case
   descriptions against NL queries — probably adequate. A code-specific model
   (`jinaai/jina-embeddings-v2-base-code`, Apache 2.0) may improve retrieval when
   queries contain code fragments ("I want to use @DataSource annotation"). Not benchmarked.

7. **Code miner LLM quality.** Minimisation quality (rewriting production code to minimal
   examples) varies significantly between models. Claude API gives best results;
   smaller local Ollama models (codellama, mistral) are lower quality but air-gapped.
   No quality benchmark exists yet.

8. **Code example licensing.** Mining from open-source projects: Apache 2.0 and MIT
   sources are clearly fine. GPL projects need care. The miner's `--dry-run` mode and
   human review gate allow licence inspection, but no automated licence check is designed.

9. **Code miner pattern library completeness.** JavaParser AST heuristics for pattern
   detection (builders, multi-datasource, etc.) will produce false positives and miss
   some instances. Pattern library is user-extensible but starts with ~10 patterns.
   Quality improves with tuning over time.

---

## What We Are Asking For

Independent review of:

1. Is the retrieval stack (Qdrant + SPLADE + Ollama + Quarkus native) appropriate?
   Are there simpler alternatives we've overlooked?

2. Are the critical validation items (ONNX + native image, LangChain4J dual-client,
   IDF modifier + SPLADE interaction) real blockers, or are we overestimating the risk?

3. Is SPLADE the right sparse model for short technical entries (50–200 lines), or
   should we benchmark BM25 first given the highly specific technical vocabulary?

4. For the code examples garden: is `nomic-embed-text` adequate for use-case-driven
   retrieval, or does code-specific embedding meaningfully improve recall?

5. Is the code miner's LLM-minimisation approach the right way to populate a code
   examples garden, or is there a better pattern for this?

6. Is the per-garden Qdrant service model the right federation approach, or is there
   a simpler design that achieves the same goals?

7. Any performance, scaling, or operational concerns we haven't accounted for?
