# Garden Health Check — 2026-04-15

Scan of `Hortora/garden` main branch. 240 actual entries across 13 domains.

---

## Summary

| Check | Status | Detail |
|-------|--------|--------|
| Truncated entries | ⚠ 1 found | GE-0107 — body cut off mid-sentence |
| Empty tags `[]` | ⚠ 123/240 | Legacy entries predate tag requirement |
| **domain field too fine-grained** | ⚠ **All 240** | **Must migrate to coarse values (jvm/python/tools etc.) — see 2026-04-16 field design decision** |
| **root_cause_layer field missing** | ℹ All 240 | New optional field — can be added progressively |
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

**Action needed:** Run `garden_tag_backfill.py` (to be built — see plan below). Low urgency — tags improve search quality but entries are still retrievable without them.

### Retrospective tag backfill plan

The content to infer from is already present in each entry: `title`, `stack`, and the body text. A rule-based script gets 80% of the way without any LLM call.

**Script: `soredium/scripts/garden_tag_backfill.py`**

For each legacy entry with `tags: []`:

1. **Extract candidates from `title` + `stack`** — split on spaces, commas, backticks, version strings. Map to known tags. Examples:
   - `stack: "Hibernate ORM 6.x, Quarkus"` → `[hibernate, jpa, quarkus]`
   - `title: "macOS BSD sed silently ignores word boundaries"` → `[sed, macos, regex, bsd]`

2. **Normalise against existing vocabulary** — build a frequency list from the 63 new-format entries (which all have good tags). Only emit tags already in use; don't invent new ones. Common tags observed: `hibernate`, `jpa`, `quarkus`, `testing`, `regex`, `git`, `macos`, `java`, `python`, `async`, `ci-cd`, `sql`, `jvm`, `logging`, `performance`, `debugging`.

3. **Extract symptom-type terms from body** — scan for patterns like `silent failure`, `race condition`, `truncated`, `ClassCastException`, etc. to add cross-cutting tags.

4. **`--dry-run` mode** — print proposed tag changes per file, no writes. Human reviews output before anything is touched.

5. **Commit per domain or in a single batch** — each commit message references which entries were updated.

**What rule-based misses (needs manual pass):**
- Conceptual tags (`strategy`, `debugging`, `performance`) not visible in title/stack
- Synonym mappings that need a lookup table (e.g. "Hibernate" → add `jpa`)
- Ambiguous entries where the title doesn't make the tech domain obvious

**Estimated effort:** one session to write + test the script, ~30 min human review of dry-run output for 123 entries, one commit per domain.

**Dependency:** existing tag vocabulary from new-format entries. Build vocabulary list first by running `garden_web_data.py` and extracting all unique non-empty tags.

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

## Issue 3 — domain field migration required (all 240 entries)

**Decision (2026-04-17):** The `domain` field must be coarse (Qdrant partition key only),
not fine-grained technology classification. Fine-grained attribution is impossible to do
reliably — a Drools bug running on Quarkus may live in the Drools engine, the
quarkus-drools extension, or CDI lifecycle, and is often indeterminate at capture time.

**Migration map:**
```
quarkus, java, drools, java-panama-ffm, casehub-engine, permuplate, apache-jexl → jvm
python, beautifulsoup → python
tools, claude-code, intellij-platform, electron, macos-native-appkit, scelight, approaches → tools
```

Technology precision moves to `stack` (already present as free text) and `tags` (being
backfilled — see Issue 2). New optional field `root_cause_layer` added for when the
layer is known with confidence.

**What also needs updating:**
- `forage/SKILL.md` — "Determine the domain" table rewritten to coarse domains;
  `root_cause_layer` added as optional field
- `forage/submission-formats.md` — entry templates updated

**Action needed:** Write `garden_migrate.py` — a single consolidated migration script
covering all four operations below. Run once; one PR; clean git history.

### Consolidated migration: `garden_migrate.py`

**Phase 1 — Rule-based (fast, no LLM cost):**

1. **Remap `domain`** to coarse values (quarkus → jvm, etc.)

2. **Backfill `tags`** from title + stack using rule-based vocabulary match.
   Build vocabulary from existing non-empty tags in 63 new-format entries first.

3. **Add `root_cause_layer: ""`** as empty optional field — populated progressively
   as knowledge grows.

**Phase 2 — LLM-assisted (one-off enrichment, costs tokens but high value):**

4. **Reconstruct WHY fields** from existing entry body text. The prose is already
   there — the LLM extracts it into structured fields:

   | Field | Extracted from |
   |---|---|
   | `rationale` | `### Why this is non-obvious` + `### Fix` sections |
   | `alternatives_considered` | `### What was tried (didn't work)` section |
   | `constraints` | Context sentences ("only when...", "only affects...") in body |
   | `invalidation_triggers` | Version/stack hints ("fixed in vX", "only on macOS") |

   These fields were added to forage CAPTURE to prevent the "red hat bureaucracy"
   problem — decisions without context, recreating the same reasoning every session.
   Existing entries already contain this information in prose; migration extracts it
   into retrievable structured form.

   Only populate fields with clear signal. Leave blank when ambiguous. Run
   `--dry-run` and human-review a sample before full corpus run.

**Script interface:**
```bash
garden_migrate.py ~/.hortora/garden \
  --remap-domain \
  --backfill-tags --tag-vocab /tmp/vocab.txt \
  --add-root-cause-layer \
  --extract-why-fields --llm-model claude-3-5-haiku \  # Phase 2
  --dry-run     # always review first
```

**Note:** Directory structure (`java/`, `quarkus/`, `tools/`) stays for GitHub
browsability. YAML `domain` field and directory name are decoupled.

---

## Not Checked (Future)

- Staleness: entries past `staleness_threshold` — run `validate_garden.py --freshness ~/.hortora/garden` to get current count
- Deduplication drift counter: check `GARDEN.md` for current drift vs threshold
- Cross-reference integrity: `See also: GE-XXXX` references pointing to non-existent entries
- Score distribution: entries below the ≥8 threshold (should not exist post-CI, but worth verifying for legacy entries)
- Missing `_summaries/` files: entries without a corresponding summary stub
