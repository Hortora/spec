# Hortora — Project Handoff

*Last updated: 2026-05-05 (session 8)*

---

## What Hortora Is / Local Folder Structure

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## The Three Repos — delta only

### `soredium`

**Phase 5 complete and closed** (#29 closed):
- `validate_schema.py` + `init_garden.py` + `validate_garden.py` integration shipped
- Code review found 4 issues; all fixed (commit `9193f1c`):
  - Critical: drift counter unbolded → `--dedupe-check` regex silently read 0
  - Important: `description` field not validated despite being in spec
  - Important: schema warnings routed to `log_info` (invisible without `--verbose`)
  - Important: dead `create_checked_md`/`create_discarded_md` functions removed
- MCP protocol tests: 29 failures fixed with `sys.executable` not `'python3'`
  (commit `80caf06`) — `McpError: Connection closed` was a wrong interpreter
- **836 tests, 0 failures** (all MCP extended tests now passing)
- CLAUDE.md updated: Phase 5 status, test deps (`pip install pytest mcp pyyaml ...`)

**garden-engine** (Phases 1–4, 171 tests, #39 closed):
*Unchanged — `git show HEAD~1:HANDOFF.md`*

**forage skill** + **langchain4j fork:**
*Unchanged — `git show HEAD~1:HANDOFF.md`*

### `Hortora/garden`

2 new entries this session:
- `tools/GE-20260505-f60bab` — MCP `StdioServerParameters command='python3'` spawns wrong interpreter
- `tools/GE-20260505-14159c` — `init_garden.py` unbolded drift counter silently breaks `--dedupe-check`

Prior session garden work — *Unchanged — `git show HEAD~1:HANDOFF.md`*

### `hortora.github.io`

- Blog entry 15 added: "Phase 5 Done — and Two Bugs That Hid" (2026-05-05)

---

## What To Do Next

**Immediate:** Upgrade langchain4j fork to target Quarkus 3.33.1, then remove
`.mvn/maven.config` workaround from garden-engine.

**Langchain4j upstream tickets:** Draft ready in session history — create
issue for JlamaProcessor @BuildStep runtime config (commits 722c5440, 18388ee8)
and comment on existing #2375 with fix commit refs.

**QE run:** Once Ollama/JLama model available with GPU, run
`qe --matrix --tasks=dedup,pattern --sample=10` to validate free model adequacy.

---

## Key ADRs / Reference Links

*Unchanged — `git show HEAD~1:HANDOFF.md`*
