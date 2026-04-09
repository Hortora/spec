# Hortora Phase 2 — GitHub Backend & Automation

*Date: 2026-04-09*
*Status: Approved*

---

## Goal

Contributions flow through GitHub. CI validates every submission before it reaches the garden. The same validation scripts run in CI and locally — no divergence between modes. The harvester (Claude) stays local and maintainer-triggered for Phase 2; Claude-in-CI is a future phase.

---

## Architecture

Two repos, one engine. `soredium` owns all scripts and skills. `Hortora/garden` owns entry data and GitHub Actions workflows. CI checks out both. soredium is public — `GITHUB_TOKEN` covers all CI operations, no deploy key needed.

```
soredium/
  scripts/
    validate_pr.py        ← runs in CI (on PR open) and locally (by submitter)
    integrate_entry.py    ← runs in CI (on merge) and locally (by harvester)
    validate_garden.py    ← existing, unchanged
    garden-setup.sh       ← one-time sparse blobless clone setup
  forage/SKILL.md         ← updated: GitHub mode + local mode
  harvest/SKILL.md        ← updated: local mode only (Phase 2)
  tests/
    test_validate_pr.py
    test_integrate_entry.py
    test_forage_capture.py

Hortora/garden/
  .github/workflows/
    validate-on-pr.yml    ← checks out soredium, runs validate_pr.py
    integrate-on-merge.yml ← checks out soredium, runs integrate_entry.py
```

### Three Operation Modes

Same scripts, different callers:

| Mode | Who calls scripts | ID source | Merge mechanism |
|------|-------------------|-----------|-----------------|
| **CI** | GitHub Actions, on PR/merge events | GitHub issue number | PR merge |
| **GitHub-local** | Submitter Claude, before pushing | GitHub issue number | PR merge |
| **Local-only** | Submitter Claude, directly | Sequential counter in GARDEN.md | Direct commit |

Mode is auto-detected by forage CAPTURE: if `git remote get-url origin` returns a GitHub URL → GitHub mode. Otherwise → local mode.

---

## Phased Delivery

### Phase 2.1 — Thin slice (end-to-end CI)

**Goal:** A real submission PR gets validated by CI.

- `validate_pr.py`: CRITICAL checks only (format, schema, score threshold, injection)
- GitHub Actions `validate-on-pr.yml` (two-repo checkout, runs script, posts PR comment)
- forage CAPTURE: GitHub mode — `gh issue create` → write `GE-XXXX.md` → `gh pr create`

### Phase 2.2 — Deepen validation + post-merge integration

**Goal:** Merge a PR and indexes update automatically.

- `validate_pr.py` deepened: vocabulary check, Jaccard duplicate scan
- `integrate_entry.py`: full index maintenance on merge
- GitHub Actions `integrate-on-merge.yml`
- Full test suite for both scripts

### Phase 2.3 — Local mode parity

**Goal:** Full submission flow works with no GitHub, same validation as CI.

- forage CAPTURE local mode: runs scripts directly, sequential ID, direct commit
- harvest SKILL: runs `integrate_entry.py` locally for manual merges
- Config auto-detection: GitHub remote → GitHub mode, no remote → local mode

### Phase 2.4 — Sparse blobless clone + retrieval

**Goal:** Entry retrieval never needs working tree materialisation.

- forage SEARCH updated to use `git cat-file --batch`
- `garden-setup.sh` encapsulates one-time clone for new machines
- Session-start `git pull --filter=blob:none`

---

## Script Specifications

### `validate_pr.py`

Runs on PR open. Exits non-zero on CRITICAL failures only.

**CRITICAL (auto-reject, exit 1):**
- YAML frontmatter not parseable
- Required fields missing: `title`, `type`, `domain`, `score`, `tags`, `verified`, `staleness_threshold`
- `score < 8`
- Prompt injection patterns detected (role-play instructions, system prompt overrides, jailbreak patterns)

**WARNING (human review required, exit 0 + warnings):**
- `score` 8–11
- Tags not in controlled vocabulary
- Jaccard similarity ≥ 0.4 against existing entries in same domain (computed over tokenised `title` + `tags` + `summary` fields)
- Code blocks present in Fix section

**INFO (posted to PR comment, no block):**
- `score ≥ 12` → flagged as auto-approve eligible
- Jaccard 0.2–0.4 → suggested related entries

Output: structured JSON to stdout (exit 0 = pass, may have warnings; exit 1 = CRITICAL failure). The script does NOT make GitHub API calls directly — it writes JSON only, keeping it testable without GitHub access. The GitHub Actions workflow parses stdout and posts the PR comment.

### `integrate_entry.py`

Runs on merge. Updates all indexes, then commits.

Steps in order:
1. Parse merged entry file — extract domain, GE-ID, tags, title, score
2. Generate `_summaries/<domain>/GE-XXXX.md` — one-line summary for DEDUPE pre-screen
3. Append entry to domain `INDEX.md`
4. Update `labels/` entries for each tag
5. Update `_index/global.md` if new domain
6. Close the GitHub issue: `gh issue close XXXX`
7. Run `validate_garden.py --structural` (sanity check — fails build if broken)
8. `git commit` index changes with message `index: integrate GE-XXXX`

---

## GitHub Actions Workflows

### `validate-on-pr.yml`

```yaml
on:
  pull_request:
    types: [opened, synchronize]
    paths: ['*/GE-*.md', 'submissions/**']

jobs:
  validate:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
        with:
          repository: Hortora/garden

      - uses: actions/checkout@v4
        with:
          repository: Hortora/soredium
          path: .soredium

      - run: pip install pyyaml

      - name: Detect changed entry file
        id: entry
        run: |
          FILE=$(git diff --name-only HEAD~1 HEAD | grep -E '^.*/GE-[0-9]+\.md$' | head -1)
          echo "file=$FILE" >> $GITHUB_OUTPUT

      - name: Validate entry
        run: python .soredium/scripts/validate_pr.py ${{ steps.entry.outputs.file }}

      - name: Post PR comment
        if: always()
        run: # post validate_pr.py JSON output as PR comment via gh api

      - name: Apply labels
        if: always()
        run: # apply rejected / needs-review / auto-approve-eligible based on exit code + output
```

### `integrate-on-merge.yml`

```yaml
on:
  pull_request:
    types: [closed]
    branches: [main]

jobs:
  integrate:
    if: >
      github.event.pull_request.merged == true &&
      contains(github.event.pull_request.labels.*.name, 'garden-submission')
    runs-on: ubuntu-latest
    permissions:
      contents: write
      issues: write
    steps:
      - uses: actions/checkout@v4
        with:
          repository: Hortora/garden
          fetch-depth: 0

      - uses: actions/checkout@v4
        with:
          repository: Hortora/soredium
          path: .soredium

      - run: pip install pyyaml

      - name: Detect merged entry file
        id: entry
        run: |
          FILE=$(git show --name-only --pretty="" HEAD | grep -E '^.*/GE-[0-9]+\.md$' | head -1)
          echo "file=$FILE" >> $GITHUB_OUTPUT

      - name: Integrate entry
        run: python .soredium/scripts/integrate_entry.py ${{ steps.entry.outputs.file }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Push index updates
        run: git push origin main
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## forage CAPTURE — Updated Workflow

### GitHub Mode

```
1. Draft entry content (existing CAPTURE logic — scoring, Fix section, tags)
2. gh issue create --repo Hortora/garden \
     --title "<slug>" \
     --label "garden-submission" \
     --body "Type: <type> | Score: <n>/15 | Domain: <domain>"
   → issue number becomes GE-ID (e.g. issue #123 → GE-0123)
3. git -C ~/claude/knowledge-garden checkout -b submit/GE-XXXX
4. Write GE-XXXX.md with complete YAML frontmatter
5. python $SOREDIUM_PATH/scripts/validate_pr.py GE-XXXX.md
   → fast local feedback before push; fix any CRITICAL issues
   → SOREDIUM_PATH defaults to ~/claude/hortora/soredium; configurable via env var
6. git -C ~/claude/knowledge-garden push origin submit/GE-XXXX
7. gh pr create --repo Hortora/garden \
     --title "submit(GE-XXXX): <slug>" \
     --body "Closes #XXXX" \
     --label "garden-submission"
8. Done — CI validates, maintainer reviews, CI integrates on merge
```

### Local Mode

```
1. Draft entry content (same)
2. Read next sequential ID from GARDEN.md counter
3. Write GE-XXXX.md with complete YAML frontmatter
4. python $SOREDIUM_PATH/scripts/validate_pr.py GE-XXXX.md
   → same validation as CI; fix any CRITICAL issues
5. python $SOREDIUM_PATH/scripts/integrate_entry.py GE-XXXX.md
   → same index maintenance as CI
6. git -C ~/claude/knowledge-garden add -p && git commit
7. Done — no PR, no CI, indexes already updated
```

---

## Sparse Blobless Clone (Phase 2.4)

### One-time Setup (`garden-setup.sh`)

```bash
git clone --filter=blob:none --no-checkout \
  https://github.com/Hortora/garden.git ~/claude/knowledge-garden
cd ~/claude/knowledge-garden
git sparse-checkout init
git sparse-checkout set \
  SCHEMA.md GARDEN.md CHECKED.md \
  "_index/" "_summaries/" "labels/" \
  "*/INDEX.md" "*/README.md"
git checkout main
```

Index files (always materialised) → read via Read tool as before.
Entry bodies → never materialised in working tree.

### Session Start

```bash
git -C ~/claude/knowledge-garden pull --filter=blob:none
```

### forage SEARCH — On-demand Entry Reads

```bash
# Single entry
git -C ~/claude/knowledge-garden cat-file blob HEAD:quarkus/cdi/GE-0123.md

# Batch (Tier 2 retrieval — 2-4 candidates, one round-trip)
printf 'HEAD:quarkus/cdi/GE-0123.md\nHEAD:tools/git/GE-0043.md\n' \
  | git -C ~/claude/knowledge-garden cat-file --batch
```

Subsequent reads of the same blob are served from `.git/objects/` — no network.

---

## Testing

All tests in `soredium/tests/`. Full suite (currently 118 tests) stays green throughout each phase.

### `test_validate_pr.py`
- CRITICAL: missing required fields → exit 1
- CRITICAL: `score < 8` → exit 1
- CRITICAL: injection patterns → exit 1
- WARNING: vocabulary violation → exit 0, warning in output
- WARNING: Jaccard hit ≥ 0.4 → exit 0, warning in output
- Clean entry, score ≥ 12 → exit 0, auto-approve-eligible flagged
- Edge: malformed YAML, missing file, empty file

### `test_integrate_entry.py`
- New entry → domain INDEX.md updated, `_summaries/` entry created, labels updated
- New domain → `_index/global.md` updated
- `validate_garden.py --structural` passes after integration
- GitHub issue close invoked with correct issue number (mocked)

### `test_forage_capture.py`
- GitHub mode: `gh issue create` called, branch created, PR created, local validate called
- Local mode: sequential ID assigned from GARDEN.md, scripts called directly, no gh commands invoked
- Mode detection: GitHub remote → GitHub mode; no remote → local mode

---

## Roadmap Items (Not Phase 2)

| Item | Description | Phase |
|------|-------------|-------|
| **Claude in CI** | GitHub Actions calls Claude API for borderline PRs (score 8–11), near-duplicate resolution | Phase 3+ |
| **soredium → PyPI** | Publish as `garden-engine`; CI does `pip install garden-engine`; eliminates two-repo checkout | Phase 3+ |
| **Harvester scheduling** | Cron-triggered harvest session rather than fully manual | Phase 3+ |

---

## Success Criteria

- [ ] A submitter Claude opens a PR; CI posts a validation comment within 60 seconds
- [ ] A CRITICAL failure auto-labels the PR `rejected`; a clean high-score entry auto-labels `auto-approve-eligible`
- [ ] On merge, indexes update automatically and the linked issue closes
- [ ] Local mode produces identical validation output to CI for the same entry file
- [ ] All 118 existing tests plus new Phase 2 tests pass
- [ ] soredium is public; no secrets beyond `GITHUB_TOKEN` required
