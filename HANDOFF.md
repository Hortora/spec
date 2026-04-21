# Hortora — Project Handoff

*Last updated: 2026-04-21 (session 4)*

---

## What Hortora Is / Local Folder Structure

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## The Three Repos — delta only

### `soredium` — Area 2 Phases 3 and 4 shipped

**Closes [#34](https://github.com/Hortora/soredium/issues/34) and [#35](https://github.com/Hortora/soredium/issues/35) (epic #31):**

Phase 3 — ecosystem mining foundation:
- `project_registry.py` — CRUD over `registry/projects.yaml`
- `feature_extractor.py` — regex-based structural fingerprinting (6 features)
- `cluster_pipeline.py` — cosine similarity clustering, known-pattern tagging
- `delta_analysis.py` — new abstractions between git tags with author attribution

Phase 4 — pattern discovery loop closed:
- `rejection_registry.py` — cosine-similarity suppression of known noise
- `candidate_report.py` — JSON serialization of pipeline output
- `pattern_entry.py` — GP-ID skeleton generator for accepted patterns
- `validate_candidates.py` — human gate (`decide_fn` callback, testable)
- `run_pipeline.py` — full orchestrator: registry → extract → cluster → delta → report

**826 tests passing on main, pushed.**

### `Hortora/garden`

- PR #78 merged (52 entries + 10 restored that bad PR #56 had dropped)
- PR #81: GE-20260420-de730c (stale rebase state gotcha) — open
- PR #84: rescue/11 local-only entries — open
- PR #86: 5-entry session sweep — open
- Git hooks added: `pre-commit` blocks commits when untracked GE-*.md exist; `post-checkout` warns. Hooks in `.githooks/`, `garden-setup.sh` now configures `core.hooksPath`.

### `hortora.github.io`

Blog entry 12: "Cleaning Up, Closing the Loop" (`2026-04-21-mdp01-cleaning-up-closing-the-loop.md`) — committed and pushed.

Previous entries (09–11) — *unchanged, `git show HEAD~1:HANDOFF.md`*

### `spec`

Phase 3 + 4 plans committed:
- `docs/superpowers/plans/2026-04-21-area2-phase3-project-registry-and-mining.md`
- `docs/superpowers/plans/2026-04-21-area2-phase4-validation-gate-and-pipeline.md`

---

## What To Do Next

**Immediate:** Merge open garden PRs #81, #84, #86.

**Next build session: Area 2 Phase 5** — seed `registry/projects.yaml` with 3–5 monitored open-source JVM projects, run `run_pipeline.py` against actual cloned repos, review the first real candidates through `validate_candidates.py`.

**Canonical garden naming** — `init_garden.py` may need updating for knowledge-type-first names (`discovery-garden`, `patterns-garden`) vs old technology-domain names. Check before creating canonical repos.

---

## Key ADRs / Reference Links

| Resource | Location |
|----------|----------|
| Area 2 taxonomy spec | `spec/docs/superpowers/specs/2026-04-18-area2-taxonomy-design.md` |
| Phase 3 plan | `spec/docs/superpowers/plans/2026-04-21-area2-phase3-project-registry-and-mining.md` |
| Phase 4 plan | `spec/docs/superpowers/plans/2026-04-21-area2-phase4-validation-gate-and-pipeline.md` |
| Blog entry 12 | `hortora.github.io/_posts/2026-04-21-mdp01-cleaning-up-closing-the-loop.md` |
| Ecosystem plan (7 areas) | `spec/docs/design/2026-04-16-garden-ecosystem-plan.md` |

*Previous references — `git show HEAD~1:HANDOFF.md`*
