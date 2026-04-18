# Hortora — Project Handoff

*Last updated: 2026-04-18*

---

## What Hortora Is

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Local Folder Structure

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## The Three Repos — delta only

### `soredium` — small change this session

forage SWEEP Step 4 rewritten for batched delivery:
- Each confirmed entry still goes through CAPTURE steps 0–6 individually (user confirmation is sequential)
- Validation deferred until all files written; parallel (`&` + `wait`) if ≥3 entries, sequential below
- Delivery: single branch + single commit + single PR for the whole sweep
- Committed [soredium#30](https://github.com/Hortora/soredium/issues/30), pushed, synced to `~/.claude/skills/forage/`

Everything else in soredium (SQLite, federation, MCP server, 697 tests) — *unchanged, `git show HEAD~1:HANDOFF.md`*

### `hortora.github.io`

Blog entry 09: "Batching the Sweep" (`_posts/2026-04-18-forage-sweep-batching.md`) — committed.

Previous session's content (garden browser, Obsidian guide, blog entry 08) — *unchanged, `git show HEAD~1:HANDOFF.md`*

### `Hortora/garden` — 3 open PRs from forage sweep

*Unchanged — `git show HEAD~1:HANDOFF.md`* (PRs #58–60 still open)

---

## What To Do Next

*Unchanged — `git show HEAD~1:HANDOFF.md`*

Key items: merge garden PRs #58–60; create canonical `jvm-garden` + `tools-garden` repos via `init_garden.py`; Phase 9 (Federation Deep + Advanced Quality) depends on canonical gardens being live.

---

## Key ADRs / Reference Links

*Unchanged — `git show HEAD~1:HANDOFF.md`*
