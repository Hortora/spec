# Hortora — Project Handoff

*Last updated: 2026-05-07 (session 9)*

---

## What Hortora Is / Local Folder Structure

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## The Three Repos — delta only

### `soredium`

*Unchanged — `git show HEAD~1:HANDOFF.md`*

### `Hortora/garden`

*Unchanged — `git show HEAD~1:HANDOFF.md`*

### `hortora.github.io`

- Blog entry 16 added: "Hortora as a Knowledge Methodology" (2026-05-07)

---

## New This Session — Methodology Design

**Design doc committed:** `docs/design/2026-05-05-hortora-project-knowledge-methodology.md`

Hortora as a full project knowledge methodology — not just discovery gardens.
Three layers: discovery gardens, platform protocols, ADRs. All RAG-optimal, all
hierarchical. Key decisions recorded in the doc:

- Protocol entry format: YAML frontmatter (id, title, type, severity, refs, violation_hint)
- `protocol` skill: CAPTURE, SWEEP, SEARCH, HEALTH, DEEP-SCAN
- DEEP-SCAN: gap finding + relevance check + violation detection
- PROTOCOL-REFS.yaml for cross-repo link integrity
- Casehub garden concept: `~/.hortora/casehub/` as project-scoped child garden
- PLATFORM.md / APPLICATIONS.md split (done — committed to casehub/parent `6cc21f0`)

**Four audits required before spec can be written:**
1. ~70 canonical quarkus/ garden entries referencing casehub concepts → classify
2. 33 casehub conventions → universal gotchas vs casehub protocols (known contradiction: quarkus-junit)
3. Folder structure validation against real entries
4. repos/ depth check (9 files — trim class-level detail)

**Immediate priorities (no audit needed):**
1. Migrate 33 conventions to YAML frontmatter format, rename `conventions/` → `protocols/`
2. Create `protocol` skill in soredium
3. Hook protocol SWEEP into handover wrap alongside forage

---

## What To Do Next (carry-forward)

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## Key ADRs / Reference Links

*Unchanged — `git show HEAD~1:HANDOFF.md`*
