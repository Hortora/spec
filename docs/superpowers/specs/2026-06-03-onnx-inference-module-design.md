# ONNX Inference Module — Design Spec

**Date:** 2026-06-03  
**Status:** Design approved — pending prototype validation  
**Repo:** `casehubio/onnx-inference`  
**Consumers:** casehub (all tiers), Hortora  
**Tracking:** casehubio/parent#158, Hortora/spec#15

---

## What This Is

A standalone, general-purpose ONNX inference module for JVM projects. Not a RAG framework. Not a wrapper around LangChain4J. A lower-level primitive that fills the gap LangChain4J leaves: general ONNX inference for NLI, classification, regression, sparse embeddings, and cross-encoder reranking.

LangChain4J covers the RAG pipeline (document loading, chunking, dense embeddings, vector stores, retrieval, context assembly, MCP). This module covers what LangChain4J does not: running arbitrary ONNX models for inference tasks that don't fit the embedding mold.

---

## What This Is Not

- Not a RAG library — no document ingestion, chunking, vector store, or retrieval pipeline
- Not a replacement for `LangChain4J OnnxEmbeddingModel` — dense float-vector embeddings stay in LangChain4J
- Not a Hortora-specific or CaseHub-specific module — zero domain dependencies enforced by ArchUnit from day one

---

## Consumer Use Cases

### CaseHub

| Use Case | Model Type | Notes |
|----------|-----------|-------|
| Hallucination detection | NLI | Scores LLM output faithfulness against input facts |
| Action risk classification | Classifier | Per-deployment ONNX model; replaces always-AUTONOMOUS stub |
| Epistemic confidence estimation | Regression | Dynamic confidence from agent output; feeds `CapabilityHealth.probe()` |
| Policy matching (regulatory) | SPLADE sparse | Interpretable term matching; required for compliance audit |
| SAR typology / clinical protocol matching | SPLADE sparse | Lexical precision for regulated domains |

### Hortora

| Use Case | Model Type | Notes |
|----------|-----------|-------|
| Precision mode reranker | Cross-encoder | Human-facing UI; top-3 from top-20 candidates |
| SPLADE sparse embeddings | SPLADE | Hybrid search with Qdrant; contingent on native image validation |

---

## Module Structure

```
inference-api/      — zero deps: SPIs and domain types
inference-runtime/  — ONNX Runtime JVM + HuggingFace Tokenizers; tokenization and session management
inference-splade/   — sparse embedding output (Map<Integer, Float>); log-saturation + thresholding
inference-tasks/    — typed task adapters: NliClassifier, TextClassifier, ScalarRegressor, CrossEncoderReranker
inference-quarkus/  — CDI wiring, @InferenceModel qualifier, Dev Services, @QuarkusTest support
inference-inmem/    — deterministic stub InferenceModel implementations for unit testing (no JNI)
```

ArchUnit rule enforced from day one: all modules except `inference-quarkus` have zero dependencies on any casehub artifact, Quarkus, or Spring. `inference-runtime`, `inference-tasks`, and `inference-splade` additionally have no LangChain4J dependency — they sit below it.

---

## Core API (inference-api — zero deps)

```java
interface InferenceModel extends AutoCloseable {
    InferenceOutput run(InferenceInput input);
    List<InferenceOutput> runBatch(List<InferenceInput> inputs);
}

// Text only — tokenization is the runtime's responsibility
record InferenceInput(List<String> texts) {
    static InferenceInput of(String text) { ... }
    static InferenceInput pair(String a, String b) { ... }  // cross-encoder: query + document
}

// Raw tensor access — task adapters interpret; callers never touch tensor names
interface InferenceOutput {
    float[] tensor(String outputName);
    float[][] batchTensors(String outputName);
}

record ModelConfig(
    Path modelPath,        // .onnx file
    Path tokenizerPath,    // tokenizer.json (HuggingFace format)
    int maxSequenceLength  // default 512
) {}
```

---

## Task Adapters (inference-tasks)

Typed wrappers that know which output tensor names to read and how to post-process them. Callers work with domain types, not tensors.

```java
// Hallucination detection
class NliClassifier {
    NliResult classify(String premise, String hypothesis);
    // NliResult: { entailment: float, neutral: float, contradiction: float }
}

// Action risk, multi-class
class TextClassifier {
    ClassificationResult classify(String text);
    // ClassificationResult: { label: String, confidence: float, allScores: Map<String, Float> }
}

// Epistemic confidence — scalar output
class ScalarRegressor {
    float predict(String text);
}

// Precision-mode retrieval reranking
class CrossEncoderReranker {
    List<RankedResult> rerank(String query, List<String> candidates);
    // RankedResult: { text: String, score: float, originalIndex: int }
}
```

Each adapter wraps any `InferenceModel` — unit tests inject `inference-inmem` stubs without touching ONNX Runtime.

---

## Sparse Embeddings (inference-splade)

```java
class SparseEmbedder {
    Map<Integer, Float> embed(String text);
    List<Map<Integer, Float>> embedBatch(List<String> texts);
}
```

Post-processing: log-saturation (`log(1 + relu(weight))`) and threshold filtering (drop weights below 0.01) applied to raw SPLADE model output. Output is a sparse term weight map suitable for Qdrant named vector space upsert.

---

## Data Flow

```
Caller text
  → InferenceInput.of("text") or .pair("a", "b")
  → inference-runtime: HuggingFace Tokenizers JNI → token IDs + attention mask
  → inference-runtime: OrtSession.run(inputTensors) → raw float tensors
  → InferenceOutput
  → Task adapter → typed result
```

---

## Threading and Model Lifecycle

`OrtSession` is thread-safe for concurrent inference. Model loading (file I/O + native heap) is expensive — one session per `ModelConfig`, loaded once at startup and held for the process lifetime. In `inference-quarkus`, each configured model is an `@ApplicationScoped` CDI bean.

Multiple models registered via config:

```properties
inference.models.nli.model-path=/models/nli.onnx
inference.models.nli.tokenizer-path=/models/tokenizer.json
inference.models.nli.max-sequence-length=512

inference.models.risk.model-path=/models/risk-classifier.onnx
inference.models.risk.tokenizer-path=/models/tokenizer.json
```

Injected with `@InferenceModel("nli")` or `@InferenceModel("risk")`.

---

## Testing

`inference-inmem` provides deterministic stub `InferenceModel` implementations: fixed outputs, no JNI, safe in native image and any test context. All task adapters accept any `InferenceModel` — unit tests inject stubs. ONNX Runtime is never needed in test scope.

---

## Native Image Gate — Hard Prerequisite

Two JNI layers must both work in Quarkus native image on macOS ARM:

1. ONNX Runtime (`com.microsoft.onnxruntime`) — native lib bundling + JNI config JSON
2. HuggingFace Tokenizers JNI — resource config + native lib

**The prototype is the first deliverable.** It must demonstrate a real ONNX model running inference end-to-end in a Quarkus native image binary on macOS ARM before `inference-quarkus` is built.

If the prototype fails:
- `inference-api`, `inference-runtime`, `inference-tasks`, `inference-splade` remain JVM-only artifacts — fully usable
- `inference-quarkus` operates in JVM mode only (no native image compilation)
- Hortora's distributable native binary goal is revisited: options include JVM mode with CDS, or separating JNI components into a sidecar process

Neither CaseHub nor Hortora commits to native image deployment until the prototype confirms viability.

---

## Relationship to LangChain4J

This module sits below LangChain4J, not beside it. LangChain4J's `OnnxEmbeddingModel` (dense float-vector embeddings) is not replaced — it continues to handle dense embeddings for both projects. This module handles the cases LangChain4J does not:

| Capability | Where it lives |
|---|---|
| Dense float-vector embeddings | LangChain4J `OnnxEmbeddingModel` |
| Sparse embeddings (SPLADE) | `inference-splade` (this module) |
| NLI, classification, regression | `inference-tasks` (this module) |
| Cross-encoder reranking | `inference-tasks` (this module) |
| RAG pipeline, vector stores, MCP | LangChain4J |

The SPLADE implementation in `inference-splade` is a candidate for upstream contribution to LangChain4J (issue #1600) once stable.

---

## Sequencing

1. **Prototype** — ONNX Runtime JNI + HuggingFace Tokenizers JNI in Quarkus native image on macOS ARM. Pass/fail gates everything else.
2. **`inference-api` + `inference-runtime` + `inference-inmem`** — core module, no framework coupling, usable immediately in JVM mode
3. **`inference-tasks`** — NliClassifier, TextClassifier, ScalarRegressor, CrossEncoderReranker
4. **`inference-quarkus`** — CDI wiring; native image mode conditional on prototype outcome
5. **`inference-splade`** — after ONNX native image validated; contributes to LangChain4J #1600 when stable
