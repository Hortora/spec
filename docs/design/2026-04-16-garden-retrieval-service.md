# Garden Retrieval Service — Design Spec

**Date:** 2026-04-16
**Status:** Proposed
**Replaces:** `soredium/scripts/mcp_garden_search.py` (Python, 3-tier hand-rolled retrieval)

---

## Problem

The current retrieval algorithm (`mcp_garden_search.py`) is a hand-rolled approximation of full-text search: domain-directory filter → INDEX.md keyword match → git grep fallback. It has no relevance ranking, no semantic understanding, and no BM25 term weighting. At 240 entries it works. At 10,000 entries it degrades. At community scale it fails.

The garden is also Python-first in its tooling. A Java-shop team contributing entries has no JVM retrieval layer to extend.

---

## Decision

Build `garden-retrieval`: a Quarkus application that provides hybrid BM25 + semantic search over the knowledge garden via Qdrant (self-hosted vector database) and Ollama (local embeddings). Exposes an MCP server endpoint so any MCP-compatible AI assistant can query it directly.

**Why Quarkus + LangChain4J:** The team has LangChain4J experience. Quarkus Dev Services provides zero-config local Qdrant and Ollama. Native image support keeps the service fast and lightweight. MCP server support is built into the Quarkus LangChain4J extension.

**Why Qdrant:** Supports both dense (semantic) and sparse (BM25) vectors natively. ACORN algorithm for filtered HNSW maintains high recall when pre-filtering by metadata. Single self-hosted binary, Rust, low memory.

**Why custom BM25 (~50 lines) instead of Meilisearch:** Meilisearch would be a second search service alongside Qdrant with no other benefit for this use case. BM25 sparse vector computation is straightforward Java and keeps the stack to one vector store. LangChain4J's Qdrant integration handles dense vectors; the Qdrant Java client handles sparse vectors directly for the parts LangChain4J doesn't yet expose.

---

## Architecture

```
Garden entries (git, flat folder or domain dirs)
        │
        │ on: integrate_entry.py merge commit
        ▼
  Indexing endpoint (POST /index)
        │
        ├── SnakeYAML parses YAML frontmatter
        │   → metadata: id, title, domain, type, score, tags, stack, submitted
        │
        ├── Ollama (nomic-embed-text, 768-dim)
        │   → dense vector
        │
        ├── BM25Scorer.computeSparseVector(title + tags + body)
        │   → sparse vector (term → weight map)
        │
        └── Qdrant Java client
            → upsert: dense vector + sparse vector + metadata payload
                       
  Search endpoint (GET /search) / MCP tool: garden_search
        │
        ├── Qdrant hybrid query
        │   ├── Prefetch dense (nomic-embed-text query embedding)
        │   ├── Prefetch sparse (BM25 query sparse vector)
        │   └── Fusion: Reciprocal Rank Fusion (RRF)
        │
        ├── Payload filter (optional, pre-fusion)
        │   domain = "java" AND score >= 10 AND type IN [gotcha, technique]
        │
        └── ONNX reranker (cross-encoder, local, no API)
            → top 5 results returned
```

---

## Repository

`Hortora/garden-retrieval` — standalone Quarkus service, not part of soredium (which stays as Python CI scripts).

```
garden-retrieval/
├── src/main/java/io/hortora/retrieval/
│   ├── GardenEntry.java          ← domain model
│   ├── GardenIndexer.java        ← ingestion pipeline
│   ├── GardenSearcher.java       ← retrieval pipeline
│   ├── BM25Scorer.java           ← sparse vector computation
│   ├── FrontmatterParser.java    ← YAML frontmatter extraction
│   ├── SearchResource.java       ← REST API
│   └── McpGardenServer.java      ← MCP server (via quarkus-langchain4j)
├── src/main/resources/
│   └── application.properties
└── pom.xml
```

---

## Qdrant Collection Schema

One collection: `garden_entries`

```json
{
  "vectors": {
    "dense": {
      "size": 768,
      "distance": "Cosine"
    }
  },
  "sparse_vectors": {
    "bm25": {}
  },
  "payload_schema": {
    "ge_id":     "keyword",
    "title":     "text",
    "domain":    "keyword",
    "type":      "keyword",
    "score":     "integer",
    "tags":      "keyword[]",
    "stack":     "text",
    "submitted": "datetime"
  }
}
```

Qdrant payload indexes (for fast pre-filtering):
```
PAYLOAD INDEX ON domain   (keyword, for domain filter)
PAYLOAD INDEX ON score    (integer, for score >= N filter)
PAYLOAD INDEX ON type     (keyword, for type filter)
PAYLOAD INDEX ON tags     (keyword, for tag filter)
```

---

## BM25Scorer.java (~50 lines)

Fits in memory, built from the corpus of indexed entries. Rebuilt on service start (fast at 10K entries). Persisted to Qdrant as sparse vectors alongside each entry — no separate index needed.

```java
@ApplicationScoped
public class BM25Scorer {

    private static final double K1 = 1.5;
    private static final double B  = 0.75;

    private final Map<String, Double> idf = new HashMap<>();
    private double avgDocLen = 1.0;
    private int totalDocs    = 0;

    /** Call once at startup with all indexed documents. */
    public void fit(List<String> corpus) {
        totalDocs = corpus.size();
        Map<String, Integer> df = new HashMap<>();
        double totalLen = 0;
        for (var doc : corpus) {
            var tokens = tokenize(doc);
            totalLen += tokens.size();
            new HashSet<>(tokens).forEach(t -> df.merge(t, 1, Integer::sum));
        }
        avgDocLen = totalLen / Math.max(totalDocs, 1);
        df.forEach((term, freq) ->
            idf.put(term, Math.log((totalDocs - freq + 0.5) / (freq + 0.5) + 1)));
    }

    /** Compute sparse BM25 vector for a document or query. */
    public Map<String, Float> score(String text) {
        var tokens = tokenize(text);
        var tf = new HashMap<String, Integer>();
        tokens.forEach(t -> tf.merge(t, 1, Integer::sum));

        var result = new HashMap<String, Float>();
        int len = tokens.size();
        tf.forEach((term, freq) -> {
            if (!idf.containsKey(term)) return;
            double tfScore = freq * (K1 + 1)
                / (freq + K1 * (1 - B + B * len / avgDocLen));
            result.put(term, (float) (idf.get(term) * tfScore));
        });
        return result;
    }

    private List<String> tokenize(String text) {
        return Arrays.stream(
            text.toLowerCase().replaceAll("[^a-z0-9@._-]", " ").split("\\s+"))
            .filter(t -> t.length() >= 2)
            .collect(Collectors.toList());
    }
}
```

**Note on tokenization:** The regex preserves `@` (annotations), `.` (package names), `_` (snake_case), `-` (kebab). `@PreUpdate`, `CompletableFuture`, `get_text` all tokenize as single terms and match precisely.

---

## FrontmatterParser.java

```java
@ApplicationScoped
public class FrontmatterParser {

    private static final Pattern FM = Pattern.compile(
        "^---\\n(.*?)\\n---", Pattern.DOTALL);

    @SuppressWarnings("unchecked")
    public GardenEntry parse(String ge_id, String content) {
        var m = FM.matcher(content.replace("\r\n", "\n"));
        if (!m.find()) throw new IllegalArgumentException("No frontmatter: " + ge_id);

        var yaml = new Yaml();
        Map<String, Object> fm = yaml.load(m.group(1));

        var body = content.substring(m.end()).strip();

        return GardenEntry.builder()
            .geId(ge_id)
            .title((String) fm.getOrDefault("title", ""))
            .domain((String) fm.getOrDefault("domain", ""))
            .type((String) fm.getOrDefault("type", "gotcha"))
            .score(((Number) fm.getOrDefault("score", 0)).intValue())
            .tags((List<String>) fm.getOrDefault("tags", List.of()))
            .stack((String) fm.getOrDefault("stack", ""))
            .submitted((String) fm.getOrDefault("submitted", ""))
            .body(body)
            .build();
    }
}
```

---

## Indexing Pipeline

`integrate_entry.py` (existing Python) calls the indexing endpoint after a successful git merge:

```python
# Added to integrate_entry.py after git_commit():
import requests
requests.post("http://localhost:8090/api/index", json={
    "ge_id": ge_id,
    "content": entry_path.read_text()
}, timeout=30)
```

Or: a git post-commit hook on the garden repo that scans for new/modified GE-*.md files and calls the indexing endpoint. Decoupled from soredium.

On the Java side:

```java
@POST
@Path("/api/index")
public void index(IndexRequest req) {
    var entry  = frontmatterParser.parse(req.geId(), req.content());
    var text   = entry.title() + " " + String.join(" ", entry.tags()) + " " + entry.body();

    // Dense vector
    var dense  = embeddingModel.embed(text).content().vectorAsList();

    // Sparse vector (BM25)
    var sparse = bm25Scorer.score(text);

    // Upsert to Qdrant (via Qdrant Java client directly — LangChain4J doesn't expose sparse)
    qdrantClient.upsert(COLLECTION, List.of(
        PointStruct.newBuilder()
            .setId(pointId(req.geId()))
            .setVectors(Vectors.newBuilder()
                .putVectors("dense",  toVector(dense))
                .putVectors("bm25",   toSparseVector(sparse)))
            .setPayload(toPayload(entry))
            .build()
    ));
}
```

---

## Search API

### REST

```
GET /api/search
  ?q=hibernate flush lifecycle callback
  &domain=java          (optional)
  &type=gotcha          (optional)
  &min_score=8          (optional, default: 0)
  &limit=5              (optional, default: 5)
```

Response:
```json
[
  {
    "ge_id": "GE-20260414-aa0001",
    "title": "Hibernate @PreUpdate fires at flush time not at persist",
    "domain": "java",
    "type": "gotcha",
    "score": 12,
    "relevance": 0.94,
    "excerpt": "...",
    "path": "java/GE-20260414-aa0001.md"
  }
]
```

### Qdrant hybrid query

```java
var queryVector = embeddingModel.embed(query).content().vectorAsList();
var sparseQuery = bm25Scorer.score(query);

var filter = buildFilter(domain, type, minScore);  // Qdrant payload filter

var results = qdrantClient.query(COLLECTION, QueryPoints.newBuilder()
    .setPrefetch(Prefetch.newBuilder()
        .setQuery(Query.newBuilder().setNearest(toNamedVector("dense", queryVector)))
        .setFilter(filter)
        .setLimit(20))
    .setPrefetch(Prefetch.newBuilder()
        .setQuery(Query.newBuilder().setNearest(toNamedSparseVector("bm25", sparseQuery)))
        .setFilter(filter)
        .setLimit(20))
    .setQuery(Query.newBuilder().setFusion(Fusion.RRF))  // Reciprocal Rank Fusion
    .setLimit(limit)
    .build());
```

RRF combines the two prefetch results by rank position — no score normalisation needed, robust to distribution differences between dense and sparse.

### Optional: ONNX reranker

If `RERANK_ENABLED=true`: after RRF, fetch the top 20 entries' full content and run a cross-encoder reranker (e.g. `cross-encoder/ms-marco-MiniLM-L6-v2`, ONNX format, ~80MB, local). Return top 5. Adds ~100ms latency on CPU.

LangChain4J supports ONNX reranking natively — no custom code needed here.

---

## MCP Server

The Quarkus LangChain4J extension exposes MCP tools declaratively:

```java
@McpTool(name = "garden_search",
         description = "Search the Hortora knowledge garden for entries matching a query. " +
                       "Returns ranked entries with title, domain, type, score, and excerpt.")
public List<SearchResult> gardenSearch(
    @McpToolParam(name = "query",      description = "Natural language query or symptom") String query,
    @McpToolParam(name = "domain",     description = "Filter by domain (java, tools, python...)") String domain,
    @McpToolParam(name = "type",       description = "Filter by type (gotcha, technique, undocumented)") String type,
    @McpToolParam(name = "min_score",  description = "Minimum garden score (1-15)") Integer minScore
) {
    return gardenSearcher.search(query, domain, type, minScore, 5);
}
```

MCP transport: stdio (for Claude Code, Cursor, Copilot) or HTTP SSE (for web clients).

Config in `~/.claude/claude_desktop_config.json` or equivalent:
```json
{
  "mcpServers": {
    "hortora-garden": {
      "command": "java",
      "args": ["-jar", "/path/to/garden-retrieval.jar"],
      "env": { "GARDEN_PATH": "/path/to/garden" }
    }
  }
}
```

---

## application.properties

```properties
# Qdrant
quarkus.langchain4j.qdrant.host=localhost
quarkus.langchain4j.qdrant.port=6334
quarkus.langchain4j.qdrant.collection-name=garden_entries

# Ollama
quarkus.langchain4j.ollama.embedding-model.model-id=nomic-embed-text
quarkus.langchain4j.ollama.embedding-model.dimensions=768
quarkus.langchain4j.ollama.base-url=http://localhost:11434

# Dev Services (zero config in dev mode)
quarkus.langchain4j.devservices.qdrant.enabled=true
quarkus.langchain4j.devservices.ollama.enabled=true

# Reranker (optional)
garden.retrieval.rerank.enabled=false
garden.retrieval.rerank.model=cross-encoder/ms-marco-MiniLM-L6-v2

# MCP server port
quarkus.mcp.server.http.port=8090
```

In dev mode (`quarkus dev`): Qdrant and Ollama start automatically via Dev Services. Zero manual setup for a new developer.

---

## Migration from Python MCP Server

The existing `garden_mcp_server.py` stays operational during the migration. Switch is a config change in the MCP client:

```
Before: python3 garden_mcp_server.py  (port 8091)
After:  java -jar garden-retrieval.jar (port 8090)
```

The Python server's `garden_status` and `garden_capture` tools can be proxied through the Java server or stay as separate Python tools initially — they don't need to move on day one.

---

## What stays Python

The soredium Python scripts are CI tools that run in GitHub Actions or locally on demand. They don't need to be fast or long-lived:

| Script | Stays Python | Reason |
|--------|--------------|--------|
| `validate_pr.py` | ✅ | CI script, runs in GH Actions |
| `integrate_entry.py` | ✅ | CI script; calls Java indexing endpoint |
| `garden_db_migrate.py` | ✅ | One-time migration script |
| `dedupe_scanner.py` | ✅ | Could eventually be replaced by Qdrant similarity search |
| `garden_db.py` | ✅ | SQLite state (CHECKED.md replacement) stays |
| `mcp_garden_search.py` | ❌ | Replaced by Java MCP server |
| `mcp_garden_status.py` | ⚠️ | Move later — simple, not urgent |
| `mcp_garden_capture.py` | ⚠️ | Move later — creates git branches |

---

## Open Questions

1. **Flat folder vs domain directories for entries?** The retrieval service makes domain directories optional — domain is just a payload field. But GitHub browsability and CI routing still benefit from them. Decision deferred; service works either way.

2. **BM25 corpus rebuild:** Currently in-memory, rebuilt at startup. At 10K entries this takes ~2 seconds. At 100K entries consider persisting the IDF table to Qdrant as a special document or to a sidecar SQLite.

3. **Cold start for native image:** Quarkus native image compiles the ONNX reranker and BM25 at build time. The embedding call to Ollama warms up on first request. Acceptable for a long-running service.

4. **Embedding model choice:** `nomic-embed-text` (768-dim) is the default. For better technical content recall, `mxbai-embed-large` (1024-dim) or a code-specific model (CodeBERT, UniXcoder) may improve retrieval for technical gotchas. Worth benchmarking once the service is live.

5. **Garden_capture in Java vs Python:** Creating a git branch and opening a PR is possible in Java (JGit + GitHub API client) but the Python version already works. Migrate only when there's a reason.
