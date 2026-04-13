# Hortora / spec

Open protocol specification for **Hortora** — a governed, federated knowledge garden for AI coding assistants.

Hortora solves a specific problem: developers rediscover the same non-obvious bugs, workarounds, and undocumented behaviours repeatedly across projects. Hortora captures them once, organises them with a quality lifecycle, and federates them across machines and teams via GitHub-backed canonical gardens.

---

## What's in this repo

### Design

[`docs/design/2026-04-07-garden-rag-redesign-design.md`](docs/design/2026-04-07-garden-rag-redesign-design.md)

The full protocol specification. Covers:

- **Entry format** — YAML frontmatter + markdown body, one file per entry
- **Retrieval algorithm** — three-tier (by technology → by symptom → full scan)
- **Deduplication** — three-level Jaccard similarity (L1/L2/L3)
- **Quality lifecycle** — Active → Suspected → Superseded → Retired
- **GitHub backend** — PR-based submission, CI validation; GE-IDs are `GE-YYYYMMDD-xxxxxx` (date + 6 random hex, generated locally — no GitHub Issues)
- **Federation protocol** — canonical / child / peer garden relationships
- **Implementation roadmap** — nine phases from data migration to hosted federation

Garden path is configurable via `HORTORA_GARDEN` env var; defaults to `~/.hortora/garden`.

### Architecture Decision Records

[`docs/adr/`](docs/adr/)

- [ADR-0001](docs/adr/0001-index-and-lazy-reference-pattern.md) — Index-and-lazy-reference pattern for session continuity
- [ADR-0002](docs/adr/0002-ci-script-location-and-soredium-visibility.md) — CI scripts live in soredium; soredium is public
- [ADR-0003](docs/adr/0003-ge-id-scheme-date-plus-random-hex.md) — GE-ID scheme: date + 6 random hex chars

### Design Snapshots

Point-in-time design records — consolidated into this README's Status section.

### Blog

[`docs/blog/`](docs/blog/)

The founding narrative. Also published at [hortora.github.io/blog](https://hortora.github.io/blog).

---

## Related repositories

| Repo | Purpose |
|------|---------|
| [Hortora/garden](https://github.com/Hortora/garden) | The live canonical garden — 177 entries, CI-validated |
| [Hortora/soredium](https://github.com/Hortora/soredium) | Claude skills (forage, harvest), validators, CI scripts |
| [Hortora/hortora.github.io](https://github.com/Hortora/hortora.github.io) | Public website |

---

## Status

Phase 2 **complete and live** (as of 2026-04-10). The garden is a live GitHub repo (`Hortora/garden`) with CI validation on every PR and automated index maintenance on merge. `forage` and `harvest` skills are deployed and in active use.

**Current garden:** 177 entries · individual `GE-*.md` files with YAML frontmatter · `~/.hortora/garden` · legacy symlink `~/claude/knowledge-garden` preserved

**Migration complete:** All entries now in individual `GE-XXXX.md` files with YAML frontmatter (`id`, `title`, `type`, `domain`, `stack`, `tags`, `score`, `verified`, `staleness_threshold`, `submitted`). Migration script: `soredium/scripts/migrate_legacy_entries.py`. Note: `_summaries/`, `_index/global.md`, and `labels/` are not yet populated from migrated legacy entries — only new-format entries run through `integrate_entry.py` get full index coverage.

**Next steps:**
- Update CI workflows in `Hortora/garden` for new GE-ID format and PR-only flow
- Test forage GitHub mode end-to-end (submit → CI validate → merge)
- Run `integrate_entry.py` as CI post-merge step (currently manual — missed entries caught by `validate_garden.py`)
- Extend `migrate_legacy_entries.py` to also update `_summaries/`, `_index/`, `labels/` for migrated entries
- Implement staleness enforcement (see `IDEAS.md` — 6 candidate solutions)
- Deprecate legacy `garden` skill in cc-praxis once forage+harvest validated
- Phase 3: Claude-in-CI for borderline score PRs; PyPI graduation for soredium

**Open questions:**
- GE-ID transition: old entries (GE-0001–GE-0172) and new entries (GE-YYYYMMDD-xxxxxx) coexist; long-term index handling TBD
- `integrate_entry.py` as CI post-merge vs manual: three merged entries were missing from index until caught manually
- `_summaries/`, `_index/global.md`, `labels/` not populated from migrated legacy entries — full index population deferred
- Staleness enforcement: `staleness_threshold` field exists in every entry but nothing enforces it; which of the 6 solutions in `IDEAS.md` to implement first
- Legacy `garden` skill deprecation timing
