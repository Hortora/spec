# Hortora — Project Handoff

*Last updated: 2026-06-09 (session 19 — neural-text gate confirmed passed)*

---

## What Hortora Is / Local Folder Structure

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## The Repos — delta only

### CaseHub peer repos — protocol routing complete

*Unchanged — `git show HEAD~1:HANDOFF.md`*

### `Hortora/garden-engine` — issues #3 and #4 closed

*Unchanged — `git show HEAD~2:HANDOFF.md`*

### `casehubio/neural-text` — C1–C7 complete, all journeys shipped

The ONNX native image prototype (C2) passed. All seven chapters shipped:
`inference-api`, `inference-runtime`, `inference-inmem`, `inference-tasks`,
`inference-splade`, `inference-quarkus`, `rag-api`, `rag`, `rag-tika`, `rag-testing`.

Hortora consumers: `casehub-inference-splade` (`SparseEmbedder`) and
`casehub-inference-tasks` (`CrossEncoderReranker`) are published and ready.
Epic #7 closed. Tracking: casehubio/parent#158, casehubio/parent#164.

### Other repos

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## Open Design Questions

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## What To Do Next

**Immediate:** garden-engine phase 2 design — SPLADE hybrid search + cross-encoder
reranker. Gate is passed. Dependencies published:
- `casehub-inference-splade` → `SparseEmbedder` for Qdrant named vector space upsert
- `casehub-inference-tasks` → `CrossEncoderReranker` for precision-mode top-N

Run `/work` against a new garden-engine issue for phase 2. Design first (brainstorm),
then TDD.

**Pending (garden-engine):**
- Federation — canonical/child chain walk (Hortora-specific, no shared module)
- Incremental re-indexing / file watching (startup-only currently)

**Monitoring:**
- Protocol routing commits on 3 active CaseHub branches (connectors, ledger, work) — no action until branches merge or are abandoned. (`neural-text` done — removed.)

---

## Key References

*Unchanged — `git show HEAD~1:HANDOFF.md`*
