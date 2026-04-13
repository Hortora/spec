# Hortora — Project Handoff

*Last updated: 2026-04-13 — all backlog items closed.*

---

## What Hortora Is

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Local Folder Structure

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## The Three Repos

### `hortora.github.io` — unchanged ✅

### `spec` — unchanged ✅

### `soredium` — significant updates ✅

- `scripts/bulk_integrate.py` — populates `_summaries/`, `_index/`, `labels/` for all entries in one pass
- `forage/SKILL.md` + `submission-formats.md` — unified to YAML frontmatter everywhere; REVISE now modifies target entry in-place; no submissions/ queue
- `harvest/SKILL.md` — DEDUPE-only; MERGE removed (no submissions queue)

### `Hortora/garden` — fully indexed ✅

- `_summaries/`, `_index/global.md`, `labels/`, domain `INDEX.md` files — all populated for all 181 entries
- `integrate-on-merge.yml` CI fixed — regex now matches new-format IDs (`GE-20260412-xxxxxx`)
- All open PRs resolved: #10–15 closed (cc-praxis wrong-repo), #18 merged, #19 closed (already on main)
- Garden pushed and current

### `cc-praxis` — garden skill removed ✅

- `garden/` skill source deleted
- Removed from `marketplace.json` and CLAUDE.md Key Skills
- forage/harvest were never in cc-praxis source — only globally installed from soredium

---

## What To Do Next

Nothing open. The garden is stable, fully indexed, and the CI correctly
integrates new entries on PR merge.

### Standing maintenance

- **DEDUPE** when drift reaches threshold 10 (`Entries merged since last sweep` in GARDEN.md)
- **`bulk_integrate.py`** if a future bulk import skips `integrate_entry.py`

---

## Reference Links

| Resource | Location |
|----------|----------|
| Latest design snapshot | `spec/snapshots/2026-04-13-garden-entry-format.md` |
| ADR-0003 (GE-ID scheme) | `spec/docs/adr/0003-ge-id-scheme-date-plus-random-hex.md` |
| Migration script | `~/claude/hortora/soredium/scripts/migrate_legacy_entries.py` |
| Bulk index script | `~/claude/hortora/soredium/scripts/bulk_integrate.py` |
| forage skill | `~/.claude/skills/forage/SKILL.md` |
| harvest skill | `~/.claude/skills/harvest/SKILL.md` |
| Live garden | `~/.hortora/garden/` |
| Live site | https://hortora.github.io |
