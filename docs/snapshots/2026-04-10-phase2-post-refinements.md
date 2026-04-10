# Hortora — Design Snapshot
**Date:** 2026-04-10
**Topic:** Post-Phase-2 refinements — path, ID scheme, tooling
**Supersedes:** [2026-04-09-phase2-github-backend-complete](2026-04-09-phase2-github-backend-complete.md)
**Superseded by:** *(leave blank)*

---

## Where We Are

Phase 2 (CI-based submission pipeline) is complete and live. This session
delivered three significant refinements: (1) the garden is now path-configurable
via `HORTORA_GARDEN`, defaulting to `~/.hortora/garden` (no tool-specific path
assumptions); (2) GE-IDs are now date + 6 random hex chars (`GE-YYYYMMDD-xxxxxx`)
— no counter, no coordination, federation-safe; (3) GitHub Issues have been
removed from the forage capture flow — PRs only. The garden has 169+ entries.

## How We Got Here

| Decision | Chosen | Why | Alternatives Rejected |
|---|---|---|---|
| Garden default path | `~/.hortora/garden` | No "claude" assumption, standard hidden-dir convention | `~/claude/garden` (tool-specific), `~/garden` (too generic) |
| Path configurability | `HORTORA_GARDEN` env var | Zero-code override, works in all shell contexts | Hardcoded migration, config file |
| GE-ID scheme | `GE-YYYYMMDD-xxxxxx` (ADR-0003) | No coordination, federation-safe, ~0.003% collision at 1k/day | Sequential counter (race conditions), UUID (unreadable), GitHub Issues (doesn't scale to millions) |
| GitHub Issues in forage | Removed entirely — PRs only | Issues don't scale; ID now generated locally | Keep issues as governance artifact |
| Knowledge staleness | Logged in IDEAS.md with 6 solutions | Not yet decided | — |

## Where We're Going

**Next steps:**
- Update CI workflows (`validate-on-pr.yml`, `integrate-on-merge.yml` in `Hortora/garden`) for new GE-ID format and PR-only flow — currently still reference old issue-based approach
- Test forage GitHub mode end-to-end: submit real entry via PR, watch CI validate, confirm integration on merge
- Deprecate legacy `garden` skill in cc-praxis once forage+harvest validated across multiple sessions
- Implement staleness enforcement: at minimum, age annotations in forage SEARCH results (IDEAS.md solution #2); ideally harvest REVIEW mode (solution #5)
- Phase 3: Claude-in-CI for borderline score PRs; PyPI graduation for garden-engine

**Open questions:**
- CI workflows still reference old issue-based flow — not yet updated
- GE-ID transition: old entries (GE-0001–GE-0172) and new entries (GE-YYYYMMDD-xxxxxx) coexist; validator accepts both but GARDEN.md index and search experience need to handle mixed formats gracefully long-term
- Staleness enforcement: which of the 6 solutions in IDEAS.md to implement first, and in what order
- Legacy `garden` skill deprecation: timing and migration path for active project sessions still using it

## Linked ADRs

| ADR | Decision |
|---|---|
| [ADR-0001](../adr/0001-index-and-lazy-reference-pattern.md) | Garden index structure and lazy entry loading |
| [ADR-0002](../adr/0002-ci-script-location-and-soredium-visibility.md) | CI scripts in soredium; soredium public |
| [ADR-0003](../adr/0003-ge-id-scheme-date-plus-random-hex.md) | GE-ID scheme: date + 6 random hex chars |

## Context Links

- Garden: `~/.hortora/garden` (legacy symlink: `~/claude/knowledge-garden` → same)
- Staleness ideas: `spec/IDEAS.md`
- Phase 2 design: `spec/docs/design/2026-04-09-hortora-phase2-github-backend-design.md`
