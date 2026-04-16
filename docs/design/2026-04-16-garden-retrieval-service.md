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

## Partitioning Strategy

**Domain is the natural top-level partitioning key** for both Git and Qdrant. The
garden-per-domain model (each canonical garden is a separate Git repo + Qdrant collection)
provides consistent partitioning at every layer of the stack.

### Why partitioning matters at scale

At 1M entries in a single Qdrant collection:

| | Single collection (1M) | Per-domain collections (~100K each) |
|---|---|---|
| HNSW graph RAM | 10–15GB | 1–1.5GB per collection |
| Query latency | 15–40ms | 5–15ms |
| Qdrant startup | 3–8 minutes | 20–60 seconds |
| IDF corpus scope | Cross-domain (noisy) | Per-domain (meaningful) |

The IDF point is significant: `"modifier": "idf"` computes term rarity within the
collection. "hibernate" in a Java-only collection has more precise IDF weighting than
"hibernate" in a mixed collection containing tools, Python, and frontend entries.
Domain partitioning improves BM25/SPLADE scoring quality, not just performance.

### Level 1: Garden-per-collection (primary partition)

Each canonical garden maps to one Qdrant collection. This is the natural unit because
each canonical garden also has its own Git repo and `garden-retrieval` service.

```
jvm-garden     → Qdrant collection: garden_entries  (on jvm-garden's Qdrant instance)
tools-garden   → Qdrant collection: garden_entries  (on tools-garden's Qdrant instance)
python-garden  → Qdrant collection: garden_entries  (on python-garden's Qdrant instance)
```

Domain-filtered queries hit only one garden's service. Cross-garden queries fan out to
multiple services and merge results.

### Level 2: Domain-per-collection (secondary partition, at scale)

If a single canonical garden grows very large (>200K entries), split further by domain
within that garden. `jvm-garden` with 500K entries across java, quarkus, spring, kotlin:

```
jvm_java       → Qdrant collection
jvm_quarkus    → Qdrant collection
jvm_kotlin     → Qdrant collection
```

The `garden-retrieval` service resolves the target collection from the `domain` parameter.
When `domain` is unspecified, it queries all collections and merges via RRF.

### Level 3: Qdrant distributed sharding (horizontal scaling)

When a single collection exceeds what one Qdrant instance can serve (hundreds of millions
of entries), Qdrant's distributed mode shards across multiple nodes. The shard key is
`domain` — all vectors with the same domain land on the same shard, so domain-filtered
queries hit only one shard without scatter-gather.

This level is not needed until well beyond 1M entries. Design for Level 1 now;
Level 2 is a collection split (no code changes); Level 3 is a Qdrant deployment change.

---

## Qdrant Collection Schema

One collection per garden (or per domain within a large garden). Schema is identical
across all collections:

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

Payload indexes on `domain`, `score`, `type`, `tags` for fast pre-filtering.

`"modifier": "idf"` — Qdrant applies IDF weighting at query time from live corpus
statistics. No IDF table in Java. Corpus statistics are per-collection, so domain
partitioning makes them more accurate.

### Collection naming convention

```
{garden-name}_entries          # single-domain garden
{garden-name}_{domain}         # split by domain within a large garden

Examples:
  jvm_entries
  tools_entries
  jvm_quarkus                  # when jvm-garden splits quarkus out
```

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
  &domain=java          (optional — routes to correct collection)
  &type=gotcha          (optional payload filter)
  &min_score=8          (optional payload filter, default: 0)
  &limit=5              (optional, default: 5 for recall, 3 for precision)
```

### Collection routing

The service resolves the target Qdrant collection from the `domain` parameter:

```java
private String resolveCollection(String domain) {
    if (domain == null) return "garden_entries";          // search all
    return collectionMap.getOrDefault(domain, "garden_entries");
    // e.g. "java" → "jvm_java", "quarkus" → "jvm_quarkus"
    //      "tools" → "tools_entries" (when split)
}
```

When `domain` is null, the service fans out to all collections for this garden and
merges results via RRF before returning. For most queries, `domain` is specified and
only one collection is touched — O(log n) per collection rather than O(log N) across all.

### Qdrant hybrid query (same structure for both modes)

```java
public List<SearchResult> search(String query, String domain,
                                  String type, int minScore,
                                  int limit, boolean precision) {

    var collection = resolveCollection(domain);

    var filter = Filter.must(
        type      != null ? fieldCondition("type",  type)     : null,
        minScore  > 0     ? rangeCondition("score", minScore) : null
        // domain filter NOT needed when routed to domain-specific collection
    );

    var results = qdrantClient.queryHybrid(collection,
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

## Sparse Model

`prithivida/Splade_PP_en_v1` (Apache 2.0). ONNX model run client-side via ONNX Runtime
Java binding. No corpus statistics — scales identically from 100 to 1M entries.

NAVER's original SPLADE models (naver/splade-v3 etc.) are **CC-BY-NC-SA — non-commercial**.
Do not use in production.

---

## Federation

Each garden is an independent stack. Retrieval federation is service-to-service HTTP —
no local clone of the parent garden required for upstream search.

### Per-garden stack

```
garden/
  ├── git repo          ← entries, SCHEMA.md (source of truth, captures go here)
  ├── Qdrant            ← vector index for this garden's entries only
  └── garden-retrieval  ← REST API + MCP server
```

`init_garden.py` generates a `docker-compose.yml` for the full local stack:

```yaml
services:
  qdrant:
    image: qdrant/qdrant
    volumes: ["./qdrant_storage:/qdrant/storage"]
    ports: ["6333:6333", "6334:6334"]

  garden-retrieval:
    image: hortora/garden-retrieval
    environment:
      QDRANT_HOST: qdrant
      GARDEN_PATH: /garden
    ports: ["8090:8090"]
    volumes: [".:/garden"]
    depends_on: [qdrant]
```

### Client config

`~/.claude/garden-config.toml` maps garden names to both local paths (for git operations)
and retrieval URLs (for search):

```toml
[[gardens]]
name = "jvm-garden"
path = "~/.hortora/jvm-garden"
retrieval_url = "https://api.hortora.io/jvm"    # canonical, publicly hosted

[[gardens]]
name = "my-private-garden"
path = "~/work/my-garden"
retrieval_url = "http://localhost:8090"          # child, running locally
```

### Upstream chain walk

Child garden's retrieval service searches its local Qdrant first. On upstream query:

```
GET https://api.hortora.io/jvm/api/search?q=hibernate+flush&mode=recall
```

Parent's SPLADE index is always current — no sync, no stale local copies of parent entries.

### Non-duplication on submission

When a PR arrives at a child garden, the child's retrieval service calls the parent's
search API with the new entry's full text. SPLADE similarity at the parent catches
semantic duplicates that Jaccard title/tag matching misses.

```python
# In validate_pr.py — semantic upstream check replaces Jaccard
response = requests.get(f"{parent_url}/api/search",
    params={"q": entry_text, "mode": "precision", "limit": 3})
if any(r["relevance_score"] > DUPLICATE_THRESHOLD for r in response.json()):
    log_error("Likely duplicate of upstream entry")
```

### Performance at scale

| Metric | Estimate (beefy server, A100 GPU) |
|--------|----------------------------------|
| 1000 entries bulk import | 10–30 seconds |
| Restart, 1M entries, indexes preserved | 3–8 minutes |
| Restart from Qdrant snapshot | 10–20 minutes |
| Full rebuild from source (1M entries) | 1–2 hours |
| Search latency, recall mode (1M entries) | 15–40ms |
| Search latency, precision mode (1M entries) | 70–150ms |

Qdrant snapshots are the catastrophic recovery mechanism at 1M entries — not full
re-embedding. Schedule daily snapshots; retain the last N.

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
