# Contextual Capture — WHY Schema Enrichment Design

## Goal

Address the "red hat bureaucracy with agentic powers" problem: garden entries capture WHAT (symptom, root cause, fix) but not WHY this decision was made, what alternatives were rejected, what constraints made it the right call, or what changes would invalidate it. Without that context, entries become decontextualised prescriptions that agents follow without understanding — dangerous at scale.

## Scope

This spec covers the **entry format** side only (capturing WHY at write time). Context-aware retrieval (using WHY fields to filter/rank at read time) is a separate sub-project, to be designed after this is implemented and real data exists.

## Approach

Approach C — hybrid freetext + optional structured fields. Freetext captures the WHY immediately with no friction. Optional structured format pre-formats data for the retrieval phase. Both are valid simultaneously; the validator detects which format is present and awards the same bonus either way.

---

## Section 1: WHY Entry Schema — Three New Fields

Three optional fields added to all entry types (gotcha, technique, undocumented). All are optional — presence earns a bonus score, absence is fine.

### `constraints` (YAML frontmatter)

What conditions must hold for this fix to apply.

```yaml
# Freetext (default)
constraints: "requires Java 17+, not applicable to reactive pipelines"

# Structured (optional, for power users — pre-formats for retrieval phase)
constraints:
  - applies_when: "java.version >= 17"
    note: "uses sealed classes introduced in Java 17"
```

### `### Alternatives considered` (body section)

What else was evaluated and why rejected. Placement by entry type:
- **gotcha**: after `### Fix`, before `### Why this is non-obvious`
- **technique**: after `### The technique`, before `### Why this is non-obvious`
- **undocumented**: after `### How to use it / where it appears`, before `### Why it's not obvious`

```markdown
### Alternatives considered
- Using `try/catch` on the outer loop — catches too broadly, masks unrelated errors
- Upgrading to Spring Boot 3.x — valid but unavailable on Java 8 projects
```

### `invalidation_triggers` (YAML frontmatter)

What changes would make this entry wrong — version bumps, architectural shifts, deprecations.

```yaml
# Freetext (default)
invalidation_triggers: "revisit if Spring Boot 4.0 changes its auto-configuration model"

# Structured (optional)
invalidation_triggers:
  - library: "spring-boot"
    version: ">= 4.0"
    reason: "auto-configuration model may change"
```

### Schema addition to submission-formats.md

The three fields are added to the Optional Frontmatter Fields table alongside `verified_on` and `last_reviewed`. The `### Alternatives considered` heading is added to each entry template between `### Fix` and `### Why this is non-obvious`.

---

## Section 2: Author Field

New optional YAML frontmatter field `author` — set by forage at CAPTURE time, read by `contributors.py` to build the scoreboard.

```yaml
author: "mdp"   # initials or handle — short identifier, not a full name
```

**Source of truth hierarchy:**
1. `author` field in YAML frontmatter — used whenever present
2. `git log --follow --format="%ae" -- <path> | tail -1` — fallback when field absent and git available
3. `unknown` — when neither is available (local-only garden, no git remote)

**forage CAPTURE behaviour:** Reads `initials` from `~/.claude/settings.json`. If not set, prompts once: *"What initials should we use for your garden entries? This appears on the contributor scoreboard."* Saves to settings.json and proceeds. Sets `author: "<initials>"` in YAML frontmatter.

**Validation:** `validate_pr.py` emits a warning (not an error) if `author` is missing — submission is not blocked, but the CI output signals the contributor won't receive scoreboard credit.

---

## Section 3: Validator Bonus Strategy

The bonus is computed at validation time by `validate_pr.py` — it does **not** mutate the `score` field. The self-reported `score` remains as-is; the effective score (base + bonus) is reported in validator output and used by the scoreboard.

**Bonus rules — configurable dict at the top of validate_pr.py:**

```python
BONUS_RULES = {
    'constraints':             1,  # +1 if constraints field present (string or list, non-empty)
    'alternatives_considered': 1,  # +1 if ### Alternatives considered section has ≥1 list item
    'invalidation_triggers':   1,  # +1 if invalidation_triggers field present (string or list, non-empty)
}
# Max possible bonus: +3
```

**Detection logic:**
- `constraints`: YAML frontmatter field present, non-empty (string or non-empty list)
- `alternatives_considered`: `### Alternatives considered` heading present in body with at least one `- ` list item beneath it
- `invalidation_triggers`: YAML frontmatter field present, non-empty (string or non-empty list)

**Validator output format:**
```
Score: 12/15 base + 2 bonus = 14 effective
  ✓ constraints present
  ✓ alternatives_considered present
  ✗ invalidation_triggers missing (optional — adds +1)
```

**Threshold rule:** The ≥8 gate applies to **base score only**. An entry scoring 7/15 base does not pass because it has all three WHY fields (7 + 3 = 10 effective). This prevents gaming: documented context rewards quality, not replaces it.

**Tuning:** To adjust the bonus strategy, edit `BONUS_RULES` — no logic changes required. Rules are data, not code.

---

## Section 4: CONTRIBUTORS.md and contributors.py

New script `soredium/scripts/contributors.py` — reads all garden entries, computes effective scores per author, generates `CONTRIBUTORS.md` in the garden root.

**CLI:**
```
contributors.py [garden_root]          # generate CONTRIBUTORS.md
contributors.py [garden_root] --json   # machine-readable output
```

**Metrics computed per author:**
- Total entries submitted
- Average effective score (base + bonus, computed fresh by re-applying BONUS_RULES)
- Average bonus (how consistently WHY fields are documented)
- Best entry (highest effective score — title + GE-ID + link)

**CONTRIBUTORS.md format:**
```markdown
# Garden Contributors

Ranked by average effective score. Updated: YYYY-MM-DD.

| Rank | Author | Entries | Avg Score | Avg Bonus | Best Entry |
|------|--------|---------|-----------|-----------|------------|
| 1 | mdp | 47 | 14.2 | +2.1 | [YAML regex skips CRLF](python/GE-20260414-c12931.md) |
| 2 | xyz | 12 | 13.8 | +1.5 | [fromisoformat crash](python/GE-20260414-2a1cd1.md) |

_Avg Bonus reflects how consistently contributors document constraints,
alternatives considered, and invalidation triggers._
```

**Author resolution per entry:**
1. `author` YAML field if present
2. `git log --follow --format="%ae" -- <path> | tail -1` if git available
3. `unknown`

**When it runs:** Manually or via weekly CI schedule. Not on every `integrate_entry.py` — a periodic snapshot is sufficient and avoids O(N) slowdown on large gardens.

---

## File Map

**Files to create:**
- `soredium/scripts/contributors.py`
- `soredium/tests/test_contributors.py`

**Files to modify:**
- `soredium/scripts/validate_pr.py` — add BONUS_RULES, bonus detection logic, author warning
- `soredium/tests/test_validate_pr.py` — add bonus scoring tests, author warning test
- `soredium/forage/submission-formats.md` — add three WHY fields to Optional Frontmatter Fields table; add `### Alternatives considered` to each template; add `author` field
- `soredium/forage/SKILL.md` — CAPTURE Step 3 (add WHY fields + author to extraction table), CAPTURE Step 5 (add soft prompts for WHY fields), CAPTURE Step 6 (set author from settings.json)

**Garden file:**
- `~/.hortora/garden/CONTRIBUTORS.md` — generated by contributors.py (not a soredium file)

---

## Implementation notes

- The hybrid format detection in validate_pr.py: `isinstance(value, str)` for freetext, `isinstance(value, list)` for structured. One branch, no ambiguity.
- `BONUS_RULES` as a module-level dict means the strategy can be changed without touching detection logic — the detection functions read from it.
- `contributors.py` re-applies BONUS_RULES at generation time — effective scores are always consistent with the current rules, even if rules change after entries were submitted.
- Legacy entries (no `author` field) are attributed via git blame where possible, `unknown` otherwise. They still appear in the scoreboard under whatever the blame resolves to.
