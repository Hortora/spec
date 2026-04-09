# 0002 — CI script location and soredium visibility

Date: 2026-04-09
Status: Accepted

## Context and Problem Statement

Phase 2 requires Python validation scripts (`validate_pr.py`,
`integrate_entry.py`) that must run identically in GitHub Actions CI and in
local Claude sessions. A home for these scripts must be chosen, and the
visibility of `Hortora/soredium` determines whether CI can access them
without secrets.

## Decision Drivers

* Scripts must run identically in CI and locally — no divergence
* soredium already owns `validate_garden.py` and its test suite
* Keeping tests and the scripts they test in the same repo
* Minimising CI secrets and credential complexity
* Keeping the garden repo as pure data (entries, indexes, workflows)

## Considered Options

* **Option A** — Scripts in `Hortora/garden` (data repo)
* **Option B** — Scripts in `Hortora/soredium`, CI checks out both repos
* **Option C** — soredium published to PyPI; CI installs via `pip install garden-engine`

## Decision Outcome

Chosen option: **Option B**, because soredium is the natural engine home
(tests already live there), and making soredium public eliminates all
secrets complexity — `GITHUB_TOKEN` covers the two-repo CI checkout.

Option C is the intended future state once soredium stabilises, eliminating
the two-repo checkout. This is recorded in the Phase 2 roadmap.

### Positive Consequences

* Single source of truth for all tooling — tests and scripts colocated
* Garden repo stays pure data; no engine code bleeds into it
* `GITHUB_TOKEN` sufficient — no deploy keys or PATs needed
* Local submitters call scripts directly from soredium path; no duplication

### Negative Consequences / Tradeoffs

* CI requires two `actions/checkout` steps (~10–20s overhead)
* Local callers need `SOREDIUM_PATH` configured (defaults to `~/claude/hortora/soredium`)
* Soredium being public exposes all tooling source — intentional but irreversible

## Pros and Cons of the Options

### Option A — Scripts in Hortora/garden

* ✅ Single CI checkout
* ✅ Garden is self-contained
* ❌ Conflicts with soredium's existing test suite — tests and scripts would live in different repos
* ❌ Garden repo mixes data and engine concerns

### Option B — Scripts in soredium, CI checks out both

* ✅ Tests and scripts colocated in soredium
* ✅ Clear separation: soredium = engine, garden = data
* ✅ Public soredium means no secrets beyond GITHUB_TOKEN
* ✅ Natural graduation path to PyPI
* ❌ Two-repo CI checkout
* ❌ Local path coordination needed

### Option C — soredium published to PyPI

* ✅ Single `pip install` in CI and locally
* ✅ Versioned, stable public API
* ❌ Adds publishing overhead and versioning discipline too early
* ❌ Soredium not yet stable enough to publish

## Links

* Phase 2 design spec: `docs/design/2026-04-09-hortora-phase2-github-backend-design.md`
* Blog entry: `docs/blog/2026-04-09-mdp01-designing-the-engine.md`
* Future: Option C (PyPI) tracked in Phase 2 roadmap items
