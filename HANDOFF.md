# Hortora — Project Handoff

*Last updated: 2026-06-08 (session 16 — RAG)*

---

## What Hortora Is / Local Folder Structure

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## The Repos — delta only

### `Hortora/garden-engine` — NEW

Phase 1 retrieval service is live. Quarkus 3.36.1, Java 25, GraalVM native CI.
Scans `~/.hortora/garden` on startup, embeds via Ollama, upserts into Qdrant.
Exposes `GET /search` and `garden_search` / `garden_status` MCP tools over HTTP SSE.
14/14 tests green. Workspace: `mdproctor/wsp-garden-engine` at `~/claude/public/hortora/garden-engine`.

Issue #2 closed (Qdrant domain payload pre-filter — `IsEqualTo` replaces post-retrieval Java filter).
Issue #3 open (multi-domain `IsIn` variant — deferred).
Issue #4 open (SearchResource pre-existing minor issues — `parseScore` metadata vs similarity, `searchFor` unchecked cast).

### `casehubio/onnx-inference` — DESIGN ONLY, not yet built

Spec at `docs/superpowers/specs/2026-06-03-onnx-inference-module-design.md`.
General ONNX inference wrapper (NLI, classification, regression, SPLADE, cross-encoder).
Sits below LangChain4J. CaseHub builds it; Hortora takes it as a phase 2 dependency.
Gated on: ONNX Runtime JNI + Quarkus native image prototype on macOS ARM.
LangChain4J #4994 (Qdrant hybrid search) is the upstream issue to watch.

### `casehubio/garden` — protocol migration complete

87 files migrated from `casehubio/parent/docs/protocols/` → `casehubio/garden/docs/protocols/`.
`parent/docs/protocols/` deleted. FOUNDATION-INDEX (74 rows), HARNESS-INDEX (9 rows),
universal/INDEX (4 rows) updated. Single source of truth is now casehubio/garden.

Note: universal/INDEX.md description ("staging area for Hortora") is misleading — needs
rewording. These are mandated conventions, not forage submissions.

### Other repos

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## Open Design Questions

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## What To Do Next

**Immediate:** garden-engine phase 2 design — SPLADE + cross-encoder reranker via
`casehubio/onnx-inference`. Gated on ONNX native image prototype. Don't start until
the prototype runs.

**Pending (garden-engine):**
- Federation — canonical/child chain walk (Hortora-specific, no shared module)
- Incremental re-indexing / file watching (startup-only currently)
- Issues #3, #4 (multi-domain filter, SearchResource cleanup)

**Pending (casehub):**
- ONNX native image prototype — must validate before onnx-inference module begins
- Peer repo CLAUDE.md routing updates still needed: `work`, `ledger`, `claudony`, `connectors`

---

## Key References

| Resource | Location |
|---|---|
| garden-engine spec | `Hortora/garden-engine` main |
| onnx-inference spec | `docs/superpowers/specs/2026-06-03-onnx-inference-module-design.md` |
| casehub RAG issue | `casehubio/parent#158` |
| casehub-rag issue | `casehubio/parent#164` |
| LangChain4J Qdrant hybrid | `langchain4j/langchain4j#4994` |

*Previous refs — `git show HEAD~1:HANDOFF.md`*
