# Hortora Website Documentation — Design Spec

*Date: 2026-04-09*
*Status: Approved*

---

## Overview

Add a documentation section to `hortora.github.io` covering Getting Started, How It Works, Skills Reference, and Design Spec. The marketplace state (soredium pushed, `claude-skill` installable) is confirmed — this work makes that publicly documented.

---

## Structure

### File layout

```
hortora.github.io/
├── index.html                  ← unchanged (landing page)
├── style.css                   ← extended with docs layout classes
├── .gitignore                  ← .superpowers/ excluded
├── blog/                       ← unchanged
└── docs/
    ├── getting-started.html
    ├── how-it-works.html
    ├── skills-reference.html
    └── design-spec.html
```

No `docs/index.html` — the nav links directly to `getting-started.html`.

### Navigation

All pages (index, blog, docs) get a "Docs" link added to the nav bar pointing to `docs/getting-started.html`.

```
Hortora    Home · Docs · Blog
```

### Sidebar (all docs pages)

Left sidebar, sticky, 220px wide. Active page highlighted in forest green. Same HTML block inline in each docs page.

```
Documentation
  ● Getting Started
    How It Works
    Skills Reference
    Design Spec
```

### Style additions to style.css

New classes only — no separate CSS file:
- `.docs-layout` — flex container
- `.docs-sidebar` — sticky, 220px, parchment-dark background, right border
- `.docs-sidebar a` — Lora, ink-light, hover + active states
- `.docs-content` — flex:1, max-width 760px, 3rem/4rem padding
- `.callout` — left-bordered info block
- `.code-block` — dark background, JetBrains Mono, sage-green text
- `.docs-footer` — prev/next page navigation

---

## Page Content

### Getting Started

1. **Prerequisites** — Claude Code (linked), Python 3.8+, macOS/Linux
2. **Install the skill manager** — curl one-liner to fetch `claude-skill` from soredium
3. **Install skills** — `python3 claude-skill install forage` then `harvest`
4. **Tip callout** — Claude Code auto-discovers skills in `~/.claude/skills/`, no registration needed
5. **Capture your first entry** — brief walkthrough: Claude offers proactively, or trigger via `/forage CAPTURE`
6. **Footer nav** — "How It Works →"

### How It Works

1. **The garden model** — machine-wide library at `~/claude/knowledge-garden/`, three entry types: gotchas, techniques, undocumented
2. **Entry lifecycle** — capture (session-time via forage) → submissions queue → merge (harvest MERGE) → dedupe (harvest DEDUPE)
3. **GE-IDs** — stable identifiers (e.g. `GE-0042`) assigned at capture, referenced across sessions
4. **Local-first** — no server, no account, plain files any Claude session can read and contribute to
5. **Footer nav** — "← Getting Started" / "Skills Reference →"

### Skills Reference

**forage — session-time operations**

| Operation | Trigger | Description |
|-----------|---------|-------------|
| CAPTURE | Non-obvious knowledge surfaces | Record a single gotcha, technique, or undocumented behaviour |
| SWEEP | End of session | Systematically scan the session for anything worth capturing |
| SEARCH | Looking up prior knowledge | Retrieve matching entries from the garden |
| REVISE | Enriching an existing entry | Update or expand a previously captured entry |

**harvest — maintenance operations**

| Operation | Trigger | Description |
|-----------|---------|-------------|
| MERGE | Submissions pending | Integrate queued submissions into the garden, assign GE-IDs |
| DEDUPE | After bulk additions | Find and resolve duplicate or overlapping entries |

Footer: link to full `SKILL.md` files on GitHub for complete reference.

**Footer nav** — "← How It Works" / "Design Spec →"

### Design Spec

1. **Architecture diagram** — CSS-only loop diagram (flex boxes + arrows, no SVG or JS):
   `Claude session → forage → submissions/ → harvest → garden/ → Claude session`
2. **Entry format** — YAML frontmatter (id, title, type, technology, tags) + markdown body
3. **Three-tier retrieval** — by technology → by symptom → full scan
4. **Quality lifecycle** — Active → Suspected → Superseded → Retired
5. **GitHub backend (planned)** — PR-based submission, CI validation, GitHub Issues as entry IDs
6. **Federation (planned)** — canonical / child / peer garden relationships
7. **"Read the full spec"** — link to `spec/docs/design/2026-04-07-garden-rag-redesign-design.md` on GitHub

**Footer nav** — "← Skills Reference"

---

## Visual Design

Matches the existing site aesthetic throughout:

- **Background:** `--parchment` (#F4EFE4), sidebar `--parchment-dark` (#EDE5D4)
- **Headings:** Cormorant Garamond, forest green (`--forest` #3A6040)
- **Body:** Lora, ink-light (#2D4A2D)
- **Code:** JetBrains Mono, dark background (#1C2B1C), sage-green text
- **Active sidebar link:** forest green background, parchment text
- **Callout blocks:** left border in sage, parchment-dark background

---

## Out of Scope

- Mobile responsiveness (sidebar collapse) — deferred
- Search across docs pages — deferred
- Versioning — not needed yet
- soredium README update — separate task, not part of this implementation
