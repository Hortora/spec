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

Three separate concerns, three tools:

| Concern | Tool | Why |
|---|---|---|
| Semantic similarity (dedup, cluster candidates) | **Qdrant + FastEmbed** (same as retrieval service) | Server-side, free, already deployed, no Java embedding code needed |
| Reasoning tasks at scale (pattern naming, classification, merge) | **Ollama (local)** — default | Free, unlimited, no API cost, Langchain4j first-class support |
| Quality evaluation / gold standard | **Claude Sonnet via Anthropic extension** | Used only for QE test runs, not production mining |

### Pluggable Reasoning Model

The reasoning model is config-driven. All AI service interfaces are provider-agnostic —
only the backing client changes.

```properties
# application.properties — default (free, unlimited, private)
garden.engine.reasoning.provider=ollama
quarkus.langchain4j.ollama.base-url=http://localhost:11434
quarkus.langchain4j.ollama.chat-model.model-id=qwen3.6:35b-a3b
quarkus.langchain4j.ollama.chat-model.temperature=0.2
quarkus.langchain4j.ollama.chat-model.format=json

# application-sonnet.properties — activated with -Dquarkus.profile=sonnet
garden.engine.reasoning.provider=anthropic
quarkus.langchain4j.anthropic.api-key=${ANTHROPIC_API_KEY}
quarkus.langchain4j.anthropic.model-name=claude-sonnet-4-6
quarkus.langchain4j.anthropic.max-tokens=4096
```

Switch model: `./garden-engine mine --all` (uses Ollama) vs
`./garden-engine mine --all --profile=sonnet` (uses Claude).

### Recommended Free Models (Ollama)

| Model | Active VRAM | Reasoning | Structured JSON | Notes |
|---|---|---|---|---|
| `qwen3.6:35b-a3b` | ~3B active / ~20GB | Excellent (73.4% SWE-bench) | Excellent | **Default** — MoE, fast, best quality |
| `qwen3:14b` | 9 GB | Very good | Very good | Fallback for <20GB VRAM |
| `gemma3:27b` | 17 GB | Good | Good | Google open model, alternative |
| `llama3.3:70b` | 40 GB | Excellent | Very good | Superseded by qwen3.6 at higher VRAM cost |

**Default: `qwen3.6:35b-a3b`** — 35B total params but only ~3B active (MoE architecture).
Stronger reasoning than Llama3.3:70b, runs on consumer hardware, Apache 2.0.

**Why not Qwen3.6-Plus via OpenRouter free tier?**
The free tier collects prompts for Alibaba training data. Garden entries are your
curated knowledge — self-hosting keeps them private. Use OpenRouter only for
non-sensitive QE experiments, never for production mining.

The Java service never computes embeddings. It passes text to Qdrant and Qdrant
does all vector work — identical to the retrieval service.

---

## Quality Evaluation (QE) Framework

Running mining at scale with a free model creates a risk of degraded output
quality vs Claude Sonnet. The QE framework measures this gap systematically.

### How it works

```
garden-engine qe --sample=50 --tasks=all --compare=sonnet
```

1. **Sample** — randomly selects N entries/clusters from the garden or last mining run
2. **Run both** — each task (pattern naming, dedup classification, merge) runs through:
   - Free model (Ollama default)
   - Claude Sonnet (evaluator and gold standard)
3. **Score** — Sonnet acts as judge: given both outputs, which is more accurate/useful?
4. **Report** — per-task agreement rate, cases where models diverge, cost estimate

### QE AI Services

```java
@RegisterAiService(modelName = "anthropic")  // always Sonnet, never Ollama
public interface QualityEvaluator {

    @SystemMessage("""
        You are evaluating two AI outputs for the same knowledge garden task.
        Score each output 1-5 for: accuracy, completeness, and usefulness.
        Identify which output is better and explain why, or mark as equivalent.
        Respond as JSON: {
          free_score: {accuracy, completeness, usefulness},
          sonnet_score: {accuracy, completeness, usefulness},
          winner: "free"|"sonnet"|"equivalent",
          reasoning: "..."
        }
        """)
    QualityScore evaluate(@UserMessage String taskAndBothOutputs);
}
```

### QE Report Format

```
Garden Engine QE Report — 2026-04-22
Tasks evaluated: pattern_naming (20), dedup_classification (20), entry_merge (10)
Free model: llama3.3:70b via Ollama

Pattern naming:
  Agreement with Sonnet: 17/20 (85%)
  Free model superior: 0
  Equivalent: 14
  Sonnet superior: 3  ← review these manually

Dedup classification:
  Agreement: 19/20 (95%)
  Sonnet superior: 1  ← false DISTINCT on semantically identical entries

Entry merge:
  Agreement: 7/10 (70%)
  Sonnet superior: 3  ← merge synthesis less complete with Ollama

Overall: 85% agreement. Recommended: promote Ollama to production for
dedup_classification and pattern_naming. Continue using Sonnet for entry_merge
until agreement reaches 85%.
```

### Promotion threshold

- ≥ 90% agreement → Ollama is production-ready for that task
- 80–90% → manually review divergent cases, re-run QE after prompt tuning
- < 80% → keep Sonnet for that task, file a prompt improvement issue

### Cost accounting

The QE command also reports the Sonnet cost for the sample run, so you can
estimate: "running the full garden through Sonnet would cost $X — Ollama saves $Y/month."

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
   DUPLICATE merges until QE shows ≥ 85% agreement, then promote to autonomous.

2. **Qdrant collection for fingerprints**: `mining-fingerprints` is separate from
   `garden-entries` because the payload schema differs (project metadata vs entry markdown).
   Confirm collection name and schema before indexing.

3. **Model routing per task**: QE results may show that `gemma3:27b` is sufficient
   for dedup classification (cheaper, faster) while `llama3.3:70b` is needed for
   merge synthesis. Add per-task model config once QE data exists.

4. **Python validator retirement**: `validate_pr.py` stays until the Java validation
   layer reaches feature parity. Track in a follow-up issue.

5. **GPU availability**: `qwen3.6:35b-a3b` needs ~20 GB VRAM (only 3B active params via MoE).
   `qwen3:14b` is the fallback for <20 GB. Record which model is running in the QE
   report so results are comparable across machines.

---

## Implementation Plan

Phase 1 — Port core pipeline (no LLM)
- `FeatureExtractor.java` (port Python, same tests as acceptance criteria)
- `ClusterPipeline.java` (interim: keep cosine on ratios, Qdrant clustering in Phase 2)
- `DeltaAnalysis.java` (port Python)
- `ProjectRegistry.java` (YAML load/save)
- CLI command: `mine --all`
- Tests: port Python test fixtures to JUnit 5

Phase 2 — Ollama integration + LLM enrichment (free model first)
- Wire Langchain4j Ollama extension (`llama3.3:70b` default)
- `PatternNamingService` backed by Ollama
- `DeltaNarrativeService` backed by Ollama
- Index fingerprints into Qdrant, replace cosine clustering with Qdrant nearest-neighbour
- Add `--profile=sonnet` switch for Anthropic backend

Phase 3 — QE framework
- `QualityEvaluator` AI service (always Anthropic/Sonnet)
- CLI command: `qe --sample=N --tasks=all --compare=sonnet`
- QE report output
- Baseline QE run: establish agreement rates for all three task types
- Tune Ollama prompts until target thresholds met

Phase 4 — Semantic harvest
- `SemanticDeduplicator` (Qdrant search)
- `DedupeClassifier` (Ollama, validated by QE)
- `EntryMergeService` (Ollama or Sonnet depending on QE threshold)
- CLI command: `harvest --sweep`
- Replace garden-agent CLAUDE.md dedup loop with `garden-engine harvest`

Phase 5 — Native image + replace Python pipeline
- GraalVM native build
- Retire Python scripts (keep as reference in `scripts/legacy/`)
- Update garden-agent-install.sh to invoke `garden-engine` binary
