# Hortora — Project Handoff

*Last updated: 2026-04-09 — session covering soredium bootstrap,
forage+harvest skills, git-only model, 118 tests, DEDUPE sweep.*

---

## What Hortora Is

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Local Folder Structure

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## The Three Repos

### `hortora.github.io` — unchanged, live

### `spec` — pushed to GitHub ✅
README and .gitignore added. All content committed and pushed.
Open issue: none.

### `soredium` — fully scaffolded ✅
CLAUDE.md (type:skills, migration constraint documented), marketplace.json,
scripts/claude-skill, Work Tracking enabled (issue tracking active).

**Skills written and installed to `~/.claude/skills/`:**
- `forage` — CAPTURE, SWEEP, SEARCH, REVISE, IMPORT (replaces garden session-time ops)
- `harvest` — MERGE, DEDUPE (replaces garden maintenance ops)

**Test suite: 118 tests passing** (`python3 -m pytest tests/ -v`)
- test_claude_skill.py, test_validate_garden.py, test_skill_structure.py
- test_git_operations.py (concurrent conflict, rebase recovery, ls-tree)
- test_integration.py (full CAPTURE→MERGE→validate with realistic fixtures)

**All soredium issues closed:** #1–#8.

---

## The Live Garden (Current State)

**Last assigned ID:** GE-0137
**Last full DEDUPE sweep:** 2026-04-09 (475 pairs — new entries only)
**Entries merged since last sweep:** 2 (GE-0136, GE-0137 pending harvest)
**119 entries** indexed in GARDEN.md (rebuilt from scratch this session).

**2 pending submissions** — GE-0136 (validator code fence false positive),
GE-0137 (git push rejected to non-bare repo). Run harvest MERGE.

---

## The Skill Migration Plan

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## Key Decisions Made

*Previous decisions unchanged — `git show HEAD~1:HANDOFF.md`*

Additional decisions this session:

| Decision | Rationale |
|----------|-----------|
| Git-only reads in forage+harvest | No filesystem reads; git HEAD is single source of truth; works local and remote |
| validate_garden.py strips code fences | Prevents false positives when GE-IDs appear as examples in entries |
| garden skill stays active until forage+harvest validated | Need more real sessions; session-handover still calls garden — intentional |

---

## What To Do Next

### Immediate — unblocked now

**1. Harvest the 2 pending submissions**
```bash
# Invoke harvest MERGE
```

**2. Continue testing forage+harvest in real sessions** — a few more sessions
before deprecating garden. When ready: update session-handover and other callers,
then deprecate garden in cc-praxis.

**3. Full DEDUPE of existing entries** — today's sweep only covered new entries
(GE-0105–GE-0124) against existing. Existing-vs-existing pairs across tools/
(60 entries), quarkus/ (22 entries), etc. are largely unchecked. Multi-session.

### Later

**4. Data migration** (Phase 1 of spec) — migrate 119 entries to Hortora v2 format.
Requires tooling in soredium first.

**5. GitHub backend** (Phase 2 of spec) — PR-based submission, CI validation.

---

## Reference Links

| Resource | Location |
|----------|----------|
| Full design spec | `~/claude/hortora/spec/docs/design/2026-04-07-garden-rag-redesign-design.md` |
| forage skill | `~/claude/hortora/soredium/forage/SKILL.md` |
| harvest skill | `~/claude/hortora/soredium/harvest/SKILL.md` |
| validate_garden.py | `~/claude/hortora/soredium/scripts/validate_garden.py` |
| Installed skills | `~/.claude/skills/` |
| Live garden | `~/claude/knowledge-garden/` |
| soredium issues | https://github.com/Hortora/soredium/issues |
| Live site | https://hortora.github.io |
