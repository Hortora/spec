# Hortora — Project Handoff

*Last updated: 2026-05-21 (session 14)*

---

## What Hortora Is / Local Folder Structure

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## The Three Repos — delta only

### `soredium`

**All soredium issues closed (session 14).** #46, #54, #55, #56, #57, #58, #59 all closed.

Key changes:
- `scripts/integrate_entry.py` — `parse_entry` fixed: `str.split('---', 2)` → regex match on line-boundary markers; handles `---` in quoted YAML values
- `scripts/validate_pr.py` — same `parse_entry` fix applied
- `scripts/integrate_entry.py` — `update_garden_by_technology` already wired (#54 was done in session 13 before handover was written)
- `forage/SKILL.md` — technique→protocol promotion note; `protocol-link` REVISE row extended
- `protocol/SKILL.md` — `garden_ref` field added to entry format; garden techniques as upstream sources documented; CAPTURE Step 2 and Skill Chaining updated
- `tests/test_integration.py` — shared `_make_git_garden()` helper extracted; two missing tests added; fixture frontmatter fields fixed

**Open issues:** None. Long-horizon epics (#10–13, #31) are the only remaining work.

### `Hortora/garden`

**Backfill complete.** 80 entries were absent from GARDEN.md By Technology section (not 512 — the `_summaries/` heuristic was wrong; most entries were already indexed). Surgical patch via `update_garden_by_technology()` only — full `integrate()` would have duplicated domain INDEX.md rows. 4 new domain INDEX.md files created (casehub-ledger, casehub-work, scelight, web). Garden validation: clean.

Garden entry submitted: GE-20260521-df2a10 — Python YAML frontmatter `str.split('---')` gotcha (tools domain).

### `casehub/parent`

*Unchanged — `git show HEAD~1:HANDOFF.md`*

### `hortora.github.io`

Blog entry 21 published: "The Parser That Couldn't Parse Itself" (2026-05-21).

---

## What To Do Next

**Immediate:** Pick from the long-horizon epics or tackle the `quarkus/` → `jvm/` merge (208 files — separate session, mechanical but large).

**Still pending:** Langchain4j fork upgrade. QE run with GPU. `quarkus/` → `jvm/` merge. Epics #10–13, #31.

---

## Key ADRs / Reference Links

| Resource | Location |
|---|---|
| Blog entry 21 | `hortora.github.io/_posts/2026-05-21-mdp02-parser-couldnt-parse-itself.md` |

*Previous refs — `git show HEAD~1:HANDOFF.md`*
