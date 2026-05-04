# Hortora — Project Handoff

*Last updated: 2026-05-04 (session 7)*

---

## What Hortora Is / Local Folder Structure

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## The Three Repos — delta only

### `soredium`

**garden-engine** shipped (Phases 1–4, 171 tests, #39 closed):
- FeatureExtractor, ClusterPipeline, DeltaAnalysis, ProjectRegistry (Phase 1)
- Langchain4j 1.9.1 wired: AI services as @RegisterAiService, MockReasoningService
  intercepts all CDI in tests, JLama as default (Phase 2+3)
- QE matrix: `qe --matrix`, ModelComparisonResult, QEMatrixReport (Phase 3)
- SemanticDeduplicator: classify pairs → merge on DUPLICATE (Phase 4)
- `.mvn/maven.config` workaround for Quarkus 3.33.1 + JLama bootstrap bug

**forage skill** updated:
- Delivery steps now use concrete resolved garden path (no shell variable expansion)
- `required-permissions` frontmatter field declared in SKILL.md for install-skills

**langchain4j fork** (`mdproctor/quarkus-langchain4j`):
- Pushed fixes to `fix/jlama-dev-mode-jvm-options-0.26.x`: ChatMemoryProcessor
  and JlamaProcessor runtime-config-in-@BuildStep fixes (722c5440, 18388ee8)
- Fork still targets Quarkus 3.15.2 — cannot use directly in garden-engine (3.33.1)
  until fork's Quarkus target is upgraded

### `Hortora/garden`

- 37 stranded entries extracted from 20 stale PRs and committed
- 63 ghost entries recovered from git history
- validate_garden.py fixed to recognise YAML `id:` frontmatter (was only seeing
  legacy `**ID:**` body format)
- Forage now pushes directly to main (no PR workflow)
- 2 new garden entries: @ConfigProperty null outside CDI, Claude Code expansion
  check separate from allowlist

### `hortora.github.io`

- Blog entry 14 added: "The Garden Debt and a Java Engine" (2026-05-04)

---

## What To Do Next

**Immediate:** Upgrade langchain4j fork to target Quarkus 3.33.1, then remove
`.mvn/maven.config` workaround from garden-engine.

**Langchain4j upstream tickets:** Draft ready in session history — create
issue for JlamaProcessor @BuildStep runtime config (commits 722c5440, 18388ee8)
and comment on existing #2375 with fix commit refs.

**Next build:** Area 2 Phase 5 — `validate_schema.py` + `init_garden.py`
(issue #29, Hortora/soredium).

**QE run:** Once Ollama/JLama model available with GPU, run
`qe --matrix --tasks=dedup,pattern --sample=10` to validate free model adequacy.

---

## Key ADRs / Reference Links

*Unchanged — `git show HEAD~1:HANDOFF.md`*
