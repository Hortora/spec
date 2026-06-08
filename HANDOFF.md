# Hortora — Project Handoff

*Last updated: 2026-06-08 (session 18 — garden-engine backlog clearance)*

---

## What Hortora Is / Local Folder Structure

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## The Repos — delta only

### CaseHub peer repos — protocol routing complete

*Unchanged — `git show HEAD~1:HANDOFF.md`*

### `Hortora/garden-engine` — issues #3 and #4 closed

Multi-domain filter (`?domain=jvm&domain=tools`) shipped via `IsIn` filter.
`SearchResource` refactored: `doSearch()` extracted, unchecked cast removed,
`parseCurationScore` renamed. Branch `issue-3-multi-domain-filter` merged to main.

### `casehubio/garden` universal/INDEX.md

"Staging area" → "mandated conventions". Committed and pushed to main.

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

**Monitoring:**
- Protocol routing commits on 4 active CaseHub branches (connectors, ledger, work, neural-text) — no action until branches merge or are abandoned

---

## Key References

*Unchanged — `git show HEAD~1:HANDOFF.md`*
