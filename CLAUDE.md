# CLAUDE.md

## Project Type

**Type:** custom

## Repository Purpose

**Hortora spec** — the open protocol specification for the Hortora knowledge garden system. Covers the federation model, entry format, governance, CI pipeline, and design decisions.

See `docs/adr/` for architectural decision records and `docs/blog/` is no longer the blog location (see below).

## Blog

**Blog directory:** `../hortora.github.io/_posts/`

Blog posts for the Hortora project live in the `hortora.github.io` repository, not in this spec repo. When writing a blog entry with `write-blog`, it will write directly to `../hortora.github.io/_posts/` which is a sibling directory on disk.

Blog posts require Jekyll frontmatter:
```yaml
---
layout: post
title: "Entry Title"
date: YYYY-MM-DD
entry: "NN"        # sequential entry number (check existing posts for next number)
type: origin | session | phase-update | pivot | correction
excerpt: "One sentence excerpt shown in the diary listing."
---
```

The site publishes automatically to `hortora.github.io` — GitHub Pages builds Jekyll on every push to `hortora.github.io` main branch.

## Key Repositories

| Repo | Purpose |
|---|---|
| `Hortora/spec` (this repo) | Protocol specification, ADRs, design docs |
| `Hortora/hortora.github.io` | Public website + blog (Jekyll) |
| `Hortora/soredium` | Claude skills (forage, harvest), CI scripts, `garden-agent-install.sh` |
| `Hortora/garden` | Live canonical knowledge garden |

**New machine garden setup:** after cloning the garden, run `~/claude/hortora/soredium/scripts/garden-agent-install.sh` from inside `~/.hortora/garden` to install the autonomous dedup agent (post-commit hook, CLAUDE.md, settings). One-time per machine.

## Architecture Decision Records

`docs/adr/` holds ADRs:
- ADR-0001: Index-and-lazy-reference pattern
- ADR-0002: CI scripts in soredium
- ADR-0003: GE-ID scheme (date + 6 hex)

## Work Tracking

**Issue tracking:** enabled
**GitHub repos:** Hortora/soredium (implementation), Hortora/spec (design/docs), Hortora/hortora.github.io (website)
**Primary repo for implementation issues:** Hortora/soredium
**Changelog:** GitHub Releases (run `gh release create --generate-notes` at milestones)

**Automatic behaviours (Claude follows these at all times in this project):**
- **Before implementation begins** — when the user says "implement", "start coding",
  "execute the plan", "let's build", or similar: check if an active issue or epic
  exists. If not, run issue-workflow Phase 1 to create one **before writing any code**.
- **Before writing any code** — check if an issue exists for what's about to be
  implemented. If not, draft one and assess epic placement (issue-workflow Phase 2)
  before starting. Also check if the work spans multiple concerns.
- **Before any commit** — run issue-workflow Phase 3 (via git-commit) to confirm
  issue linkage and check for split candidates. This is a fallback — the issue
  should already exist from before implementation began.
- **All commits should reference an issue** — `Refs #N` (ongoing) or `Closes #N` (done).
  If the user explicitly says to skip ("commit as is", "no issue"), ask once to confirm
  before proceeding — it must be a deliberate choice, not a default.
- **Route issues by concern:** implementation → Hortora/soredium, design/spec → Hortora/spec, website → Hortora/hortora.github.io
