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
| `Hortora/soredium` | Claude skills (forage, harvest), CI scripts |
| `Hortora/garden` | Live canonical knowledge garden |

## Architecture Decision Records

`docs/adr/` holds ADRs:
- ADR-0001: Index-and-lazy-reference pattern
- ADR-0002: CI scripts in soredium
- ADR-0003: GE-ID scheme (date + 6 hex)
