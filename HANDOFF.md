# Hortora — Project Handoff

*Last updated: 2026-05-12 (session 10)*

---

## What Hortora Is / Local Folder Structure

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## The Three Repos — delta only

### `soredium`

- `protocol/SKILL.md` added — CAPTURE, SWEEP, SEARCH, HEALTH, DEEP-SCAN (stub). Generic tool; no casehub content.
- `README.md` updated with protocol skill entry

### `Hortora/garden`

- GE-20260415-3ce5f3 deleted — quarkus-junit direction was inverted; GARDEN.md index cleaned
- 3 new entries (2026-05-12): `tools/GE-20260512-0dc5df` (sed silently empties file on macOS), `jvm/GE-20260512-47f92e` (quarkus-junit5 relocation stub since 3.31), `tools/GE-20260512-deda31` (Python replace over sed for safe multi-file edits)

### `hortora.github.io`

- Blog entry 17 added: "Protocol Layer and Coherence" (2026-05-12)

---

## New This Session — Protocol Layer + Methodology Refinement

**Epic #40 closed.** Three priorities shipped:

1. **33 casehub conventions → `protocols/`** — YAML frontmatter added (PP-YYYYMMDD-xxxxxx IDs, full schema), directory renamed, CLAUDE.md and PLATFORM.md updated. `casehub/parent` commit `080f53f`.
2. **`protocol` skill** — soredium commit `f26ee0d` → `d284b38` (stripped casehub-specific posture after it moved to java-dev)
3. **Handover wrap** — protocol SWEEP now default-on alongside forage. cc-praxis commit `6effd7a`.

**Definition of protocols settled:**
Protocols are about platform coherence and consistency — standing architectural rules that answer "how do we do this consistently?" not "how do I avoid this gotcha?" ~9 of the 33 are genuine protocols; ~24 are Quarkus/Java gotchas that belong in the garden. Audit 2 is half-done but entries not yet moved.

**java-dev updated (cc-praxis):**
- "Minimize changes" → "Keep commits focused" (commit discipline, not code avoidance)
- Multi-level consolidation added (class → module → repo)
- "Clean APIs and abstractions" section added

**config-architecture.md split:**
- `cc-praxis/docs/config-architecture.md` → generic, no casehub content
- `casehub/parent/docs/config-architecture.md` → casehub-specific (Platform coherence, protocols dir, Known Duplications)
- `update-claude-md` Step 0 now reads `**Config architecture:**` URL from CLAUDE.md, falls back to cc-praxis generic

**Child repo CLAUDE.mds fixed:**
All 7 child repos (connectors, claudony, devtown, aml, qhorus, work, clinical): absolute `/Users/mdproctor/...` paths → `~/claude/casehub/parent/docs/` with skip-if-absent note.

---

## What To Do Next

**Immediate: complete Audit 2** — go through the remaining ~24 generic Quarkus/Java protocols and move them to the canonical garden as discovery entries. Start with the clearest ones (cdi-*, quarkus-test-*, panache-*).

**Then Audit 1** — ~70 canonical `quarkus/` garden entries referencing casehub concepts → classify as universal or casehub-specific.

**Still pending (unchanged):** Audits 3 and 4 (folder structure validation, repos/ depth check). Langchain4j fork upgrade. QE run with GPU.

---

## Key ADRs / Reference Links

| Resource | Location |
|----------|----------|
| Methodology design doc | `spec/docs/design/2026-05-05-hortora-project-knowledge-methodology.md` |
| Protocol skill | `soredium/protocol/SKILL.md` |
| casehub config-architecture | `casehub/parent/docs/config-architecture.md` |
| Blog entry 17 | `hortora.github.io/_posts/2026-05-12-mdp01-protocol-layer-coherence.md` |

*Previous ADRs/refs — `git show HEAD~1:HANDOFF.md`*
