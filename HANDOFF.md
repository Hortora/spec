# Hortora — Project Handoff

*Last updated: 2026-05-17 (session 12)*

---

## What Hortora Is / Local Folder Structure

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## The Three Repos — delta only

### `soredium`

**soredium#44 complete.** `type: convention` + `variant:` field shipped:
- `validate_pr.py` — `find_same_title_siblings`, 4-branch variant check (CRITICAL/WARNING/suppression/non-convention), Jaccard suppression for confirmed sibling pairs
- `validate_garden.py` — check 8: same-title group must all have `variant:` or ERROR
- `forage/SKILL.md` — 11 enumeration sites updated, SWEEP Step 4 (convention scan), Proactive Trigger, editorial bar, scoring note, `quarkus/` marked as legacy domain
- `forage/submission-formats.md` — `variant:` in Optional Fields table, Convention Template

Protocol skill: `VALID_SCOPES` bug fixed (was missing `universal`, `application`). Synced.

**Open issues:** #45 (GARDEN.md two-level rendering), #46 (work-start garden search), #47 (harvest trigger research), #48 (protocol: field), #49-53 (minor nits).

### `Hortora/garden`

**Audit 3 complete:**
- `approaches/` domain retired — 3 entries moved to `jvm/`, `domain:` frontmatter corrected
- `quarkus/` documented as frozen legacy — new Quarkus entries route to `jvm/` only
- Garden pre-commit hook gotcha captured: GE-20260517-3dddfa (`tools/`)
- 5 convention/technique entries committed: GE-20260515-6e8205, da8abd, 70021c, c272d2, ffde26

**Critical finding — 512 unindexed entries:**
forage CAPTURE's deliver step never calls `integrate_entry.py`, so GARDEN.md index is incomplete for half the garden. git grep (forage SEARCH Step 3) still finds all entries. **Top priority for next session.**

### `casehub/parent`

**Protocol schema clean:** 7 violations fixed (severity: required/error → critical/important/guidance; type: convention → rule). 25/25 protocols pass HEALTH.

### `hortora.github.io`

Blog entry 19 published: "Conventions, Audits, and an Embarrassing Gap" (2026-05-17).

---

## What To Do Next

**Immediate:** Fix the 512-entry index gap — wire `integrate_entry.py` into forage CAPTURE Step 8 (Deliver). Read `scripts/integrate_entry.py` first to understand what it updates (GARDEN.md, `_index/global.md`, `labels/`, domain `INDEX.md`). Test against a scratch garden before touching the live one. Open a Hortora/soredium issue before starting.

**Then: Audit 4** — `repos/` depth check in `casehub/parent/docs/repos/`. 9 files (6 foundation repos + devtown, aml, clinical). Rule: each file stays at family-awareness level — module ownership, what it does NOT do, dependency graph. No class names, no method signatures. For each file: identify anything that has crept into class-level design detail, trim it, add a reference to the module's own DESIGN.md in its place. Start by listing all 9 files and skimming for the depth boundary.

**Still pending:** Langchain4j fork upgrade. QE run with GPU. `quarkus/` → `jvm/` merge (208 files, separate session).

---

## Key ADRs / Reference Links

| Resource | Location |
|---|---|
| Convention schema spec | `soredium/docs/superpowers/specs/2026-05-14-convention-schema-design.md` |
| Blog entry 19 | `hortora.github.io/_posts/2026-05-17-mdp01-conventions-audits-index-gap.md` |
| integrate_entry.py | `soredium/scripts/integrate_entry.py` |

*Previous refs — `git show HEAD~1:HANDOFF.md`*
