# 0004 — Aggregate State Storage: SQLite over Markdown Flat Files

Date: 2026-04-15
Status: Accepted

## Context and Problem Statement

The garden maintains three mutable aggregate state files as markdown tables: `CHECKED.md` (deduplication pair results), `DISCARDED.md` (retired entries), and the By Technology index in `GARDEN.md`. These require full-file reads for any lookup (O(n) scan) and grow without bound — `CHECKED.md` is already 4,925 lines at 240 entries. The pattern breaks at community scale: at 1,000 entries, `CHECKED.md` could hold hundreds of thousands of rows; concurrent CI writes produce git merge conflicts on shared files.

## Decision Drivers

* `load_checked_pairs()` reads and regex-scans the entire `CHECKED.md` on every call — O(n) for a lookup that should be O(1)
* Markdown tables cannot be written concurrently without git conflicts
* At community scale (1,000+ entries), `CHECKED.md` becomes operationally unworkable
* Git diff readability on CHECKED.md is a nice-to-have but not a hard requirement — commit messages capture DEDUPE session summaries; SQL queries answer specific lookups better than grep on a diff
* No new runtime dependencies: `sqlite3` is Python stdlib since 2.5
* Clear upgrade path to distributed SQLite (Turso/libSQL) when write concurrency demands it

## Considered Options

* **Option A** — SQLite (`garden.db`) for all mutable aggregate state
* **Option B** — Per-domain sharded markdown files (`java/CHECKED.md`, etc.)
* **Option C** — JSON key-value file (`CHECKED.json`)
* **Option D** — Git refs as key-value store (`refs/garden/checked/<pair-hash>`)

## Decision Outcome

Chosen option: **Option A — SQLite**, because it is the only option that actually solves the scaling problem rather than deferring it.

Options B and C both maintain O(n) scan characteristics, just with smaller files. Option D (git refs) was considered as a complement to SQLite for simple k-v cases, but was rejected on the same grounds: git's ref store was not designed as a general-purpose k-v store. At 1,000 entries with 10% pair check coverage, git refs would hold ~50,000 entries (manageable); at 5,000 entries with any meaningful coverage, millions of refs cause `git status`, `git fetch`, `git gc`, and `git clone` to slow proportionally — the same ceiling as CHECKED.md, just higher. Git refs are designed for branch and tag management (hundreds to low thousands of refs), not aggregate storage at scale.

SQLite delivers O(log n) lookup on a B-tree index, ACID transactional writes, scales to 100M+ rows practically, and requires no new dependencies. It is the correct tool for mutable structured state.

### Positive Consequences

* Pair lookup: O(log n) on `pair_hash` index, sub-millisecond at millions of rows
* Concurrent CI writes: SQLite WAL mode serialises writers without git conflicts
* `GARDEN.md` becomes metadata-only (7 lines) — no index rewriting on every merge
* `entries_index` enables SQL queries (WHERE domain, ORDER BY score) that markdown tables cannot support
* `garden.db` is committed to git and pushed like any other file — one binary, not thousands of refs
* Upgrade path: `sqlite3.connect()` → `libsql.connect()` (Turso) with zero schema or query changes

### Negative Consequences / Tradeoffs

* `garden.db` is a binary file — `git diff` shows noise without a textconv driver
* Textconv mitigation: add `.gitattributes` entry (`garden.db diff=sqlite3`) and configure `[diff "sqlite3"] textconv = sqlite3` — restores readable SQL dump diffs
* Existing scripts require migration to SQLite API (bounded, mechanical effort)
* One-time migration of `CHECKED.md` and `DISCARDED.md` content into `garden.db`

## Pros and Cons of the Options

### Option A — SQLite

* ✅ O(log n) lookup — scales to 100M+ rows
* ✅ ACID transactions — no corruption from concurrent writes
* ✅ `sqlite3` stdlib — no new dependencies
* ✅ SQL queries for `entries_index` (WHERE, ORDER BY, aggregations)
* ✅ Turso/libSQL upgrade path with no code changes
* ❌ Binary file — requires textconv config for readable git diffs

### Option B — Per-domain sharded markdown

* ✅ Smaller files, git-friendly diffs
* ✅ No new dependencies
* ❌ Still O(n) scan — defers, does not solve the scaling problem
* ❌ Cross-domain queries require loading multiple files

### Option C — JSON key-value file

* ✅ O(1) dict lookup after full parse
* ✅ Git-diffable text
* ❌ Full parse on every process startup — no partial read
* ❌ Concurrent writes corrupt the file
* ❌ No query capability

### Option D — Git refs as key-value store

* ✅ Zero extra dependencies — git already required
* ✅ Native sync via `git push refs/garden/*`
* ❌ Git ref store not designed for 100K+ refs — `git status`, `git fetch`, `git gc` degrade
* ❌ At 5,000 entries with meaningful DEDUPE coverage: millions of refs, same ceiling as CHECKED.md just higher
* ❌ No query capability — cannot support `entries_index` requirements
* ❌ `git update-ref` subprocess per write — batch mode (`--stdin`) mitigates but adds complexity
* ❌ Custom refs not fetched by default clone — requires explicit refspec configuration

## Schema

```sql
CREATE TABLE checked_pairs (
    pair_hash  TEXT PRIMARY KEY,   -- hex(sha256(canonical_pair)[:8])
    pair       TEXT NOT NULL,      -- "GE-XXXX × GE-YYYY" canonical form
    result     TEXT NOT NULL,      -- distinct | related | duplicate-discarded
    checked_at TEXT NOT NULL,      -- ISO 8601 date
    notes      TEXT DEFAULT ''
);

CREATE TABLE discarded_entries (
    ge_id          TEXT PRIMARY KEY,
    conflicts_with TEXT NOT NULL,
    discarded_at   TEXT NOT NULL,
    reason         TEXT DEFAULT ''
);

CREATE TABLE entries_index (
    ge_id               TEXT PRIMARY KEY,
    title               TEXT NOT NULL,
    domain              TEXT NOT NULL,
    type                TEXT NOT NULL,
    score               INTEGER NOT NULL,
    submitted           TEXT NOT NULL,
    staleness_threshold INTEGER NOT NULL DEFAULT 730,
    tags                TEXT DEFAULT '[]',  -- JSON array
    verified_on         TEXT DEFAULT '',
    last_reviewed       TEXT DEFAULT '',
    file_path           TEXT NOT NULL       -- domain/GE-XXXX.md
);

CREATE INDEX idx_entries_domain    ON entries_index(domain);
CREATE INDEX idx_entries_score     ON entries_index(score DESC);
CREATE INDEX idx_entries_submitted ON entries_index(submitted);

CREATE TABLE schema_version (
    version INTEGER NOT NULL,
    applied_at TEXT NOT NULL
);
INSERT INTO schema_version VALUES (1, date('now'));
```

`entries_index` is derived — can be fully rebuilt from `git ls-tree` + frontmatter parsing if `garden.db` is lost. `checked_pairs` and `discarded_entries` are operational state and cannot be reconstructed from entry files; they must be committed and pushed.

## Git Diff Support

Add to garden root `.gitattributes`:
```
garden.db diff=sqlite3
```

Add to local git config (or document for contributors; `init_garden.py` can automate this):
```ini
[diff "sqlite3"]
    textconv = sqlite3
    cachetextconv = true
```

With this, `git diff HEAD garden.db` produces a readable SQL dump showing exactly which rows changed. `cachetextconv = true` caches the conversion for repeated diffs.

Commit messages for DEDUPE sessions provide a sufficient human-readable summary regardless:
```
dedupe: sweep 47 pairs — 30 distinct, 12 related, 5 discarded (java domain)
```

## Migration Path

1. `garden_db.py init <garden_path>` — creates `garden.db` with schema
2. `garden_db.py migrate <garden_path>` — reads `CHECKED.md` and `DISCARDED.md`, inserts all rows, renames originals to `.bak`
3. `garden_db.py rebuild-index <garden_path>` — scans all entry files via `git ls-tree`, populates `entries_index`
4. `validate_garden.py` gains `--check-db` flag — verifies schema version and referential integrity
5. `dedupe_scanner.py`, `integrate_entry.py`, `validate_garden.py`, `harvest SKILL.md` all updated to use `garden_db.py` API

## Turso Upgrade Path

When community-scale concurrent writes require distributed SQLite:
```python
# Local (current)
import sqlite3
conn = sqlite3.connect(str(garden / 'garden.db'))

# Turso (future — zero query changes)
import libsql_experimental as libsql
conn = libsql.connect(database="libsql://...", auth_token="...")
```

Schema, queries, and all script logic are identical. The connection string is the only change.

## Links

* Supersedes current markdown-table approach for `CHECKED.md`, `DISCARDED.md`, and `GARDEN.md` index
* Relates to ADR-0001 (index-and-lazy-reference — entry files stay in git)
* Garden health check: `spec/docs/garden-health-2026-04-15.md` — CHECKED.md scaling concern
* Implementation: `scripts/garden_db.py` (new module), `scripts/garden_db_migrate.py`
