# Hortora — Project Handoff

*Last updated: 2026-04-18 (session 2)*

---

## What Hortora Is

*Unchanged — `git show HEAD~2:HANDOFF.md`*

## Local Folder Structure

*Unchanged — `git show HEAD~2:HANDOFF.md`*

---

## The Three Repos — delta only

### `soredium` — major changes this session

**Area 2 Phase 1 shipped** (epic [#31](https://github.com/Hortora/soredium/issues/31), closes [#32](https://github.com/Hortora/soredium/issues/32)):

- `GARDEN_TYPES` registry + `GARDEN_DEFAULT` in `validate_pr.py` — 6 garden types with valid entry types, required fields, staleness defaults
- `validate()` now validates `garden` field, rejects types invalid for the garden, enforces garden-specific required fields (`changed_in` for evolution, `severity` for risk)
- Backward compatible: absent `garden` defaults to `discovery`
- `forage/submission-formats.md` — templates for all 6 garden types with editorial bars
- `forage/SKILL.md` — Step 4 now guides garden + domain selection with editorial bar table
- `tests/test_garden_types.py` — 25 tests (unit, correctness, happy path, CLI integration)
- 722 tests passing on main, pushed

Everything before this session in soredium — *unchanged, `git show HEAD~2:HANDOFF.md`*

### `hortora.github.io`

Blog entries 09 + 10 committed:
- 09: "Batching the Sweep" (`2026-04-18-forage-sweep-batching.md`)
- 10: "Code Like Sanne" (`2026-04-18-code-like-sanne.md`)

### `Hortora/garden`

Open PRs:
- #58–60 — from Apr 15 forage sweep (still open)
- #78 — 2 new entries from today's sweep: `str.replace list[0] gotcha`, `observed_at/indexed_at technique`

### `spec` — significant changes this session

- Area 2 taxonomy design locked: `docs/superpowers/specs/2026-04-18-area2-taxonomy-design.md`
- Knowledge dimensions session notes updated with 6-garden revision + patterns-garden extension + developer profiles + trend data design
- Phase 1 implementation plan: `docs/superpowers/plans/2026-04-18-area2-phase1-core-taxonomy.md`

---

## What To Do Next

**Immediate:** Merge garden PRs #58–60 and #78.

**Next build session: Area 2 Phase 2** — patterns-garden extended entry format:
- New YAML fields: `observed_in`, `suitability`, `variants`, `variant_frequency`, `authors`, `stability`
- Validator support for patterns-garden extended fields
- `submission-formats.md` patterns template update
- Tests for all new fields

**Spec reference:** `docs/superpowers/specs/2026-04-18-area2-taxonomy-design.md` — Patterns-Garden Extended Model section

**Canonical gardens still not created** — `jvm-garden`/`tools-garden` concept is now superseded by knowledge-type-first naming; create `discovery-garden` and `patterns-garden` canonical repos via `init_garden.py`. Blocked on: does the naming change affect `init_garden.py`?

---

## Key ADRs / Reference Links

| Resource | Location |
|----------|----------|
| Area 2 taxonomy spec | `spec/docs/superpowers/specs/2026-04-18-area2-taxonomy-design.md` |
| Area 2 Phase 1 plan | `spec/docs/superpowers/plans/2026-04-18-area2-phase1-core-taxonomy.md` |
| Knowledge dimensions (session notes) | `spec/docs/design/2026-04-16-knowledge-dimensions.md` |
| Ecosystem plan (7 areas) | `spec/docs/design/2026-04-16-garden-ecosystem-plan.md` |
| Blog entry 10 | `hortora.github.io/_posts/2026-04-18-code-like-sanne.md` |

*Previous references (ADR-0004, MCP server, garden health) — `git show HEAD~2:HANDOFF.md`*
