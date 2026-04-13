# Hortora — Project Handoff

*Last updated: 2026-04-13 — garden#17 merged, 174 legacy entries migrated to individual files.*

---

## What Hortora Is

*Unchanged — `git show HEAD~2:HANDOFF.md`*

## Local Folder Structure

*Unchanged — `git show HEAD~2:HANDOFF.md`*

---

## The Three Repos

### `hortora.github.io` — unchanged ✅

### `spec` — design snapshot added ✅

- `snapshots/2026-04-13-garden-entry-format.md` — freezes the entry format decision

### `soredium` — migration script added ✅

- `scripts/migrate_legacy_entries.py` — migrates legacy multi-entry .md files to individual YAML-frontmatter files; reusable

### `Hortora/garden` — major structural migration complete ✅

- garden#17 (Read tool 256KB limit gotcha) — **merged and pushed**
- **174 legacy entries** extracted from multi-entry `.md` files to individual `GE-NNNN.md` files
  - 63 source files deleted (fully migrated)
  - 6 source files updated (unindexed entries retained)
  - 303 GARDEN.md links updated
- 3 previously-unindexed new-format entries indexed: GE-20260412-17c8ce, GE-20260412-e4773d, GE-20260412-b6c0f8
- Garden pushed to remote — `Hortora/garden` main is current

**Garden structure:** all entries are now `<domain>/GE-XXXX.md` with YAML frontmatter. Multi-entry topic files are gone (except 6 files with unindexed entries).

---

## Migration Status

**Complete.** forage + harvest are the only garden skills. Legacy `garden` skill source still in cc-praxis (low priority removal).

---

## What To Do Next

### Immediate

**Nothing urgent.** Garden is stable and pushed.

### Later

**1. Auto-integrate on PR merge** — three entries were missing from GARDEN.md index after PR merges because `integrate_entry.py` doesn't run automatically post-merge. Either add a CI step or document the manual step clearly.

**2. Populate `_summaries/`, `_index/`, `labels/`** for the 174 migrated legacy entries — `integrate_entry.py` was never run for them.

**3. Remove legacy `garden` skill from cc-praxis source** — low priority.

---

## Reference Links

| Resource | Location |
|----------|----------|
| Latest design snapshot | `spec/snapshots/2026-04-13-garden-entry-format.md` |
| ADR-0003 (GE-ID scheme) | `spec/docs/adr/0003-ge-id-scheme-date-plus-random-hex.md` |
| Migration script | `~/claude/hortora/soredium/scripts/migrate_legacy_entries.py` |
| forage skill | `~/.claude/skills/forage/SKILL.md` |
| harvest skill | `~/.claude/skills/harvest/SKILL.md` |
| Live garden | `~/.hortora/garden/` |
| Live site | https://hortora.github.io |
