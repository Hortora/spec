# Hortora Garden — Design Snapshot
**Date:** 2026-04-13
**Topic:** Garden entry format — individual files with YAML frontmatter
**Supersedes:** *(none)*
**Superseded by:** *(leave blank — filled in if this snapshot is later superseded)*

---

## Where We Are

The garden has completed its structural migration: all 177 entries now live as
individual `GE-XXXX.md` files with YAML frontmatter rather than packed together
in multi-entry `.md` files. Every entry has a machine-readable header (`id`,
`title`, `type`, `domain`, `stack`, `tags`, `score`, `verified`,
`staleness_threshold`, `submitted`). The migration script
(`soredium/scripts/migrate_legacy_entries.py`) is committed and reusable for
any future entries still in legacy format. The live garden is at
`~/.hortora/garden/` and the canonical remote is `Hortora/garden`.

## How We Got Here

| Decision | Chosen | Why | Alternatives Rejected |
|---|---|---|---|
| GE-ID format | Date + 6 hex (`GE-YYYYMMDD-xxxxxx`) | No coordination required; negligible collision risk at realistic scale | Sequential counter (coordination required); GitHub Issues (doesn't scale) |
| Entry storage | Individual `GE-XXXX.md` files | Machine-readable; enables per-entry CI validation; unambiguous authorship | Multi-entry topic files (parsing fragile; can't validate per-entry) |
| YAML frontmatter fields | `id`, `title`, `type`, `domain`, `stack`, `tags`, `score`, `verified`, `staleness_threshold`, `submitted` | Captures enough metadata for filtering, staleness management, and CI | Inline markdown headers only (not machine-readable) |
| CI pipeline | `validate_pr.py` on PR, `integrate_entry.py` on merge | Validates before merge; updates indexes automatically | Manual review only |
| Garden location | `~/.hortora/garden/` via `$HORTORA_GARDEN` | No tool-specific path assumptions baked in | `~/claude/knowledge-garden/` (fragile, tool-specific) |

See [ADR-0003](../docs/adr/0003-ge-id-scheme-date-plus-random-hex.md) for the
GE-ID scheme decision in full.

## Where We're Going

The garden format is stable. Remaining work is tooling and content:

**Next steps:**
- `soredium/scripts/migrate_legacy_entries.py` — committed; could be extended
  to also update `_summaries/`, `_index/`, `labels/` for migrated entries
- Remove legacy `garden` skill source from cc-praxis (low priority)
- Integrate entries from merged PRs into GARDEN.md automatically (currently
  manual after PR merge — `integrate_entry.py` must be run by hand)

**Open questions:**
- Should `integrate_entry.py` run as a CI step post-merge, or remain manual?
  Currently manual; three merged entries were missing from the index until
  caught by `validate_garden.py` during the migration.
- The `staleness_threshold` field exists in every entry but nothing enforces
  it — stale entries are not flagged in SEARCH results. Mechanism TBD.
- `_summaries/`, `_index/global.md`, and `labels/` dirs exist but are not
  populated from the migrated legacy entries — only new-format entries run
  through `integrate_entry.py`. Full index population deferred.

## Linked ADRs

| ADR | Decision |
|---|---|
| [ADR-0001](../docs/adr/0001-index-and-lazy-reference-pattern.md) | GARDEN.md as sparse index; entry files loaded on demand |
| [ADR-0002](../docs/adr/0002-ci-script-location-and-soredium-visibility.md) | CI scripts live in soredium, not garden |
| [ADR-0003](../docs/adr/0003-ge-id-scheme-date-plus-random-hex.md) | GE-ID: date + 6 random hex chars |

## Context Links

- Migration script: `~/claude/hortora/soredium/scripts/migrate_legacy_entries.py`
- Garden remote: https://github.com/Hortora/garden
- Live garden: `~/.hortora/garden/`
