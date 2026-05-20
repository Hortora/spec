# Hortora — Project Handoff

*Last updated: 2026-05-21 (session 13)*

---

## What Hortora Is / Local Folder Structure

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## The Three Repos — delta only

### `soredium`

**Nit sweep complete (session 13).** Issues #49–53, #48, #46, #47 closed:
- `validate_garden.py` — `SKIP_NAMES` module-level constant; `defaultdict` to module imports; PyYAML-absent test
- `validate_pr.py` — `garden_path` conversion at entry; Jaccard leading-space guard; double-warning suppression for non-convention same-title siblings; `protocol:` format check; `invalidation_status: pending|resolved` format check
- `forage/SKILL.md` — `protocol-link`, `flag-pending`, `flag-resolved` revision kinds in REVISE table
- `harvest/SKILL.md` — REVIEW Step 3 surfaces `invalidation_triggers`; focused prompt for `invalidation_status: pending` entries
- `forage/submission-formats.md` — `protocol:` and `invalidation_status:` in Optional Fields table
- `work-start` skill — Step 3b garden search added (synced via cc-praxis)
- `docs/protocols/` — created; PP-20260521-f84059 (`validate_schema.py` scope rule)

**#45 skipped** — GARDEN.md two-level rendering is blocked by #54. `integrate_entry.py` doesn't write the By Technology listing at all (only drift counter); the rendering is a refinement of indexing work that hasn't landed yet.

**Open issues:** #54 (CAPTURE deliver step — top priority), #56 (backfill 512 entries), #45 (two-level rendering, blocked by #54), #57–59 (test infrastructure nits).

### `Hortora/garden`

*Unchanged — `git show HEAD~1:HANDOFF.md`*

### `casehub/parent`

*Unchanged — `git show HEAD~1:HANDOFF.md`*

### `hortora.github.io`

Blog entry 20 published: "Nits and a Design Call" (2026-05-21).

---

## What To Do Next

**Immediate:** Start #54 — wire `integrate_entry.py` into forage CAPTURE Step 8 (Deliver). The CAPTURE skill currently writes the entry file but never calls `integrate_entry.py`, so GARDEN.md, `_index/global.md`, `labels/`, and domain `INDEX.md` are never updated — 512 entries are dark. Read `forage/SKILL.md` CAPTURE Step 8 first, then read `scripts/integrate_entry.py` to confirm the invocation signature. Note: `integrate_entry.py` only updates the drift counter in GARDEN.md, not the By Technology listing — implementing the listing update is also part of #54's scope (it's what #45 depends on). Test against a scratch garden before touching the live one.

**Then #45** — once integrate_entry.py writes the By Technology listing, add two-level rendering for same-title convention entries (single entry → direct link; multiple → sub-list with variant as label).

**Still pending:** Langchain4j fork upgrade. QE run with GPU. `quarkus/` → `jvm/` merge (208 files, separate session).

---

## Key ADRs / Reference Links

| Resource | Location |
|---|---|
| integrate_entry.py | `soredium/scripts/integrate_entry.py` |
| forage CAPTURE Step 8 | `soredium/forage/SKILL.md` |
| soredium protocols index | `soredium/docs/protocols/INDEX.md` |
| Blog entry 20 | `hortora.github.io/_posts/2026-05-21-mdp01-nits-and-a-design-call.md` |

*Previous refs — `git show HEAD~1:HANDOFF.md`*
