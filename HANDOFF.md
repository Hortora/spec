# Hortora — Project Handoff

*Last updated: 2026-04-15*

---

## What Hortora Is

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Local Folder Structure

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## The Three Repos — significant updates ✅

### `soredium` — major changes this session

**SQLite aggregate state (ADR-0004) — fully implemented:**
- `scripts/garden_db.py` — WAL-mode SQLite: checked_pairs, discarded_entries, entries_index, schema_version
- `scripts/garden_db_migrate.py` — one-time migration from CHECKED.md → garden.db (`--dry-run` supported)
- `dedupe_scanner.py` — delegates load_checked_pairs/record_pair to garden_db
- `integrate_entry.py` — upserts entries_index after every merge
- `init_garden.py` — creates garden.db + .gitattributes instead of CHECKED.md
- `validate_garden.py` — `--check-db` flag: schema version + table counts
- `harvest/SKILL.md` — DEDUPE references garden.db throughout
- 697 tests passing

**Phase 5-8 tooling also in soredium:**
- `validate_schema.py`, `init_garden.py` — garden initialization (Phase 5)
- `garden_config.py`, `route_submission.py`, `augment_entry.py` — federation (Phase 6)
- `mcp_garden_search.py`, `mcp_garden_status.py`, `mcp_garden_capture.py`, `garden_mcp_server.py` — MCP server (Phase 8)
- `garden_web_data.py` — JSON data builder for web app (Phase 7)

### `hortora.github.io` — significant updates

- `docs/garden.html` — garden browser (GitHub API + Fuse.js + entry detail panel)
- `docs/obsidian.html` — Obsidian integration guide (Dataview queries, sparse checkout setup)
- `docs/architecture.html` — reasoning preservation section added
- Blog entry 08: "Beyond Claude" — phases 5-8 narrative

### `Hortora/garden` — 3 open PRs from forage sweep

- #58 — `GE-20260415-d07a2c`: FastMCP list serialization (one TextContent per element)
- #59 — `GE-20260415-7ca64f`: truncated hash PK + INSERT OR IGNORE silently discards on collision
- #60 — `GE-20260415-95b40f`: git textconv driver makes binary diffs human-readable

---

## What To Do Next

**Immediate:** Merge garden PRs #58–60.

**Practical gap:** The canonical gardens haven't been created yet. `jvm-garden` and `tools-garden` exist as concepts but not repos. Run `init_garden.py`, push to GitHub, merge existing entries. The MCP server and garden browser point to `Hortora/garden` — they need live canonical repos to be useful.

**Phase 9 (next major):** Federation Deep + Advanced Quality — `_watch/` CI, retrieval success feedback, contributor credibility. Depends on canonical gardens being live.

**Garden health backlog** (`spec/docs/garden-health-2026-04-15.md`):
- 1 truncated entry: `GE-0107` in tools/
- 123 legacy entries with empty tags — plan documented in health check file
- CHECKED.md migration: run `garden_db_migrate.py` on the live garden

### Standing maintenance

- Merge garden PRs as they accumulate
- Run `validate_garden.py --check-db` after migration to confirm garden.db is healthy
- Run `validate_garden.py --freshness` to check staleness

---

## Key ADRs

- ADR-0004: `spec/docs/adr/0004-sqlite-aggregate-state.md` — SQLite replaces CHECKED.md (decided and implemented)

## Reference Links

| Resource | Location |
|----------|----------|
| Phase 5 plan | `spec/docs/superpowers/plans/2026-04-15-phase5-ecosystem-foundation.md` |
| Phase 6 plan | `spec/docs/superpowers/plans/2026-04-15-phase6-federation-shallow.md` |
| Phase 7 plan | `spec/docs/superpowers/plans/2026-04-15-phase7-human-interfaces.md` |
| Phase 8 plan | `spec/docs/superpowers/plans/2026-04-15-phase8-mcp-server.md` |
| SQLite plan | `spec/docs/superpowers/plans/2026-04-15-sqlite-aggregate-state.md` |
| Garden health check | `spec/docs/garden-health-2026-04-15.md` |
| ADR-0004 | `spec/docs/adr/0004-sqlite-aggregate-state.md` |
| Blog entry 08 | `hortora.github.io/_posts/2026-04-15-beyond-claude.md` |
| MCP server | `soredium/scripts/garden_mcp_server.py` |
| garden_db module | `soredium/scripts/garden_db.py` |
