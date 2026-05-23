# Hortora — Project Handoff

*Last updated: 2026-05-23 (session 15)*

---

## What Hortora Is / Local Folder Structure

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## The Three Repos — delta only

### `soredium`

*Unchanged — `git show HEAD~1:HANDOFF.md`*

### `Hortora/garden`

*Unchanged — `git show HEAD~1:HANDOFF.md`*

### `casehub/parent`

Protocol store removed. `docs/protocols/` deleted. CLAUDE.md updated to point to `casehub/garden` and explicitly says not to write protocol files here.

### `casehub/garden` — NEW

New repo (`casehubio/garden`). Always on main, no branches. Holds the full CaseHub protocol store (68 files: `docs/protocols/casehub/` + `docs/protocols/universal/`). CLAUDE.md written. GitHub: `https://github.com/casehubio/garden`. Local: `~/claude/casehub/garden/`.

All peer repo CLAUDE.md files updated: engine, clinical, devtown, platform, aml, qhorus.

Remaining peer repos not yet updated (needs coordination — each must be on main, no active session): `work`, `ledger`, `claudony`, `connectors` and their workspace CLAUDE.md files at `public/casehub/*/CLAUDE.md`.

### `hortora.github.io`

Blog entry 22 published: "A Garden for CaseHub" (2026-05-23).

### `cc-praxis` (protocol skill)

Two fixes committed and synced:
- Resolution order now checks CLAUDE.md routing row first, handles sibling `../parent/` layout, creates as last resort only.
- Commit step resolves git root from `PROTOCOLS_DIR`, not `PROJECT_ROOT` — prevents commits landing in wrong repo.

---

## Open Design Question

**`technology` field for garden entries** — proposed but not decided. Coarse `domain` field is correct as Qdrant partition key, but `tags` are empty for 51% of entries, so technology-level retrieval is broken. A best-effort `technology: quarkus` field (no certainty required, unlike `root_cause_layer`) would restore filtering without demanding reliable attribution. Pending decision before the `quarkus/` → `jvm/` YAML migration proceeds.

---

## What To Do Next

**Immediate:** Coordinate updates to `work`, `ledger`, `claudony`, `connectors` CLAUDE.md files — add `protocols → garden` routing row to each when each is on main.

**Pending:** Decide on the `technology` field, then run the `quarkus/` → `jvm/` domain YAML migration (~192 entries). Tags backfill still needed (51% empty).

**Still on the list:** 16 missing protocols in `casehub/garden/docs/protocols/PENDING-MODULE-UPDATES.md`. `quarkus/` → `jvm/` physical directory question (design says keep dirs; only the YAML field migrates). Langchain4j fork upgrade. QE run with GPU.

---

## Key ADRs / Reference Links

| Resource | Location |
|---|---|
| Blog entry 22 | `hortora.github.io/_posts/2026-05-23-mdp01-casehub-garden.md` |
| casehub/garden | `~/claude/casehub/garden/` — `https://github.com/casehubio/garden` |

*Previous refs — `git show HEAD~1:HANDOFF.md`*
