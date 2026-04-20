# Hortora — Project Handoff

*Last updated: 2026-04-18 (session 3)*

---

## What Hortora Is

*Unchanged — `git show HEAD~3:HANDOFF.md`*

## Local Folder Structure

*Unchanged — `git show HEAD~3:HANDOFF.md`*

---

## The Three Repos — delta only

### `soredium` — Area 2 Phase 2 shipped this session

**Closes [#33](https://github.com/Hortora/soredium/issues/33) (epic #31):**

- `validate_patterns_extended(fm)` in `validate_pr.py` — validates 6 optional patterns-garden fields
- `VALID_AUTHOR_ROLES = {'originator', 'adopter', 'innovator'}`, `VALID_STABILITY = {'low', 'medium', 'high'}`
- Malformed extended fields → WARNINGs (not CRITICALs); entry still accepted
- Non-patterns gardens completely unaffected (isolation tested)
- `forage/submission-formats.md` patterns template updated with all extended fields
- `forage/SKILL.md` Step 3 extended with patterns-garden optional field extraction table
- 747 tests passing on main, pushed

Phase 1 details — *unchanged, `git show HEAD~1:HANDOFF.md`*

### `hortora.github.io`

Blog entry 11: "Schema, Done Right" (`2026-04-18-schema-done-right.md`) — committed.

Previous entries (09, 10) — *unchanged, `git show HEAD~1:HANDOFF.md`*

### `Hortora/garden`

*Unchanged — `git show HEAD~1:HANDOFF.md`* (PRs #58–60 and #78 still open)

### `spec`

Phase 2 plan committed: `docs/superpowers/plans/2026-04-18-area2-phase2-patterns-extended.md`

---

## What To Do Next

**Immediate:** Merge garden PRs #58–60 and #78.

**Next build session: Area 2 Phase 3** — project registry + ecosystem mining pipeline:
- Project registry schema + tooling (curated index of monitored projects with `last_processed_commit`)
- Structural clustering pipeline (extract features, cluster, surface candidate patterns)
- Delta analysis (monitor major version changes for pattern origin stories)
- These are new infrastructure, not just validator extensions

**Canonical garden naming** — `init_garden.py` may need updating for knowledge-type-first names (`discovery-garden`, `patterns-garden`) vs the old technology-domain names. Check before creating canonical repos.

---

## Key ADRs / Reference Links

| Resource | Location |
|----------|----------|
| Area 2 taxonomy spec | `spec/docs/superpowers/specs/2026-04-18-area2-taxonomy-design.md` |
| Area 2 Phase 1 plan | `spec/docs/superpowers/plans/2026-04-18-area2-phase1-core-taxonomy.md` |
| Area 2 Phase 2 plan | `spec/docs/superpowers/plans/2026-04-18-area2-phase2-patterns-extended.md` |
| Ecosystem plan (7 areas) | `spec/docs/design/2026-04-16-garden-ecosystem-plan.md` |
| Blog entry 11 | `hortora.github.io/_posts/2026-04-18-schema-done-right.md` |

*Previous references — `git show HEAD~1:HANDOFF.md`*
