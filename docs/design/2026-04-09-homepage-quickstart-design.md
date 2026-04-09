# Homepage Quick Start Section — Design Spec

*Date: 2026-04-09*
*Status: Approved*

---

## Overview

Add a Quick Start section to the homepage (`index.html`) immediately after the hero, and simplify the Getting Started docs page to match the same install approach. The install flow is just one command in Claude Code — no curl, no Python script, no config.

---

## Scope

Two files change:

1. **`index.html`** — new Quick Start section inserted between the hero and the first divider
2. **`style.css`** — new `.qs-*` CSS classes appended
3. **`docs/getting-started.html`** — simplify install section to use `/install-skills`

---

## Quick Start Section

### Placement

Inserted in `index.html` between the `</section>` closing tag of `.hero` and the first `<div class="divider">`.

### Visual design

- `background: var(--parchment-dark)`, `border-top` and `border-bottom: 1px solid var(--sage-light)` — matches the pull-quote section treatment
- `padding: 5rem 2rem`
- Inner container: `max-width: 1100px; margin: 0 auto`
- Section label: `"Get started in three steps"` in `.section-label` style (JetBrains Mono, uppercase, sage, centered)

### Three-step grid

Three equal-width columns (`grid-template-columns: 1fr 1fr 1fr`, `gap: 2.5rem`). Each step:
- Large step number (Cormorant Garamond, 2.5rem, sage-light)
- Optional `.qs-tag` badge (steps 2 and 3 only)
- `h3` heading (Cormorant Garamond, 1.3rem, forest)
- Short paragraph (Lora, 0.85rem, ink-light)
- Dark code block (`.qs-code`) with `.dim` and `.prompt` spans

**Step 01 — Install from the marketplace**
```
# in any Claude Code session
> /install-skills https://github.com/Hortora/soredium
```
No curl, no Python, no config. Claude Code handles discovery and installation automatically.

**Step 02 — forage** (tag: `session-time`)
```
# in any project session
> /forage
```
Queues a submission as a git-tracked file in `submissions/`. Nothing touches the live garden yet.

**Step 03 — harvest** (tag: `maintenance`)
```
# in a maintenance session
> /harvest
```
Integrates pending submissions — assigns GE-IDs, deduplicates, commits to the garden.

### Git graphic

Below the three steps, a two-column terminal grid with a full-width git log panel beneath:

**Left terminal — forage session**
```
> /forage
Captured: Maven skips tests silently
  type: gotcha · tech: maven

$ git status
M  submissions/GE-pending-042.md
$ git log --oneline -1
9e3d8a1 forage: capture GE-042 pending
```

**Right terminal — harvest session**
```
> /harvest
Merged GE-0042 — maven-surefire-skip.md
  1 submission · 0 duplicates

$ git status
A  tools/maven/maven-surefire-skip.md
$ git log --oneline -1
4a2f1bc harvest: merge GE-0042
```

**Full-width git log panel** (two columns side by side):
- Left: `git log --oneline` showing 4 commits (two forage captures, two harvest merges)
- Right: garden state after harvest (two `.md` files listed, `submissions/ — empty`)

Terminal styling: dark background (`var(--ink)`), rounded corners, title bar with 3 sage dots + monospace label, JetBrains Mono body text with `.cmd` (white), `.dim` (sage), `.file` (sage-light), `.prompt` (sage) classes.

---

## CSS additions to style.css

Append a new `/* ── Quick Start section ── */` block with:

| Class | Purpose |
|-------|---------|
| `.quickstart` | Section container — parchment-dark bg, borders, padding |
| `.quickstart-inner` | Max-width 1100px centered wrapper |
| `.qs-steps` | 3-column equal grid, gap 2.5rem, margin-bottom 3rem |
| `.qs-step` | Border-top 2px sage-light, padding-top 1.25rem |
| `.qs-step-num` | Cormorant Garamond 2.5rem, sage-light |
| `.qs-step h3` | Cormorant Garamond 1.3rem, forest |
| `.qs-step p` | Lora 0.85rem, ink-light, margin-bottom 0.75rem |
| `.qs-code` | Dark bg (--ink), #a8d5a2 text, JetBrains Mono 0.75rem, border-radius 4px |
| `.qs-code .dim` | Sage color for comments |
| `.qs-code .prompt` | #c8dcca, user-select: none |
| `.qs-tag` | Inline badge — JetBrains Mono 0.6rem, #E8F4E8 bg, forest text |
| `.qs-graphic` | 2-column grid, gap 1.5rem |
| `.qs-terminal` | Dark bg (--ink), border-radius 6px, overflow hidden |
| `.qs-terminal-bar` | #2d4a2d bg, flex row with dots + label |
| `.qs-dot` | 8px circle, sage, opacity 0.4 |
| `.qs-terminal-label` | JetBrains Mono 0.62rem, sage |
| `.qs-terminal-body` | Padding 1rem 1.25rem, JetBrains Mono 0.72rem, line-height 1.9 |
| `.qs-terminal-body .cmd` | #f4efe4 (white) |
| `.qs-terminal-body .dim` | Sage |
| `.qs-terminal-body .file` | #c8dcca (sage-light) |
| `.qs-terminal-body .prompt` | Sage, user-select: none |
| `.qs-git-tree` | grid-column 1/-1, dark bg, flex row with gap 3rem |
| `.qs-git-tree .label` | JetBrains Mono 0.6rem, sage, uppercase |
| `.qs-git-tree .sha` | #c8dcca |

---

## Getting Started page simplification

Replace the current two-step install section in `docs/getting-started.html`:

**Remove:**
- "Install the skill manager" section (curl command)
- The python3 invocation note paragraph
- "Install forage and harvest" section (two python3 commands)
- The auto-discovery callout

**Replace with a single section:**

**"Install from the marketplace"**

> In any Claude Code session, give it the Hortora marketplace URL. Claude Code discovers and installs the skills automatically — no scripts or config needed.

```
# in any Claude Code session
> /install-skills https://github.com/Hortora/soredium
```

> Callout: Claude Code places skills in `~/.claude/skills/` and makes them immediately available in any session.

The Prerequisites section stays (Claude Code, macOS/Linux). The "Capture your first entry" section stays unchanged.

---

## Out of Scope

- Mobile/responsive layout for the Quick Start section (deferred)
- Animated terminal (static HTML only)
- The soredium README (separate task)
