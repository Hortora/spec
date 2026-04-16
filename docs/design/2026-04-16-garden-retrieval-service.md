# Garden Retrieval Service — Design Spec

**Date:** 2026-04-16
**Status:** Proposed
**Replaces:** `soredium/scripts/mcp_garden_search.py` (Python, 3-tier hand-rolled retrieval)

---

## Problem

The current retrieval algorithm (`mcp_garden_search.py`) is a hand-rolled approximation of full-text search: domain-directory filter → INDEX.md keyword match → git grep fallback. No relevance ranking. No semantic understanding. No BM25 term weighting. At 240 entries it works. At 10,000 entries it degrades. At community scale it fails.

---

## Decision

Build `garden-retrieval`: a native Quarkus application providing hybrid BM25 + semantic search over the knowledge garden. Intelligence lives in Qdrant. The Java service is an orchestration layer and API surface.

**Technology:**
- **Quarkus** (native image — fast startup, ~80MB footprint, distributable binary)
- **LangChain4J** (Quarkus extension — RAG pipeline, MCP server, ONNX reranking)
- **Qdrant** (self-hosted — hybrid search engine, manages all embedding server-side)
- **Ollama** (local — dense embeddings via `nomic-embed-text`, no API key)

**No custom retrieval code.** BM25 sparse vectors, dense vectors, hybrid query fusion, and metadata filtering all run inside Qdrant. The Java service sends text; Qdrant handles everything else.

---

## Why Qdrant handles both embedding types

Qdrant ships with **FastEmbed** — an embedded inference library running inside the Qdrant server process. Two models, both Apache 2.0:

| Vector type | Model | Provided by | License |
|-------------|-------|-------------|---------|
| Dense (semantic) | `nomic-embed-text` (768-dim) | Ollama | Apache 2.0 |
| Sparse (BM25) | `Qdrant/bm25` | FastEmbed (inside Qdrant) | Apache 2.0 |

The Java service **never computes embeddings**. It passes raw text to Qdrant at index time and raw query text at search time. Qdrant generates both vector types server-side and handles all corpus statistics for BM25 internally.

This eliminates:
- Any in-memory IDF table in Java
- Corpus rebuild at startup
- BM25 scaling problems
- Migration paths — `Qdrant/bm25` works identically at 1,000 and 1,000,000 entries

When BM25 is insufficient at very high scale: upgrade to `prithivida/Splade_PP_en_v1` (Apache 2.0 SPLADE++ via FastEmbed) — a config change in Qdrant, zero Java code changes.

---

## Architecture

```
Garden entries (git, markdown + YAML frontmatter)
        │
        │  on: new entry merged (integrate_entry.py calls /api/index)
        ▼
  ┌─────────────────────────────────┐
  │  garden-retrieval (Quarkus)     │
  │                                 │
  │  POST /api/index                │
  │    parse YAML frontmatter       │
  │    extract: title, domain,      │
  │             type, score, tags,  │
  │             stack, body         │
  │    → Qdrant upsert (text only)  │
  └────────────┬────────────────────┘
               │
               ▼
  ┌─────────────────────────────────┐
  │  Qdrant (self-hosted)           │
  │                                 │
  │  FastEmbed generates:           │
  │    dense  ← nomic-embed-text    │
  │    sparse ← Qdrant/bm25         │
  │                                 │
  │  Stores:                        │
  │    dense vector (768-dim)       │
  │    sparse vector (BM25 terms)   │
  │    payload (metadata fields)    │
  └─────────────────────────────────┘

  GET /api/search?q=...&mode=recall|precision
        │
        ├── build payload filter (domain, type, score, tags)
        ├── Qdrant hybrid query (dense prefetch + sparse prefetch + RRF)
        ├── [precision mode only] ONNX reranker (local, no API)
        └── return ranked entries (full body or card format)
```

---

## Repository

`Hortora/garden-retrieval` — standalone Quarkus service.

```
garden-retrieval/
├── src/main/java/io/hortora/retrieval/
│   ├── GardenEntry.java          ← domain model (record)
│   ├── FrontmatterParser.java    ← YAML frontmatter extraction (~30 lines)
│   ├── GardenIndexer.java        ← ingestion: parse → Qdrant upsert
│   ├── GardenSearcher.java       ← search: build query → Qdrant → format response
│   ├── SearchResource.java       ← REST API (/api/index, /api/search)
│   └── GardenMcpServer.java      ← MCP tools (garden_search, garden_status)
├── src/main/resources/
│   └── application.properties
└── pom.xml
```

---

## Qdrant Collection Setup

One collection: `garden_entries`

```json
{
  "vectors": {
    "dense": { "size": 768, "distance": "Cosine" }
  },
  "sparse_vectors": {
    "bm25": { "modifier": "idf" }
  },
  "payload_schema": {
    "ge_id":     "keyword",
    "domain":    "keyword",
    "type":      "keyword",
    "score":     "integer",
    "tags":      "keyword[]",
    "submitted": "datetime"
  }
}
```

Qdrant payload indexes on `domain`, `score`, `type`, `tags` for fast pre-filtering before vector search.

The `"modifier": "idf"` on the sparse vector tells Qdrant to apply IDF weighting to BM25 — corpus statistics managed entirely by Qdrant.

---

## FrontmatterParser.java (~30 lines)

The only custom logic in the Java service. SnakeYAML is already on the Quarkus classpath.

```java
@ApplicationScoped
public class FrontmatterParser {

    private static final Pattern FM = Pattern.compile(
        "^---\\n(.*?)\\n---", Pattern.DOTALL);

    @SuppressWarnings("unchecked")
    public GardenEntry parse(String geId, String content) {
        var m = FM.matcher(content.replace("\r\n", "\n"));
        if (!m.find())
            throw new IllegalArgumentException("No frontmatter in: " + geId);

        Map<String, Object> fm = new Yaml().load(m.group(1));
        String body = content.substring(m.end()).strip();

        return new GardenEntry(
            geId,
            (String)  fm.getOrDefault("title",   ""),
            (String)  fm.getOrDefault("domain",  ""),
            (String)  fm.getOrDefault("type",    "gotcha"),
            ((Number) fm.getOrDefault("score",    0)).intValue(),
            (List<String>) fm.getOrDefault("tags", List.of()),
            (String)  fm.getOrDefault("stack",   ""),
            (String)  fm.getOrDefault("submitted",""),
            body
        );
    }
}
```

---

## Indexing

`integrate_entry.py` (existing Python CI script) calls the indexing endpoint after a successful git merge. The Java service does no embedding — it passes text to Qdrant and Qdrant does everything.

```java
@POST @Path("/api/index")
public void index(IndexRequest req) {
    var entry = frontmatterParser.parse(req.geId(), req.content());

    // Text for embedding: title + tags + body (Qdrant embeds this server-side)
    var text = entry.title() + " "
             + String.join(" ", entry.tags()) + " "
             + entry.body();

    // Payload: metadata for filtering
    var payload = Map.of(
        "ge_id",     entry.geId(),
        "title",     entry.title(),
        "domain",    entry.domain(),
        "type",      entry.type(),
        "score",     entry.score(),
        "tags",      entry.tags(),
        "stack",     entry.stack(),
        "submitted", entry.submitted()
    );

    // Qdrant generates dense (nomic-embed-text) + sparse (Qdrant/bm25) server-side
    qdrantClient.upsert("garden_entries", text, payload);
}
```

---

## Two Search Modes

The service has two distinct retrieval modes based on who consumes the results.

### Mode 1: Recall (LLM downstream)

An LLM on the end-user side (Claude Code, Copilot, Cursor) reads all returned entries and selects the relevant one. The service's job is to ensure the right answer is somewhere in the result set. The LLM is the reranker.

- **Result count:** 5–8
- **Content:** Full markdown body — LLM must read the full entry to determine relevance
- **Reranker:** None — the LLM reranks for free, and better than any ONNX model
- **Latency target:** < 200ms
- **Optimise for:** Recall (right answer in the set)

```json
[
  {
    "ge_id": "GE-20260414-aa0001",
    "title": "Hibernate @PreUpdate fires at flush time not at persist",
    "domain": "java",
    "type": "gotcha",
    "score": 12,
    "relevance_rank": 1,
    "content": "## Hibernate @PreUpdate...\n\n**Symptom:**...\n\n### Root cause\n..."
  }
]
```

### Mode 2: Precision (Human downstream)

A human is using the garden browser or a CLI. They want the best result at the top and don't want to read 8 entries. The service must surface the right answer itself.

- **Result count:** 1–3
- **Content:** Title + symptom excerpt + fix snippet + URL (human scans cards)
- **Reranker:** ONNX cross-encoder (local, no API, ~100ms) — necessary because there's no LLM downstream
- **Latency target:** < 500ms
- **Optimise for:** Precision (right answer at position 1)

```json
[
  {
    "ge_id": "GE-20260414-aa0001",
    "title": "Hibernate @PreUpdate fires at flush time not at persist",
    "type": "gotcha",
    "score": 12,
    "staleness_days": 1,
    "staleness_threshold": 730,
    "symptom": "@PreUpdate callback not firing when expected.",
    "fix_snippet": "Force a flush or restructure lifecycle logic.",
    "url": "https://github.com/Hortora/garden/blob/main/java/GE-20260414-aa0001.md"
  }
]
```

---

## Search API

```
GET /api/search
  ?q=hibernate flush lifecycle callback
  &mode=recall          (default: recall)
  &domain=java          (optional payload filter)
  &type=gotcha          (optional payload filter)
  &min_score=8          (optional payload filter, default: 0)
  &limit=5              (optional, default: 5 for recall, 3 for precision)
```

### Qdrant hybrid query (same structure for both modes)

```java
public List<SearchResult> search(String query, String domain,
                                  String type, int minScore,
                                  int limit, boolean precision) {

    var filter = Filter.must(
        domain    != null ? fieldCondition("domain", domain)   : null,
        type      != null ? fieldCondition("type",   type)     : null,
        minScore  > 0     ? rangeCondition("score",  minScore) : null
    );

    // Qdrant generates query vectors server-side (same models as indexing)
    var results = qdrantClient.queryHybrid("garden_entries",
        QueryRequest.builder()
            .prefetch(dense(query,  filter, limit * 4))   // semantic candidates
            .prefetch(sparse(query, filter, limit * 4))   // BM25 candidates
            .fusion(Fusion.RRF)                           // Reciprocal Rank Fusion
            .limit(precision ? limit * 3 : limit)         // extra for reranker
            .withPayload(true)
            .build());

    if (precision) {
        results = onnxReranker.rerank(query, results, limit);
    }

    return formatResults(results, precision);
}
```

Reciprocal Rank Fusion combines the dense and sparse ranked lists by position — no score normalisation needed, robust to distribution differences between vector types.

---

## MCP Server

Exposed via Quarkus LangChain4J MCP extension. Both tools use recall mode by default — an LLM is always downstream in MCP contexts.

```java
@McpTool(
    name = "garden_search",
    description = "Search the Hortora knowledge garden. Returns 5 ranked entries " +
                  "for the calling LLM to evaluate. Supports domain, type, and " +
                  "minimum score filters."
)
public List<SearchResult> gardenSearch(
    @McpToolParam(name = "query",     description = "Symptom or problem description") String query,
    @McpToolParam(name = "domain",    description = "Technology domain: java, tools, python...") String domain,
    @McpToolParam(name = "type",      description = "Entry type: gotcha, technique, undocumented") String type,
    @McpToolParam(name = "min_score", description = "Minimum garden score (1–15)") Integer minScore
) {
    return gardenSearcher.search(query, domain, type,
        minScore != null ? minScore : 0, 5, false); // recall mode
}
```

MCP transport: stdio (Claude Code, Cursor) or HTTP SSE (web clients). Config:

```json
{
  "mcpServers": {
    "hortora-garden": {
      "command": "garden-retrieval",
      "args": [],
      "env": { "QDRANT_HOST": "localhost", "QDRANT_PORT": "6334" }
    }
  }
}
```

---

## application.properties

```properties
# Qdrant
qdrant.host=localhost
qdrant.port=6334
qdrant.collection=garden_entries

# Ollama (dense embeddings for indexing — Qdrant uses nomic-embed-text via FastEmbed)
# Note: if Qdrant FastEmbed handles dense embedding too, Ollama can be removed entirely
quarkus.langchain4j.ollama.base-url=http://localhost:11434
quarkus.langchain4j.ollama.embedding-model.model-id=nomic-embed-text

# ONNX reranker (precision mode only — optional, disable if native image issues)
garden.retrieval.rerank.enabled=true
garden.retrieval.rerank.model=cross-encoder/ms-marco-MiniLM-L6-v2

# Dev Services — zero config in quarkus dev mode
quarkus.langchain4j.devservices.qdrant.enabled=true
quarkus.langchain4j.devservices.ollama.enabled=true

# Search defaults
garden.retrieval.recall.limit=5
garden.retrieval.precision.limit=3
```

---

## Native Image

Target: GraalVM native image. Implications:

- Startup: milliseconds (runs as developer laptop sidecar)
- Memory: ~80MB (acceptable alongside an IDE)
- Distribution: single binary (`brew install hortora-garden` or similar)
- ONNX reranker: uses JNI — test native image compatibility early; if problematic, precision mode falls back to Qdrant-ranked results without reranking (acceptable degradation)
- Quarkus native: all Qdrant client calls, REST, MCP are native-image-safe; SnakeYAML requires reflection config (standard Quarkus setup)

---

## Sparse Model Upgrade Path

No code changes required at any scale. Config change in Qdrant only.

| Scale | Sparse model | Notes |
|-------|-------------|-------|
| Any | `Qdrant/bm25` (FastEmbed) | Default, Apache 2.0, Qdrant manages corpus stats |
| 1M+ | `prithivida/Splade_PP_en_v1` (FastEmbed) | Learned sparse, Apache 2.0, better semantic coverage |

To upgrade: change the Qdrant collection sparse vector config and reindex. Java service unchanged.

NAVER's original SPLADE models (naver/splade-v3 etc.) are **CC-BY-NC-SA — non-commercial**. Do not use in production.

---

## What Stays Python (soredium)

| Script | Stays Python | Notes |
|--------|--------------|-------|
| `validate_pr.py` | ✅ | CI script — runs in GitHub Actions |
| `integrate_entry.py` | ✅ | CI script — calls `/api/index` after merge |
| `garden_db.py` + `garden_db_migrate.py` | ✅ | CHECKED.md replacement (SQLite operational state) |
| `dedupe_scanner.py` | ✅ | Could eventually use Qdrant similarity for dedup |
| `mcp_garden_search.py` | ❌ | Replaced by Java MCP server |
| `mcp_garden_status.py` | ⚠️ | Move later |
| `mcp_garden_capture.py` | ⚠️ | Move later — creates git branches |

---

## Open Questions

1. **Dense embedding: Ollama vs Qdrant FastEmbed.** If Qdrant's FastEmbed supports `nomic-embed-text` for dense vectors too, the Ollama dependency disappears and Qdrant handles both vector types entirely. Worth checking at implementation time — would simplify the stack further.

2. **ONNX reranker in native image.** JNI-based; test early on all target platforms. If it proves unreliable in native image on macOS ARM, precision mode falls back to Qdrant-ranked results without reranking — still better than the current algorithm.

3. **Flat folder vs domain directories.** The retrieval service is indifferent — domain is a payload filter field, not a directory path. Decision deferred to the garden structure discussion.

4. **Garden browser (Phase 7) upgrade.** The garden browser currently uses client-side Fuse.js. Replacing it with a call to `?mode=precision` on this service gives proper hybrid search + reranking at no additional infrastructure cost (Qdrant is already running).
