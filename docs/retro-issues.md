# Retro Issues Proposal

Covers all three Hortora repos. Existing soredium issues #1–9 cover 2026-04-08/09 init and Phase 1/2 foundation. All issues below are new.

---

## SOREDIUM — Hortora/soredium

### Epic A: GE-ID scheme migration and post-Phase-2 stabilisation
**Dates:** 2026-04-10 → 2026-04-13
**DoD:** All entries use GE-YYYYMMDD-xxxxxx format; validator handles both legacy and new IDs; YAML frontmatter is the single source of truth; bulk integration works.

| # | Title | Type | Commits |
|---|---|---|---|
| A1 | Implement date+hex GE-ID scheme — ADR-0003 | enhancement | `71b7b7f` feat: new GE-ID scheme |
| A2 | Fix validator and skill files for new GE-ID and YAML frontmatter | bug | `1208ad2` fix alternation; `902082b` read GE-ID from frontmatter; `043be65` update validator field name; `afbf8a5` unify YAML frontmatter; `a3cae7f` restore REVISE logic |
| A3 | Add bulk_integrate.py for batch garden index population | enhancement | `819957c` bulk_integrate.py; `7eb73b7` docs: mark migration complete |

### Epic B: Phase 3 — Hybrid staleness enforcement
**Dates:** 2026-04-14
**DoD:** Every SEARCH result shows entry age; stale entries are flagged; harvest REVIEW mode exists; `validate_garden.py --freshness` reports overdue count.

| # | Title | Type | Commits |
|---|---|---|---|
| B1 | Add --freshness flag to validate_garden.py (TDD) | enhancement | `9625cca` tests; `1b08145` impl; `b7cdcd8` fix CRLF+ValueError |
| B2 | Document verified_on and last_reviewed optional fields in forage | documentation | `c059ddf` docs(forage) |
| B3 | Add S2 staleness annotation to forage SEARCH | enhancement | `db9929c` feat(forage): S2 |
| B4 | Add S6 verified_on and S3 rationale prompts to forage CAPTURE | enhancement | `0f0444b` feat(forage): S6+S3 |
| B5 | Add S1 domain-filtered staleness spot-check to forage SWEEP | enhancement | `80c104f` feat(forage): S1 |
| B6 | Add harvest REVIEW mode for systematic staleness enforcement | enhancement | `9735a76` feat(harvest): REVIEW |

### Epic C: Phase 4 — Full deduplication system
**Dates:** 2026-04-14
**DoD:** Drift counter auto-increments on every integrate; --dedupe-check warns at threshold; dedupe_scanner.py Jaccard-scores unchecked pairs; harvest DEDUPE uses scanner.

| # | Title | Type | Commits |
|---|---|---|---|
| C1 | Auto-increment drift counter in integrate_entry.py (TDD) | enhancement | `8b6ef86` feat(integrate) |
| C2 | Add --dedupe-check flag with git-log cross-verification (TDD) | enhancement | `456ac4b` feat(validate) |
| C3 | Add dedupe_scanner.py with Jaccard pre-scoring and full test suite | enhancement | `453a251` feat(dedupe); `04f7de5` test integration; `6117c36` test CLI; `ddbea83` test E2E; `ef24b70` feat(harvest): update DEDUPE |

### Epic D: Contextual capture — WHY schema enrichment and contributor scoreboard
**Dates:** 2026-04-14
**DoD:** Garden entries can capture constraints, alternatives considered, invalidation triggers; validator awards bonus points per WHY field; contributors.py generates CONTRIBUTORS.md ranked by avg effective score.

| # | Title | Type | Commits |
|---|---|---|---|
| D1 | Add WHY bonus scoring (BONUS_RULES, compute_bonus) to validate_pr.py (TDD) | enhancement | `255ceb3` feat(validate): WHY bonus |
| D2 | Add contributors.py for scoreboard generation (TDD) | enhancement | `4cb583e` feat(contributors) |
| D3 | Update submission-formats.md and forage SKILL.md for WHY field capture | enhancement | `a64a820` docs(forage); `eaf7205` feat(forage): WHY prompts |

---

## SPEC — Hortora/spec

### Epic E: Phase 2 technical specifications and design artifacts
**Dates:** 2026-04-09
**DoD:** Design specs and implementation plans exist for Phase 2 GitHub backend, website documentation, and homepage Quick Start.

| # | Title | Type | Commits |
|---|---|---|---|
| E1 | Phase 2 GitHub backend design spec and implementation plan | documentation | `cbeef1d` design; `f9332b1` plan |
| E2 | Website documentation design spec and implementation plan | documentation | `ad2ed9c` design; `a5bdb6b` plan |
| E3 | Homepage Quick Start design spec and implementation plan | documentation | `0fcb0cc` design; `9b755f7` plan |
| E4 | ADR-0002 (CI script location) and founding blog entries | documentation | `1a3a18e` ADR-0002; `9ca3c0` blog-designing-the-engine |

### Epic F: Phase 3–4 and contextual capture design documentation
**Dates:** 2026-04-14
**DoD:** Design specs and implementation plans exist for Phase 3 staleness, Phase 4 deduplication, contextual capture, and the architecture page.

| # | Title | Type | Commits |
|---|---|---|---|
| F1 | Phase 3 staleness enforcement — design spec and implementation plan | documentation | `f3b19b9` plan |
| F2 | Phase 4 full deduplication system — design spec and implementation plan | documentation | `fb53999` spec; `be6df01` plan |
| F3 | Contextual capture (WHY schema enrichment) — design spec and implementation plan | documentation | `3a3d276` spec; `6ddc3f3` plan |
| F4 | Architecture page for enterprise architects — design spec and implementation plan | documentation | `069bad4` spec; `167cc77` spec+section6; `9bee08a` plan |

### Standalones — spec

| # | Title | Type | Commits |
|---|---|---|---|
| S-spec-1 | ADR-0003 GE-ID scheme and staleness IDEAS.md documentation | documentation | `9294e4f` ADR-0003; `dc19123` ideas log; `bcc0ab1` ideas expanded |
| S-spec-2 | Design snapshots and README consolidation | documentation | `5f3ed0d` snapshot Phase2; `6aaa75e` snapshot 2026-04-10; `fab7def` snapshot 2026-04-13; `4ba8d3f` consolidate README; `f57d9554` consolidate snapshots |

### Excluded — spec

| Commit | Message | Reason |
|---|---|---|
| `59489c9` | docs: session handover 2026-04-09 | Operational continuity artifact |
| `b7538a2` | docs: session handover 2026-04-10 | Operational continuity artifact |
| `9265c4f` | docs: session handover 2026-04-12 | Operational continuity artifact |
| `05e860f` | docs: session handover 2026-04-12 | Operational continuity artifact |
| `97fed23` | docs: session handover 2026-04-13 | Operational continuity artifact |
| `fc88082` | docs: session handover 2026-04-13 | Operational continuity artifact |
| `58db178` | docs: session handover 2026-04-14 | Operational continuity artifact |
| `2d0dfe1` | init: add README and .gitignore | Init scaffolding |
| `2a781f3` | docs: fix handoff folder structure | Trivial path fix |
| `07c1e4b` | docs: add project handoff | Operational — initial HANDOFF.md |
| `8eea28b` | init: seed hortora/spec with design docs | Init — covered by spec #1 |
| `13d966` | docs: remove Red Hat name from blog entry | Trivial content fix |
| `1f78368` | docs: add CLAUDE.md — blog directory | Config/setup |
| `b0f11dd` | chore: enable Work Tracking | Setup, this session |

---

## SITE — Hortora/hortora.github.io

### Standalones — site

| # | Title | Type | Commits |
|---|---|---|---|
| S-site-1 | Migrate site to Jekyll with GitHub Actions CI | enhancement | `9830e3d` convert to Jekyll; `1a4e00d` GH Actions CI; `19eda7d` fix permalinks; `b10ea6f` highlight.js |
| S-site-2 | Phase 2 documentation updates — GE-ID, garden path, Phase 2 features | documentation | `2be4b981` GE-ID update; `1a32ba2e` garden path; `b5a8533` Phase 2 docs; `5639c46` Phase 2 content |
| S-site-3 | Add Architecture page for platform/enterprise architects | enhancement | `acac692` nav updates; `9500c30` architecture.html |
| S-site-4 | Session blog entries — Phase 2 cleanup through Phase 3 (entries 03–06) | documentation | `f63dc23` blog-2026-04-10; `677896` blog-2026-04-12; `c242030` blog-2026-04-14 |

### Excluded — site

| Commit | Message | Reason |
|---|---|---|
| `0370d25` | init: Hortora landing page | Init — covered by existing site work |
| `e5a56c5` | feat: add blog with two founding entries | Covered by existing site structure |
| `bac19c7` | fix: explicit index.html in all hrefs | Trivial path fix |
| `0415e11` | fix: use relative paths throughout | Trivial path fix |
| `e00ba5a` | fix: --sage-light variable | Trivial CSS fix |
| `29e9fb7` | feat: add docs layout CSS classes | Covered by existing site #1 "Add documentation section" |
| `47d13f3` | feat: add Docs nav link | Covered by site #1 |
| `57b321c` | feat: add Docs nav link to blog pages | Covered by site #1 |
| `96f7c43` | feat: add Getting Started docs page | Covered by site #1 |
| `a25d7d2` | fix: clarify forage trigger, aria-labels | Trivial accessibility fix |
| `0b9e558` | feat: add How It Works docs page | Covered by site #1 |
| `739d55c` | feat: add Skills Reference docs page | Covered by site #1 |
| `cd8080a` | feat: add Design Spec docs page | Covered by site #1 |
| `33a5c48` | feat: add Quick Start CSS classes | Covered by site #2 |
| `e33c22d` | feat: add Quick Start section to homepage | Covered by site #2 |
| `99868628` | fix: simplify Getting Started install | Covered by site #1/2 |
| `5639c46` | docs: update site for Phase 2 | Part of S-site-2 above |

---

## Issue count summary

- soredium: 4 epics, 16 child issues
- spec: 2 epics, 8 child issues, 2 standalones
- site: 4 standalones

Total new issues: ~30 (plus 6 epics)
