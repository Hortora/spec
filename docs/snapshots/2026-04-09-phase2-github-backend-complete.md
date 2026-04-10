# Hortora — Design Snapshot
**Date:** 2026-04-09
**Topic:** Phase 2 complete — GitHub backend and CI pipeline live
**Supersedes:** [2026-04-07-hortora-knowledge-garden-design](2026-04-07-hortora-knowledge-garden-design.md)
**Superseded by:** [2026-04-10-phase2-post-refinements](2026-04-10-phase2-post-refinements.md)

---

## Where We Are

Phase 2 of the Hortora implementation is complete. The garden (`Hortora/garden`) is a live GitHub repo with CI validation on every PR and automated index maintenance on merge. Submitter Claudes open a GitHub issue to get a conflict-free GE-ID, write the entry, validate locally with `validate_pr.py`, and open a PR — CI handles everything from there. The same Python scripts run in CI and locally, so local-only mode produces identical results without GitHub. `forage` and `harvest` are deployed and synced to `~/.claude/skills/`.

## How We Got Here

Key decisions made to reach this point, in rough chronological order.

| Decision | Chosen | Why | Alternatives Rejected |
|---|---|---|---|
| Script location | soredium owns all scripts; garden CI checks out soredium | Tests and scripts colocated; garden stays pure data | Scripts in garden (breaks test ownership), PyPI now (premature) — see [ADR-0002](../adr/0002-ci-script-location-and-soredium-visibility.md) |
| soredium visibility | Public | Eliminates need for deploy keys; `GITHUB_TOKEN` covers two-repo CI checkout | Private + deploy key (unnecessary complexity) |
| Harvester mode | Claude stays local, maintainer-triggered | Simpler; semantic judgment doesn't need CI latency | Claude-in-CI for borderline PRs — deferred to Phase 3+ |
| Dual-mode operation | Same scripts, different callers (CI or local Claude) | No divergence between CI and local validation | Separate scripts per mode (drift risk) |
| GE-ID source (Phase 2) | GitHub issue number | Conflict-free, atomic, provides discussion thread | Sequential counter (race condition on concurrent PRs) |
| Sparse blobless clone | `git cat-file --batch` for entry bodies | On-demand retrieval; index always materialised; persistent local cache | Full clone (unnecessary), GitHub Contents API (slow, rate-limited) |

## Where We're Going

**Next steps:**
- Harvest the 2 pending submissions (GE-0160, GE-0161) via `/harvest`
- Resolve the GitHub Issues → GE-ID transition gap: issues start at #1 but existing entries go to GE-0161; need to either create dummy issues to offset, or use a separate counter for the transition
- Continue testing forage GitHub mode in real project sessions before fully deprecating the legacy `garden` skill
- Full DEDUPE of existing entries — existing-vs-existing pairs across tools/ (60 entries), quarkus/ (22 entries) largely unchecked

**Open questions:**
- GitHub Issues GE-ID transition: how to handle the offset between existing entry count (161) and GitHub issue numbers (starting at 1)? Options: bulk-create dummy issues, use a namespace prefix, or accept the gap and start new entries from issue #162 onward
- When is forage+harvest validated enough to deprecate the legacy `garden` skill in cc-praxis?
- Phase 3+: Claude-in-CI for borderline PRs (score 8–11) — which model, what prompt, what budget?
- PyPI graduation (soredium → `garden-engine`): when does soredium stabilise enough to publish?

## Linked ADRs

| ADR | Decision |
|---|---|
| [ADR-0001 — index and lazy reference pattern](../adr/0001-index-and-lazy-reference-pattern.md) | Garden index structure and on-demand retrieval |
| [ADR-0002 — CI script location and soredium visibility](../adr/0002-ci-script-location-and-soredium-visibility.md) | Scripts in soredium, public, two-repo CI checkout |

## Context Links

- Full design spec: `docs/design/2026-04-07-garden-rag-redesign-design.md`
- Phase 2 design: `docs/design/2026-04-09-hortora-phase2-github-backend-design.md`
- Phase 2 implementation plan: `docs/superpowers/plans/2026-04-09-phase2-github-backend.md`
- Blog: `docs/blog/2026-04-09-mdp01-designing-the-engine.md`
