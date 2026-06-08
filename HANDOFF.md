# Hortora — Project Handoff

*Last updated: 2026-06-08 (session 17 — casehub protocol routing audit)*

---

## What Hortora Is / Local Folder Structure

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## The Repos — delta only

### CaseHub peer repos — protocol routing complete

All 15 CaseHub workspace/project CLAUDE.mds now have `## Reference Documents (casehub-parent)`
pointing to the garden protocol indexes. This session closed the backlog item.

Index used per repo type:
- **FOUNDATION-INDEX:** claudony, connectors, ledger, work, eidos
- **HARNESS-INDEX + universal/INDEX:** drafthouse
- **universal/INDEX only:** neural-text

5 of the 7 commits landed on active feature branches (not main):
- `connectors` → issue-16-mcp-slack-bot-tools
- `ledger` → issue-122-postgresql-devservices
- `work` → issue-256-multi-tenancy-tenantid
- `neural-text` → issue-5-inference-quarkus-batch
- `eidos` → committed to main (squash branch had already merged)

Changes will reach main when those branches merge. No follow-up needed unless
a branch is abandoned without merging.

### `Hortora/garden-engine`

*Unchanged — `git show HEAD~1:HANDOFF.md`*

### `casehubio/onnx-inference` — DESIGN ONLY, not yet built

*Unchanged — `git show HEAD~1:HANDOFF.md`*

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
- universal/INDEX.md description reword in casehub/garden ("staging area" → "mandated conventions")

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
