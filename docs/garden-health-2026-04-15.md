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

**Action needed:** Write `garden_migrate.py` — a single consolidated migration script.
Run once; one PR; clean git history.

### What each phase improves

**Critical distinction established by literature review (2026-04-17):**

SPLADE and BM25 retrieval optimise for term overlap and semantic similarity — they
find entries based on symptom, stack, tags, root cause. WHY fields (rationale,
alternatives_considered, why_non_obvious) are largely invisible to the retriever.

This means the two phases address different problems:

| Phase | Improves | How |
|---|---|---|
| Phase 1 | **Retrieval quality** — finding the right entry | Correct domain partitioning, populated tags, accurate stack matching |
| Phase 2 | **Comprehension quality** — what the LLM does with the retrieved entry | Structured WHY context so the LLM understands reasoning, not just the fix |

Phase 1 is higher priority: it determines whether the right entry is found at all.
Phase 2 enriches what happens after retrieval — the LLM applies the fix with
understanding rather than cargo-culting it. Both matter; Phase 1 comes first.

---

### Phase 1 — Rule-based, retrieval improvement (HIGH priority, no LLM cost)

1. **Remap `domain`** to coarse values (quarkus → jvm, etc.)
   — fixes Qdrant partition routing, prevents misrouted queries

2. **Backfill `tags`** from title + stack using rule-based vocabulary match.
   Build vocabulary from existing non-empty tags in 63 new-format entries first.
   — tags are what SPLADE matches on; empty tags are invisible to retrieval

3. **Add `root_cause_layer: ""`** as empty optional field — populated progressively.
   — structural only; no retrieval impact until populated

---

### Phase 2 — LLM-assisted, comprehension improvement (LOWER priority, costs tokens)

4. **Reconstruct WHY fields** from existing entry body text.

   These fields do NOT improve retrieval — SPLADE/BM25 don't use them to find entries.
   They improve what the LLM does AFTER retrieval: understanding why a fix works,
   not just what the fix is. This prevents cargo-culting (blindly copying a fix
   without understanding its conditions) and prevents re-deriving the same reasoning
   every session ("red hat bureaucracy with agentic powers").

   The prose is already in existing entries — the LLM extracts it into structured form:

   | Field | Extracted from | What it prevents |
   |---|---|---|
   | `rationale` | `### Why this is non-obvious` + `### Fix` | Cargo-culting the fix without understanding why |
   | `alternatives_considered` | `### What was tried (didn't work)` | Re-trying approaches already shown not to work |
   | `constraints` | Context sentences in body | Applying the fix in the wrong environment |
   | `invalidation_triggers` | Version/stack hints in body | Using stale knowledge past its validity |

   Only populate fields with clear signal in the body. Leave blank when ambiguous.
   Run `--dry-run` and human-review a sample (10–20 entries) before full corpus run.

---

**Script interface:**
```bash
# Phase 1 only (do this first — fixes retrieval immediately)
garden_migrate.py ~/.hortora/garden \
  --remap-domain \
  --backfill-tags --tag-vocab /tmp/vocab.txt \
  --add-root-cause-layer \
  --dry-run

# Phase 2 (run after Phase 1 is merged — enriches comprehension)
garden_migrate.py ~/.hortora/garden \
  --extract-why-fields --llm-model claude-3-5-haiku \
  --dry-run
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
