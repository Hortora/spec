# Hortora — Project Handoff

*Last updated: 2026-04-22 (session 6)*

---

## What Hortora Is / Local Folder Structure

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## The Three Repos — delta only

### `soredium`

`garden-agent-install.sh` updated: installs `run-scanner.sh` wrapper and
uses `Bash(bash run-scanner.sh *)` in `settings.json` allowlist instead of
`python3 */dedupe_scanner.py *`. Removes `--dangerously-skip-permissions`.

### `Hortora/garden`

- `run-scanner.sh` added — wraps dedupe_scanner.py to avoid `${:-}` expansion
  in agent commands, keeping permission system intact
- `CLAUDE.md` updated — agent now calls `bash run-scanner.sh` instead of
  constructing python3 commands with shell expansions
- `settings.json` updated — allowlist now covers `bash run-scanner.sh *`
- Garden agent ran autonomously: `dedupe: sweep 50 pairs — 21 related, 29 distinct`
- 3 forage entries submitted (claude-code domain): dangerously-skip-permissions
  risk, expansion prompt ordering, wrapper script pattern

### `hortora.github.io` / `spec`

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## What To Do Next

**Immediate:** Merge open garden PRs #81, #84, #86.

**Monitor:** Check `garden-agent.log` after next forage commit to confirm
agent runs without any prompts.

**Next build session: Area 2 Phase 5** — seed `registry/projects.yaml` with
3–5 JVM projects, run `run_pipeline.py` against real cloned repos.

**soredium README** — `garden-agent-install.sh` still undocumented there.

---

## Key ADRs / Reference Links

*Unchanged — `git show HEAD~1:HANDOFF.md`*
