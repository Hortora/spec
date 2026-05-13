# Hortora — Project Handoff

*Last updated: 2026-05-13 (session 11)*

---

## What Hortora Is / Local Folder Structure

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## The Three Repos — delta only

### `soredium`

- Issue #44 opened: add `type: convention` + `variant:` field to garden schema (`validate_pr.py`, `submission-formats.md`, forage skill). Convention entries blocked on this.

### `Hortora/garden`

**Audit 2 — 22 entries added to `jvm/` + `tools/`:**
All universal Quarkus/Java gotchas and techniques migrated from casehub protocols. Commit `4ced46b`.

**Audit 1 — 33 entries reclassified:**
Moved from `quarkus/` and `jvm/` to casehub domain directories. Commit `5996b15`.
- `casehub-engine/` — 15 entries (CaseContextImpl, engine SPI, persistence-memory)
- `casehub-work/` — 10 entries (WorkItem lifecycle, WorkerCandidate, quarkus-work API)
- `casehub-ledger/` — 4 entries (LedgerAttestation, sequence_number)
- `casehub-qhorus/` — 4 entries (new domain; EVENT content, check_messages, reactive activation)

8 duplicates discarded (superseded by new jvm/ entries). GARDEN.md updated with all new sections.

2 new tools entries: `GE-20260513-176ca1` (git mv missing target dir gotcha), `GE-20260513-01e602` (git show recovery technique).

### `casehub/parent`

- `docs/protocols/INDEX.md` rebuilt: 15 casehub-specific protocols, Garden References section (22 GE-IDs), Casehub Domain Entries section (33 GE-IDs grouped by domain).

### `hortora.github.io`

- Blog entry 18 added: "Three Kinds of Knowledge" (2026-05-13) — garden taxonomy design, audit results.

---

## Garden Schema Decision — `type: convention`

**Settled this session.** Three-way taxonomy:
- **Universal** — always true, no alternatives (`gotcha`, `technique`, `undocumented`)
- **Convention** — style choice, alternatives exist → new `type: convention`
- **Casehub-specific** — stays in protocols or casehub-* garden domains

**`variant:` field** — when two entries share a `title:`, each adds `variant:` with alternative-specific descriptor. GARDEN.md groups them under the shared title. Single entries omit `variant:`.

**6 convention entries pending schema change:** module-tier-structure, maven-submodule-naming, maven-module-scoping, quartz-ram-store, optional-module-pattern, spi-blocking-reactive-parity.

---

## What To Do Next

**Immediate: implement soredium#44** — `type: convention` + `variant:` in `validate_pr.py` and `submission-formats.md`. Then write the 6 pending convention entries.

**Then: Audit 3** — folder structure validity (does every entry have a natural home?).
**Then: Audit 4** — `repos/` depth check (trim class-level detail, add references).

**Still pending (unchanged):** Langchain4j fork upgrade. QE run with GPU.

---

## Key ADRs / Reference Links

| Resource | Location |
|---|---|
| Methodology design doc | `spec/docs/design/2026-05-05-hortora-project-knowledge-methodology.md` |
| Convention schema issue | Hortora/soredium#44 |
| Blog entry 18 | `hortora.github.io/_posts/2026-05-13-mdp01-garden-audit-taxonomy.md` |

*Previous ADRs/refs — `git show HEAD~1:HANDOFF.md`*
