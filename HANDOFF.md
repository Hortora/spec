# Hortora — Project Handoff

*Last updated: 2026-04-14*

---

## What Hortora Is

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Local Folder Structure

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## The Three Repos

### `hortora.github.io` — significant updates ✅

- `docs/architecture.html` — new page for platform architects: 6 sections (knowledge decay, submission integrity, storage reliability, retrieval accuracy, federation, deduplication), each with property claims, guarantees, and graceful degradation callouts
- Sidebar nav updated in all 4 existing docs pages to include Architecture
- `_posts/2026-04-14-five-layers.md` — blog entry 06: staleness hybrid + architecture page

### `spec` — significant updates ✅

- `docs/superpowers/specs/2026-04-14-architecture-page-design.md` — design spec for the architecture page (6 sections including deduplication)
- `docs/superpowers/plans/2026-04-14-architecture-page.md` — implementation plan (executed)
- `docs/superpowers/plans/2026-04-14-staleness-hybrid-enforcement.md` — staleness plan (executed)

### `soredium` — significant updates ✅

**Phase 3 staleness hybrid enforcement — fully implemented:**

- `scripts/validate_garden.py` — `--freshness` flag: scans all entries, reports overdue count, exits 2 if any found. CRLF-normalises before regex, try/except around fromisoformat.
- `tests/test_validate_garden.py` — `TestFreshnessFlag` class, 6 tests, all passing
- `forage/submission-formats.md` — documented `verified_on`, `last_reviewed` optional fields and `### Why this fix` body section
- `forage/SKILL.md` — 3 changes:
  - SEARCH: S2 staleness annotation (age shown, ⚠️ if past threshold, uses `verified_on` for version gap)
  - CAPTURE: S6 `verified_on` prompt (library/tool entries) + S3 rationale soft prompt (≥12 score)
  - SWEEP: S1 domain-filtered staleness spot-check (Step 5, before Report which became Step 6)
- `harvest/SKILL.md` — REVIEW mode added (6-step systematic sweep, updates `Last staleness review` in GARDEN.md)

**Installed skills synced:** forage and harvest updated on this machine.

### `Hortora/garden` — 3 PRs open

- #40 — `GE-20260414-c12931`: YAML frontmatter regex silently skips CRLF files (gotcha, score 12)
- #41 — `GE-20260414-2a1cd1`: regex date validation insufficient for calendar values (gotcha, score 11)
- #42 — `GE-20260414-3d73c3`: property-claims structure for architecture docs (technique, score 14)

---

## What To Do Next

**Immediate:** Review and merge garden PRs #40–42.

**Phase 4 (next major work):** Full deduplication system — L2/L3 Jaccard sweep, DEDUPE drift counter in harvest.

**Phase 5:** Ecosystem foundation — GitHub org creation, `garden-spec` published, domain gardens launched.

### Standing maintenance

- **DEDUPE** when `Entries merged since last sweep` hits 10
- **harvest REVIEW** when stale entries accumulate (run `validate_garden.py --freshness ~/.hortora/garden` to check)

---

## Reference Links

| Resource | Location |
|----------|----------|
| Staleness plan | `spec/docs/superpowers/plans/2026-04-14-staleness-hybrid-enforcement.md` |
| Architecture page spec | `spec/docs/superpowers/specs/2026-04-14-architecture-page-design.md` |
| Latest design snapshot | `spec/snapshots/2026-04-13-garden-entry-format.md` |
| ADR-0003 (GE-ID scheme) | `spec/docs/adr/0003-ge-id-scheme-date-plus-random-hex.md` |
| Blog entry 06 | `hortora.github.io/_posts/2026-04-14-five-layers.md` |
| forage skill | `~/.claude/skills/forage/SKILL.md` |
| harvest skill | `~/.claude/skills/harvest/SKILL.md` |
| validate_garden.py | `~/claude/hortora/soredium/scripts/validate_garden.py` |
