# Hortora — Project Handoff

*Last updated: 2026-04-12 — garden#16 merged, DEDUPE complete, blog entry 05 written.*

---

## What Hortora Is

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Local Folder Structure

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## The Three Repos

### `hortora.github.io` — blog entry added ✅

- Blog entry 05 written and committed: `_posts/2026-04-12-from-prototype-to-installable.md`
- Everything else unchanged from previous handover

### `spec` — unchanged ✅

### `soredium` — unchanged ✅

### `Hortora/garden` — DEDUPE complete, garden#17 open ✅

- garden#16 (regex alternation fix) — **merged**
- DEDUPE run: 1,002 pairs across java-panama-ffm, quarkus, electron, tools
  - 8 related (cross-refs added), 994 distinct, 0 duplicates
  - Drift reset to 0
- **garden#17 open** — forage submission GE-20260412-b6c0f8 (Read tool 256KB limit gotcha)
  - Waiting on CI + merge, same pattern as #16

---

## Migration Status

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## What To Do Next

### Immediate

**1. Merge garden#17** — Read tool 256KB limit gotcha. CI validates, then merge (same flow as #16).

### Later

**2. Migrate 172 entries to YAML frontmatter** (task #5) — needs a migration script in soredium.

**3. Fully remove `garden` skill from cc-praxis source** — low priority, no functional impact.

---

## Reference Links

| Resource | Location |
|----------|----------|
| Latest design docs | `spec/docs/design/` |
| ADR-0003 (GE-ID scheme) | `spec/docs/adr/0003-ge-id-scheme-date-plus-random-hex.md` |
| forage skill | `~/.claude/skills/forage/SKILL.md` |
| harvest skill | `~/.claude/skills/harvest/SKILL.md` |
| validate_garden.py | `~/claude/hortora/soredium/scripts/validate_garden.py` |
| integrate_entry.py | `~/claude/hortora/soredium/scripts/integrate_entry.py` |
| Live garden | `~/.hortora/garden/` |
| garden PR#17 | https://github.com/Hortora/garden/pull/17 |
| Live site | https://hortora.github.io |
