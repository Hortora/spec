# Hortora — Project Handoff

*Last updated: 2026-04-10 — harvest, garden migration, path config, GE-ID redesign, staleness.*

---

## What Hortora Is

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Local Folder Structure

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## The Three Repos

### `hortora.github.io` — updated for path and ID scheme
Docs updated: `how-it-works.html`, `getting-started.html`, `index.html` all reflect
`~/.hortora/garden`, `HORTORA_GARDEN` env var, new GE-ID format, GitHub Issues removed.
Pushed.

### `spec` — pushed ✅
ADR-0003, design snapshot 2026-04-10, blog entry 2026-04-10, IDEAS.md created.

### `soredium` — pushed ✅
- `validate_garden.py`: accepts both `GE-\d{4}` (legacy) and `GE-\d{8}-[0-9a-f]{6}` (new)
- `garden-setup.sh`: uses `${HORTORA_GARDEN:-~/.hortora/garden}`
- `forage/SKILL.md`, `harvest/SKILL.md`: 62 path replacements, new GE-ID generation, GitHub Issues removed
- Skills synced to `~/.claude/skills/`

### `Hortora/garden` — live ✅
- Moved to `~/.hortora/garden` (was `~/claude/garden`)
- Symlink `~/claude/knowledge-garden` → `~/.hortora/garden` for backward compat
- 169+ entries; 13 submissions merged this session
- 1 pending submission: GE-20260410-5fd0c3 (validate_garden.py fence-stripping false positive)

---

## The Live Garden (Current State)

**Last assigned ID (legacy):** GE-0172  
**New IDs:** `GE-YYYYMMDD-xxxxxx` format (ADR-0003) — no counter, no coordination  
**Drift:** 0 (full DEDUPE completed this session)  
**Pending submissions:** 1 (GE-20260410-5fd0c3 — run `/harvest`)

---

## Key Changes This Session

| Change | Detail |
|--------|--------|
| Garden path | `~/claude/garden` → `~/.hortora/garden`; `HORTORA_GARDEN` env var; symlink preserved |
| GE-ID scheme | Sequential counter → `GE-YYYYMMDD-xxxxxx` (date + 6 random hex); ADR-0003 |
| GitHub Issues | Removed from forage capture flow entirely; PRs only |
| Staleness problem | Named and documented in `spec/IDEAS.md` with 6 solutions and likelihood ratings |
| Fence-stripping bug | Fixed in GE-0091 prose (literal ` ``` ` was confusing the validator regex) |

---

## What To Do Next

### Immediate — unblocked now

**1. Harvest the pending submission**
```bash
/harvest
```

**2. Update CI workflows in `Hortora/garden`**  
`validate-on-pr.yml` and `integrate-on-merge.yml` still reference old issue-based flow.
Need to accept new `GE-YYYYMMDD-xxxxxx` ID format and remove issue-creation/closing steps.

**3. Test forage GitHub mode end-to-end**  
Submit a real entry via PR, watch CI validate, merge, confirm indexes update.
Blocked on #2.

### Later

**4. Implement staleness enforcement** — `spec/IDEAS.md` has 6 solutions.
Easiest first: age annotations in forage SEARCH results (solution #2, hours of work).

**5. Deprecate legacy `garden` skill** — once forage+harvest validated across several sessions.

**6. Phase 3** — Claude-in-CI for borderline PRs, PyPI graduation for `garden-engine`.

---

## Reference Links

| Resource | Location |
|----------|----------|
| Design snapshot (this session) | `spec/docs/snapshots/2026-04-10-phase2-post-refinements.md` |
| ADR-0003 (GE-ID scheme) | `spec/docs/adr/0003-ge-id-scheme-date-plus-random-hex.md` |
| Staleness ideas | `spec/IDEAS.md` |
| Blog entry (this session) | `spec/docs/blog/2026-04-10-mdp01-post-phase2-cleanup.md` |
| forage skill | `~/.claude/skills/forage/SKILL.md` |
| harvest skill | `~/.claude/skills/harvest/SKILL.md` |
| validate_garden.py | `~/claude/hortora/soredium/scripts/validate_garden.py` |
| Live garden | `~/.hortora/garden/` |
| soredium issues | https://github.com/Hortora/soredium/issues |
| Live site | https://hortora.github.io |
