# Hortora — Project Handoff

*Last updated: 2026-04-12 — website docs, homepage Quick Start, garden migration complete.*

---

## What Hortora Is

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Local Folder Structure

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## The Three Repos

### `hortora.github.io` — major updates ✅

New this session:
- **4 docs pages** — Getting Started, How It Works, Skills Reference, Design Spec (with sidebar nav, C2 layout)
- **Homepage Quick Start section** — install via `/install-skills https://github.com/Hortora/soredium`, forage, harvest with git graphic
- **Getting Started simplified** — single `/install-skills` command (removed curl/python3 approach)
- All pushed and live at hortora.github.io

### `spec` — unchanged since 2026-04-10 ✅

### `soredium` — multiple fixes pushed ✅

- `validate_garden.py`: new GE-ID format now first in alternation — prevents `GE-\d{4}` from greedily matching prefix of `GE-YYYYMMDD-xxxxxx`
- `integrate_entry.py`: reads GE-ID from frontmatter `id` field (not filename); removed `close_github_issue`; guards missing frontmatter
- `CLAUDE.md`: migration constraint section replaced with "Migration complete" status

### `Hortora/garden` — housekeeping complete ✅

- 9 submissions merged (electron×3, git×3, jest×1, python×1, hortora validator×1)
- 11 previously unindexed entries given GE-IDs and added to GARDEN.md
- `Last assigned ID` renamed to `Last legacy ID` (vestigial counter, new IDs are date-based)
- **Drift: 20** — DEDUPE should run next session (threshold 10)
- 1 pending submission: Hortora/garden#16 (regex alternation PR, awaiting CI + merge)

### cc-praxis — updated ✅

- `CLAUDE.md` Key Skills: `garden` replaced with `forage` + `harvest` entries
- `handover/SKILL.md`: all "garden sweep" → "forage sweep", `garden` CAPTURE → `forage` CAPTURE
- `~/.claude/skills/garden/` removed — legacy skill deprecated

---

## Migration Status

**Complete.** forage + harvest are the only garden skills installed. All CLAUDE.md files updated. The legacy `garden` skill source still lives in cc-praxis repo (task #8 — full removal deferred, low priority).

---

## What To Do Next

### Immediate

**1. Merge Hortora/garden#16** — regex alternation PR (CI validates, then merge)

**2. Run DEDUPE** — drift at 20, threshold 10. Run `/harvest DEDUPE` in a dedicated session.

**3. Write blog entry** — this session has good material (docs site, migration complete, garden housekeeping). Skipped due to context budget.

### Later

**4. Migrate 172 entries to YAML frontmatter** (task #5) — needs a migration script in soredium. Design questions: which markdown fields map to which YAML keys, how to handle missing fields, derive `domain` from directory path. Start with brainstorm in a fresh session.

**5. Fully remove `garden` skill from cc-praxis source** — low priority, no functional impact.

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
| garden PR#16 | https://github.com/Hortora/garden/pull/16 |
| Live site | https://hortora.github.io |
