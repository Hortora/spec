# Phase 4 — Full Deduplication System Design

## Goal

Complete the deduplication system with two components: (1) drift counter infrastructure that auto-increments on every integration and warns when DEDUPE is overdue, and (2) a Jaccard pre-scoring scanner that prioritises duplicate candidates for Claude review and owns CHECKED.md state.

## Approach

Approach A as the backbone, with selected concepts from B and C:
- **A**: `integrate_entry.py` increments counter; `dedupe_scanner.py` is a standalone script; harvest calls it
- **B**: scanner owns CHECKED.md — reads to filter, writes via `--record`; harvest no longer touches CHECKED.md directly
- **C**: `validate_garden.py --dedupe-check` verifies drift counter against git log as a safety net

## Components

### 1. Drift Counter in `integrate_entry.py`

After a successful entry integration, read `Entries merged since last sweep` from GARDEN.md, increment by 1, write back. Included in the same atomic commit as the index update.

**GARDEN.md field:** `**Entries merged since last sweep:** N` (already exists, currently not auto-incremented)

### 2. `validate_garden.py --dedupe-check <garden_root>`

Early-exit flag (same pattern as `--structural`, `--freshness`):

1. Read drift counter from GARDEN.md header
2. Read drift threshold from GARDEN.md header
3. Cross-check against git log: count commits with message matching `^index: integrate` since the last commit matching `^dedupe:` — this is the authoritative count if counter and log diverge
4. If git-log count > GARDEN.md counter: use git-log count (counter got out of sync)
5. Exit 0 if drift < threshold; exit 2 with message if drift ≥ threshold

**Message format:**
```
Dedupe check: drift=N, threshold=T — DEDUPE recommended
```
or:
```
Dedupe check: drift=N, threshold=T — OK
```

### 3. `scripts/dedupe_scanner.py`

New standalone script. Single responsibility: compute Jaccard similarity for unchecked within-domain entry pairs, ranked highest-first, and manage CHECKED.md state.

**CLI interface:**
```
dedupe_scanner.py [garden_root]                          # all domains, all unchecked pairs
dedupe_scanner.py [garden_root] --domain quarkus         # limit to one domain
dedupe_scanner.py [garden_root] --top 20                 # limit to top N by score
dedupe_scanner.py [garden_root] --record "GE-X × GE-Y" <result> "<note>"
dedupe_scanner.py [garden_root] --json                   # machine-readable output
```

**Jaccard algorithm:**
- Tokenizer: 3+ char lowercase words extracted via `\b[a-z]{3,}\b` (matches `validate_pr.py`)
- Text compared: `title` + `tags` (space-joined) from YAML frontmatter
- Formula: `len(tokens_a & tokens_b) / len(tokens_a | tokens_b)`
- Similarity thresholds (informational only — not a gate):
  - ≥ 0.4: high similarity — likely related or duplicate
  - 0.2–0.4: moderate similarity — worth reviewing
  - < 0.2: low similarity — probably distinct

**Pair generation:**
- Scan domain directories directly for `.md` files with YAML frontmatter (skip GARDEN.md, CHECKED.md, DISCARDED.md, INDEX.md, and any file without `---` frontmatter). This is more robust than reading GARDEN.md By Technology — it finds entries regardless of index completeness.
- Group entries by their `domain` YAML field (not directory name, to handle legacy entries correctly)
- Read CHECKED.md to build excluded-pairs set (both orderings: A×B and B×A)
- Generate all within-domain pairs not in excluded set
- Compute Jaccard for each, sort descending by score

**CHECKED.md management (`--record`):**
- Canonical ID ordering: lower ID first (legacy GE-NNNN before new GE-YYYYMMDD-xxxxxx; within same format, lexicographic)
- Appends: `| GE-X × GE-Y | <result> | YYYY-MM-DD | <note> |`
- Results: `distinct`, `related`, `duplicate-discarded`
- Idempotent: will not record the same pair twice

**Output format (default):**
```
DOMAIN: quarkus  (4 unchecked pairs)
0.72  GE-0063 × GE-0056  [CDI NullPointerException / Drools JUnit context]
0.41  GE-0063 × GE-0105  [CDI NullPointerException / Phase 2 rules]
0.18  GE-0056 × GE-0105  [Drools JUnit context / Phase 2 rules]

DOMAIN: tools  (0 unchecked pairs)
```

Exit codes: 0 always (report tool, not a gate).

### 4. harvest DEDUPE — Two Changes

**Step 1 replacement — run scanner instead of manual enumeration:**
```bash
python3 ${SOREDIUM_PATH:-~/claude/hortora/soredium}/scripts/dedupe_scanner.py \
  ${HORTORA_GARDEN:-~/.hortora/garden} --top 50
```
Claude reviews pairs highest-score-first. No manual GARDEN.md or CHECKED.md reading.

**Step 5 replacement — record via scanner instead of direct CHECKED.md append:**
```bash
python3 ${SOREDIUM_PATH:-~/claude/hortora/soredium}/scripts/dedupe_scanner.py \
  ${HORTORA_GARDEN:-~/.hortora/garden} --record "GE-XXXX × GE-YYYY" distinct "brief note"
```
Run once per pair after classifying. Scanner enforces canonical format and idempotency.

---

## Testing Strategy

### Unit Tests

**New file: `tests/test_dedupe_scanner.py`**

Jaccard function:
- Empty sets → 0.0
- Identical sets → 1.0
- Disjoint sets → 0.0
- Partial overlap → correct ratio
- Threshold boundaries: 0.2, 0.4 exact

Tokenizer:
- 3-char minimum enforced (2-char words excluded)
- Punctuation stripped
- Deduplication within entry
- Hyphenated words split correctly
- Mixed case normalised

Pair generator:
- 0 entries → 0 pairs
- 1 entry → 0 pairs
- 2 entries → 1 pair
- 3 entries → 3 pairs
- Cross-domain pairs excluded
- Already-checked pairs excluded (both orderings)

CHECKED.md reader:
- Empty file → empty set
- Single pair → set with 1 canonical entry
- Multiple pairs → full set
- Malformed rows skipped gracefully
- Both ID orderings (A×B, B×A) recognised as the same pair

`--record`:
- Appends correct pipe-delimited format
- Canonical ID ordering enforced regardless of input order
- Idempotent: recording same pair twice produces one row only
- All three results accepted: distinct, related, duplicate-discarded

**Additions to `tests/test_integrate_entry.py`**

- After successful integration: GARDEN.md drift counter = previous + 1
- Counter starts at 0 and increments correctly to 1, then 2
- GARDEN.md without counter field: handled gracefully (treated as 0, writes 1)

**Additions to `tests/test_validate_garden.py`**

- `--dedupe-check`: drift=0, threshold=10 → exit 0
- `--dedupe-check`: drift=10, threshold=10 → exit 2
- `--dedupe-check`: drift=9, threshold=10 → exit 0
- `--dedupe-check`: drift > threshold → exit 2, message contains "DEDUPE recommended"
- Git-log divergence: counter=2, git-log=4 → uses git-log count (4)
- Missing counter field: falls back to git-log count

### Integration Tests

**Additions to `tests/test_dedupe_scanner.py`** (using `GitGarden` fixture from `garden_fixture.py`):

- Scanner against fixture with 3 domains:
  - Domain A: 2 nearly-identical entries → pair appears, high score, correct sort
  - Domain B: 2 clearly distinct entries → pair appears, low score
  - Domain C: 1 entry → no pairs generated
- Scanner with `--domain`: only domain A pairs returned
- Scanner with `--top 1`: only highest-scoring pair returned
- `--record` then re-scan: recorded pair absent from subsequent scan output
- Already-checked pairs in CHECKED.md: excluded from scanner output

**Additions to `tests/test_validate_garden.py`**:

- Full integrate → drift → `--dedupe-check` cycle:
  - Create fixture garden, run `integrate_entry.py` 3 times, assert drift=3, assert `--dedupe-check` exits 0 (threshold=10)
  - Run `integrate_entry.py` 7 more times, assert drift=10, assert `--dedupe-check` exits 2
- Git-log verification: mock 4 "index: integrate" commits since last "dedupe:" commit, assert `--dedupe-check` uses 4 not GARDEN.md counter

### End-to-End Tests

**Additions to `tests/test_integration.py`**:

**Happy path 1 — related pair:**
1. Create fixture garden with 2 similar entries in same domain
2. Run `integrate_entry.py` for both → assert drift=2
3. Run `dedupe_scanner.py` → assert pair appears with score ≥ 0.4
4. Run `dedupe_scanner.py --record "GE-X × GE-Y" related "similar symptom"` → assert CHECKED.md updated
5. Run `dedupe_scanner.py` again → assert pair no longer in output
6. Run DEDUPE drift reset (set drift=0 in GARDEN.md) → run `--dedupe-check` → exit 0

**Happy path 2 — distinct pair:**
1. Create fixture garden with 2 unrelated entries in same domain
2. Run scanner → assert pair appears with score < 0.2
3. Record as distinct → assert CHECKED.md row added
4. Re-scan → assert pair absent

**Happy path 3 — bulk import:**
1. Create 4 entries in same domain, run `integrate_entry.py` 4 times
2. Assert drift=4 in GARDEN.md
3. Run `--dedupe-check` with threshold=3 → exit 2
4. Run scanner → assert 6 pairs generated (4×3/2)
5. Record all 6 as distinct → re-scan → 0 unchecked pairs

---

## File Map

**Files to create:**
- `soredium/scripts/dedupe_scanner.py`
- `soredium/tests/test_dedupe_scanner.py`

**Files to modify:**
- `soredium/scripts/integrate_entry.py` — add drift counter increment
- `soredium/scripts/validate_garden.py` — add `--dedupe-check` flag
- `soredium/tests/test_integrate_entry.py` — add drift counter tests
- `soredium/tests/test_validate_garden.py` — add `--dedupe-check` tests
- `soredium/tests/test_integration.py` — add E2E tests
- `soredium/harvest/SKILL.md` — update Step 1 and Step 5 of DEDUPE workflow
