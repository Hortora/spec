# Garden Health Check — 2026-04-15

Scan of `Hortora/garden` main branch. 240 actual entries across 13 domains.

---

## Summary

| Check | Status | Detail |
|-------|--------|--------|
| Truncated entries | ⚠ 1 found | GE-0107 — body cut off mid-sentence |
| Empty tags `[]` | ⚠ 123/240 | Legacy entries predate tag requirement |
| Legacy GE-ID format | ℹ 177/240 | GE-0NNN vs GE-YYYYMMDD-xxxxxx |
| `_summaries/` stubs | ✅ 185 — all 1-line | Correct by design |
| CHECKED.md size | ℹ 4,925 lines | Scaling concern (see ADR-0004 in progress) |
| DISCARDED.md size | ✅ 11 lines | Fine |
| Entries missing YAML frontmatter | ✅ 0 | All entries have frontmatter |

---

## Domain Breakdown

| Domain | Entries | Notes |
|--------|---------|-------|
| tools | 117 | Largest domain |
| quarkus | 63 | |
| intellij-platform | 14 | |
| java | 12 | |
| python | 6 | |
| drools | 5 | |
| claude-code | 5 | |
| beautifulsoup | 5 | |
| java-panama-ffm | 4 | |
| macos-native-appkit | 3 | |
| electron | 3 | |
| permuplate | 2 | |
| casehub-engine | 1 | |
| apache-jexl, approaches, scelight | 0 | Directories exist, no entries |

---

## Issue 1 — Truncated Entry: GE-0107

**File:** `tools/GE-0107.md` (20 lines)
**Title:** "Partial Heading Match in `str.replace()` Corrupts Markdown Headings When Heading Has a Suffix"

The file ends mid-sentence after opening a fenced code block. The body was never completed — the symptom starts, then cuts off:

```
**Symptom:** After running a script that inserts content after a markdown heading,
the heading is split mid-line. A heading like `## Title — subtitle` becomes:
```

No root cause, no fix, no why-non-obvious section. The frontmatter and title are valid.

**Action needed:** Complete the entry body, or mark as `status: incomplete` and REVISE when the fix is known.

---

## Issue 2 — Empty Tags on 123/240 Entries

Legacy entries (GE-0NNN format, predating the YAML frontmatter migration) have `tags: []`. These were migrated from the old multi-entry file format and the tags field was not populated during migration.

**Examples:**
- `tools/GE-0107.md` — `tags: []`
- `tools/GE-0177.md` — `tags: []`
- Most GE-0NNN entries

The newer GE-YYYYMMDD-xxxxxx entries (63 total) all have populated tags.

**Action needed:** A harvest REVIEW pass or a dedicated tagging session to backfill tags on legacy entries. Low urgency — tags improve search quality but entries are still retrievable without them. Best addressed alongside a staleness review.

---

## Observation — Legacy vs New ID Format

177 entries use the sequential `GE-0NNN` format (pre-ADR-0003).
63 entries use the `GE-YYYYMMDD-xxxxxx` format (ADR-0003, date + 6 hex chars).

No action required — both formats are valid and the garden tooling handles both. Mixed state is expected during the transition period.

---

## Observation — `_summaries/` Are Correct

185 summary stub files in `_summaries/`, each exactly 1 line in the format:
```
GE-0004: [technique, 11/15] Use typed element return in generator/predicate pairs... | java-dsl
```

This is the designed compact summary format for the retrieval index. The 1-line count is correct behaviour, not truncation.

---

## Observation — CHECKED.md Scaling Concern

`CHECKED.md` is 4,925 lines — each line is a pair comparison result from previous DEDUPE sweeps. At the current 240-entry count this is manageable, but the file grows O(n²) in entries. ADR-0004 (SQLite aggregate state) is in progress to address this before it becomes operational.

See: `spec/docs/adr/` — ADR-0004 draft pending approval.

---

## Not Checked (Future)

- Staleness: entries past `staleness_threshold` — run `validate_garden.py --freshness ~/.hortora/garden` to get current count
- Deduplication drift counter: check `GARDEN.md` for current drift vs threshold
- Cross-reference integrity: `See also: GE-XXXX` references pointing to non-existent entries
- Score distribution: entries below the ≥8 threshold (should not exist post-CI, but worth verifying for legacy entries)
- Missing `_summaries/` files: entries without a corresponding summary stub
