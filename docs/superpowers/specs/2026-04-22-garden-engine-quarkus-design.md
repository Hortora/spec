# Garden Engine — Quarkus Native Design

**Status:** Draft  
**Replaces:** Python mining pipeline (`soredium/scripts/run_pipeline.py` et al.)  
**Extends:** Dedup agent (garden-agent), harvest skill  
**Relates to:** `2026-04-16-garden-retrieval-service.md` (shared Qdrant instance)

---

## Goal

Port the ecosystem mining pipeline and the deduplication/harvest layer to a single
native Quarkus application (`garden-engine`). Integrate Langchain4j at both layers
to move from structural heuristics (Jaccard, cosine on regex counts) to semantic
similarity and LLM-driven reasoning (pattern naming, entry merging, classification).

The Python pipeline stays as the **reference spec** — its tests define the Java
port acceptance criteria. The Python scripts are retired once the Java service
passes them.

---

## What Changes and Why

### Mining side

| Current (Python) | Garden Engine (Java) |
|---|---|
| Regex over source files → raw count fingerprint | Same regex port to Java — `Pattern` + `Files.walk` |
| Min-max cosine clustering | **Replace with Qdrant semantic similarity** — fingerprint converted to text description, Qdrant embeds and clusters |
| No pattern naming | **New:** Claude reads cluster members, names the pattern, explains the shared decision |
| Delta analysis: new abstractions between tags | Same logic ported to Java via `ProcessBuilder(git...)` |
| No delta narrative | **New:** Claude reads the diff, explains the architectural decision made |

### Harvest/dedup side

| Current (Python + garden agent) | Garden Engine (Java) |
|---|---|
| Jaccard similarity (character n-gram overlap) | **Replace with Qdrant semantic search** — finds near-duplicates by meaning, not text overlap |
| Human classification: distinct / related / duplicate | **New:** Claude classifies the pair with reasoning, human confirms or overrides |
| "Related" → append `**See also:**` | Same, but Claude generates the cross-reference text |
| "Duplicate" → keep higher score, delete other | **New:** Claude synthesises both entries into one enriched entry rather than discarding |
| Validation: YAML schema check + Jaccard scan | **Extend:** semantic duplicate check against Qdrant at submit time |

---

## LLM Stack

Two separate concerns, two tools:

| Concern | Tool | Why |
|---|---|---|
| Semantic similarity (dedup, cluster candidates) | **Qdrant + FastEmbed** (same as retrieval service) | Server-side, free, already deployed, no Java embedding code needed |
| Reasoning (pattern naming, classification, merge synthesis) | **Claude via Langchain4j Anthropic extension** | Best structured output, already the primary model in the ecosystem |

The Java service never computes embeddings. It passes text to Qdrant and Qdrant
does all vector work — identical to the retrieval service. Claude handles all
tasks that require judgment.

```properties
# Shared with retrieval service — same Qdrant instance
quarkus.langchain4j.anthropic.api-key=${ANTHROPIC_API_KEY}
quarkus.langchain4j.anthropic.model-name=claude-sonnet-4-6
quarkus.langchain4j.anthropic.max-tokens=4096
```

---

## Architecture

Single Quarkus application with two execution modes:

```
garden-engine
├── CLI mode (default)        — triggered by scripts, cron, or garden post-commit hook
│   ├── mine [--project <name>|--all]   — run mining pipeline
│   └── harvest [--sweep|--review]      — run dedup/harvest pipeline
└── REST mode (optional)      — HTTP trigger for CI integration
    ├── POST /api/mine
    └── POST /api/harvest
```

GraalVM native image for both modes — fast cold start matters for the post-commit
hook use case (currently garden-agent.sh is already slow due to JVM startup).

---

## Component Map (Python → Java)

### Feature Extractor

`scripts/feature_extractor.py` → `FeatureExtractor.java`

```java
@ApplicationScoped
public class FeatureExtractor {

    private static final Pattern JAVA_INTERFACE =
        Pattern.compile("\\b(?:interface|abstract\\s+class)\\b");
    private static final Pattern JAVA_INJECT =
        Pattern.compile("@(?:Inject|Autowired|ApplicationScoped|RequestScoped|Singleton)");
    private static final Pattern JAVA_EXTENDS =
        Pattern.compile("\\bclass\\s+\\w+(?:\\s*<[^>]*>)?\\s+(?:extends|implements)\\b");

    public Fingerprint extract(Path root) throws IOException {
        var counts = new AtomicReference<>(new int[4]); // iface, inject, extends, spi
        var fileCount = new AtomicInteger();

        try (var stream = Files.walk(root)) {
            stream.filter(Files::isRegularFile)
                  .filter(p -> isSpiFile(p) || isSourceFile(p))
                  .forEach(p -> process(p, counts, fileCount));
        }
        int[] c = counts.get();
        int fc = fileCount.get();
        return new Fingerprint(c[0], fc > 0 ? (double) c[0] / fc : 0.0,
                               c[1], c[2], fc, c[3]);
    }
}
```

**Known gotcha (from garden):** `Files.walk` follows symlinks by default —
use `Files.walk(root, FileVisitOption.FOLLOW_LINKS)` deliberately or pass
`FileVisitOption` set explicitly to avoid surprise traversal of symlinked dirs.

### Cluster Pipeline

`scripts/cluster_pipeline.py` → **replaced by Qdrant semantic clustering**

Instead of cosine similarity on ratio fingerprints, we:

1. Convert each fingerprint to a natural-language description:
   ```
   "JVM project with 253 interfaces per 1000 files, 403 injection points per 1000 files,
    468 extension signatures per 1000 files, 9 SPI service files per 1000 files.
    Abstraction depth ratio: 0.25."
   ```
2. Index into Qdrant collection `mining-fingerprints`
3. Query for nearest neighbours — Qdrant finds semantic clusters

This eliminates the normalisation bug entirely (no min-max collapse) and
produces distance-based groupings that a human can understand.

### Delta Analysis

`scripts/delta_analysis.py` → `DeltaAnalysis.java`

Same logic — `ProcessBuilder` calling `git diff`, `git show`, `git log`.
Requires blobless clone (`--filter=blob:none`), not shallow. The registry
records the clone strategy per project.

### Pattern Naming Service (new)

```java
@RegisterAiService
public interface PatternNamingService {

    @SystemMessage("""
        You are an expert in JVM framework architecture. You will be given a cluster
        of structurally similar projects and representative source files from each.
        Identify the shared architectural pattern, give it a precise name, and explain
        what structural decision they share and why it exists.
        Respond as JSON: {name, description, structural_signal, why_it_exists}
        """)
    PatternCandidate namePattern(@UserMessage String clusterContext);
}
```

### Delta Narrative Service (new)

```java
@RegisterAiService
public interface DeltaNarrativeService {

    @SystemMessage("""
        You are an expert in JVM framework architecture. You will be given a git diff
        between two versions of a project showing new interface or abstract class files.
        Explain the architectural decision made: what problem does this new abstraction
        solve, and what pattern does it introduce?
        Respond as JSON: {decision, pattern_name, motivation, introduced_at}
        """)
    DeltaNarrative explainDelta(@UserMessage String diffContext);
}
```

### Semantic Deduplicator (harvest upgrade)

`scripts/dedupe_scanner.py` → `SemanticDeduplicator.java`

```java
@ApplicationScoped
public class SemanticDeduplicator {

    @Inject QdrantClient qdrant;
    @Inject DedupeClassifier classifier;  // AI service

    public List<DedupePair> findCandidates(GardenEntry entry) {
        // Search Qdrant for semantically similar entries
        var hits = qdrant.search("garden-entries",
            entry.title() + " " + entry.body(), 10, 0.85f);

        return hits.stream()
            .filter(h -> !h.id().equals(entry.id()))
            .map(h -> new DedupePair(entry, h.payload(), h.score()))
            .toList();
    }
}
```

### Dedup Classifier (replaces human-only pair review)

```java
@RegisterAiService
public interface DedupeClassifier {

    @SystemMessage("""
        You are reviewing two knowledge garden entries for duplication.
        Classify as: DISTINCT (different topics), RELATED (same area, worth a See Also link),
        or DUPLICATE (same knowledge, one should be merged into the other).
        If DUPLICATE, identify which entry to keep and what to preserve from the other.
        Respond as JSON: {classification, reasoning, keep_id, preserve_from_other}
        """)
    DedupeDecision classify(@UserMessage String entryPair);
}
```

### Entry Merge Service (new — replaces discard)

When two entries are classified as DUPLICATE, instead of discarding the lower-scored
one, Claude synthesises both into a single enriched entry:

```java
@RegisterAiService
public interface EntryMergeService {

    @SystemMessage("""
        You are merging two knowledge garden entries that cover the same topic.
        Produce a single enriched entry that preserves the best elements of both:
        the most precise title, the most complete root cause, all code examples,
        all caveats from both entries, and the highest score.
        Output the full merged entry in the garden YAML frontmatter + body format.
        """)
    String mergeEntries(@UserMessage String entryPair);
}
```

---

## Improved Harvest Pipeline

The full semantic harvest flow replaces the current Jaccard-based dedupe agent:

```
1. New entry committed to garden
2. garden-engine harvest triggered (post-commit hook or REST)
3. SemanticDeduplicator searches Qdrant for similar entries (threshold: 0.85)
4. For each candidate pair:
   a. DedupeClassifier classifies: DISTINCT / RELATED / DUPLICATE
   b. DISTINCT → record in CHECKED.md, no action
   c. RELATED → Claude generates See Also text, both entries updated
   d. DUPLICATE → EntryMergeService synthesises merged entry
              → original two entries replaced by merged entry
              → DISCARDED.md records what was merged and why
5. New/merged entries indexed into Qdrant
6. Commit: "harvest: semantic sweep — N related, M merged"
```

This runs **fully autonomously** — the garden agent invokes `garden-engine harvest`
instead of Claude running the dedup CLAUDE.md instructions manually. Claude's judgment
is inside the registered AI services, not in the agent prompt.

---

## Validation Upgrade

`scripts/validate_pr.py` (Python) stays for YAML schema validation — no reason
to port a working schema checker. Garden Engine adds:

- **Semantic duplicate check at submit time**: before any entry is accepted,
  search Qdrant for existing entries scoring > 0.92. If found, reject with:
  `"Near-duplicate of GE-XXXX '[title]' (similarity: 0.94). Consider REVISE instead."`
- **Score floor enforcement**: entries < 8 are rejected at the Java API level,
  not just warned about in Python.

---

## Native Image Considerations

- `ProcessBuilder` for git — works in native, no reflection issues
- Langchain4j Anthropic extension — native-compatible, no dynamic proxies needed
- Qdrant Java client — gRPC-based, native-compatible with standard Quarkus gRPC config
- `Files.walk` — standard NIO, no issues
- YAML frontmatter parsing — use SnakeYAML (Quarkus native-compatible) or Jackson with YAML dataformat

Avoid:
- Reflection-based class scanning (unnecessary — we're scanning source text, not bytecode)
- Dynamic `Pattern.compile` at runtime in hot paths (compile patterns as static finals)

---

## Project Registry in Java

`registry/projects.yaml` → unchanged format, read by Jackson YAML at startup via
`@ConfigMapping` or a `@StartupEvent` loader. The Java registry service replaces
`project_registry.py`.

---

## Relationship to Retrieval Service

| | Garden Engine | Retrieval Service |
|---|---|---|
| **Purpose** | Mining, harvest, dedup | Search, RAG, MCP |
| **Mode** | CLI / batch | REST / real-time |
| **Qdrant collections** | `mining-fingerprints` (new), `garden-entries` (shared) | `garden-entries` |
| **LLM** | Claude (reasoning), Qdrant FastEmbed (embedding) | Qdrant FastEmbed only |
| **Deployment** | Same machine as garden clone | Same machine or remote |
| **Trigger** | Post-commit hook, cron, manual | HTTP request |

Both services share the `garden-entries` Qdrant collection. Retrieval service
indexes at merge time; Garden Engine reads at harvest time and writes merged entries.

---

## Open Questions

1. **Autonomous vs supervised merge**: should EntryMergeService run fully autonomously
   (write merged entry, delete originals) or produce a draft for human confirmation?
   Recommended: fully autonomous for RELATED (See Also links), human confirmation for
   DUPLICATE merges on first N runs, then promote to autonomous once quality is verified.

2. **Qdrant collection for fingerprints**: `mining-fingerprints` is separate from
   `garden-entries` because the payload schema differs (project metadata vs entry markdown).
   Confirm collection name and schema before indexing.

3. **Claude model for reasoning tasks**: `claude-sonnet-4-6` is the default.
   Haiku would be sufficient for dedup classification (cheaper); Sonnet for pattern
   naming (more creative). Consider model routing per task type.

4. **Python validator retirement**: `validate_pr.py` stays until the Java validation
   layer reaches feature parity. Track in a follow-up issue.

---

## Implementation Plan

Phase 1 — Port core pipeline (no LLM)
- `FeatureExtractor.java` (port Python, same tests as acceptance criteria)
- `ClusterPipeline.java` (interim: keep cosine on ratios, Qdrant clustering in Phase 2)
- `DeltaAnalysis.java` (port Python)
- `ProjectRegistry.java` (YAML load/save)
- CLI command: `mine --all`
- Tests: port Python test fixtures to JUnit 5

Phase 2 — Qdrant integration + LLM enrichment
- Index fingerprints into Qdrant
- `PatternNamingService` (Claude)
- `DeltaNarrativeService` (Claude)
- Replace cosine clustering with Qdrant nearest-neighbour

Phase 3 — Semantic harvest
- `SemanticDeduplicator` (Qdrant search)
- `DedupeClassifier` (Claude)
- `EntryMergeService` (Claude)
- CLI command: `harvest --sweep`
- Replace garden-agent CLAUDE.md dedup loop with `garden-engine harvest`

Phase 4 — Native image + replace Python pipeline
- GraalVM native build
- Retire Python scripts (keep as reference in `scripts/legacy/`)
- Update garden-agent-install.sh to invoke `garden-engine` binary
