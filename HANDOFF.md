# Hortora — Project Handoff

*Last updated: 2026-04-22 (session 5)*

---

## What Hortora Is / Local Folder Structure

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## The Three Repos — delta only

### `soredium` — Garden Agent shipped

`soredium/scripts/garden-agent-install.sh` — idempotent bash installer, 12 TDD tests.
Installs into any garden: `garden-agent.sh`, `.claude/settings.json`, `CLAUDE.md`,
post-commit hook, `.gitignore` entries. No tokens consumed at install time.

**New machine setup:**
```bash
cd ~/.hortora/garden
~/claude/hortora/soredium/scripts/garden-agent-install.sh
```

### `Hortora/garden`

- DEDUPE sweep: 50 pairs processed — 4 duplicates discarded, cross-references added
  to 10 clusters. Drift counter reset.
- Garden agent installed: `garden-agent.sh`, `.claude/settings.json`, `CLAUDE.md`,
  post-commit hook active.
- Log: `~/.hortora/garden/garden-agent.log` (rotates at 1MB, keeps last 5).
- PRs #81, #84, #86 — status unchanged (still open from previous session).

### `hortora.github.io`

Blog entry 13: "Letting the Garden Tend Itself"
(`2026-04-22-mdp01-letting-the-garden-tend-itself.md`) — committed.

### `spec`

- `docs/superpowers/specs/2026-04-21-garden-agent-design.md` — agent design spec
- `docs/superpowers/plans/2026-04-21-garden-agent-install.md` — implementation plan
- Phases 5–8 plans committed (Apr 14–15 dates, were untracked)

---

## What To Do Next

**Immediate:** Merge open garden PRs #81, #84, #86.

**Next build session: Area 2 Phase 5** — seed `registry/projects.yaml` with 3–5
JVM projects, run `run_pipeline.py` against real cloned repos, review first real
candidates through `validate_candidates.py`.

**Garden agent** — monitor `garden-agent.log` over the next few sessions to verify
it fires correctly on forage commits.

**soredium README** — `garden-agent-install.sh` not yet documented in the README
(noted at session end, not yet done).

---

## Key ADRs / Reference Links

*Unchanged — `git show HEAD~1:HANDOFF.md`*
