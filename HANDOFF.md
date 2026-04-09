# Hortora — Project Handoff

This file is the entry point for any Claude working on Hortora. Read this before touching anything.

---

## What Hortora Is

Hortora is a **federated, governed knowledge garden system** — a cross-project, machine-wide library of hard-won technical knowledge (gotchas, techniques, undocumented behaviours). The name comes from *hortus* (Latin: garden) + *-ora*.

The problem it solves: developers rediscover the same non-obvious bugs and workarounds repeatedly across projects. Hortora captures them once, organises them with quality lifecycle management, and eventually federates them across machines and teams via GitHub-backed canonical gardens.

It replaces an ad-hoc system (`~/claude/knowledge-garden/`) that works but has no structure, no validation, no deduplication tooling, and no federation. The full design is in this repo.

---

## Local Folder Structure

`~/claude/hortora/` is the **Hortora GitHub organisation** root. Each subfolder is a separate GitHub repository. This file lives in `spec/` — the other repos are siblings:

```
~/claude/hortora/
├── spec/                       ← Hortora/spec — you are here
│   ├── HANDOFF.md              ← this file
│   └── docs/                   ← design, ADRs, blog, snapshots
├── hortora.github.io/          ← Hortora/hortora.github.io
└── soredium/                   ← Hortora/soredium
```

**Starting a Hortora Claude session:** open Claude in `~/claude/hortora/spec/` and add the sibling directories (`../soredium`, `../hortora.github.io`) as needed for the work at hand.

---

## The Three Repos

### `hortora.github.io` — Public website

**GitHub:** `Hortora/hortora.github.io` · **Live:** https://hortora.github.io

The public-facing landing page and founding blog. Botanical editorial aesthetic. Already live.

```
hortora.github.io/
├── index.html          ← landing page
├── style.css           ← shared stylesheet
└── blog/
    ├── index.html      ← blog index
    ├── 2026-04-07-the-rootstock.html    ← founding blog post 1
    ├── 2026-04-07-the-session.html      ← founding blog post 2
    └── hortora-blog[1-2]-img[1-4].png  ← illustration images (8 files)
```

**State:** Complete and live. No open issues. No immediate work needed.

---

### `spec` — Protocol specification and design docs

**GitHub:** `Hortora/spec`

The open protocol specification for Hortora gardens — how entries are structured, how retrieval works, how federation operates. Also houses ADRs and the founding blog entries in markdown.

```
spec/
└── docs/
    ├── design/
    │   └── 2026-04-07-garden-rag-redesign-design.md   ← THE spec (45k tokens, comprehensive)
    ├── snapshots/
    │   └── 2026-04-07-hortora-knowledge-garden-design.md
    ├── adr/
    │   └── 0001-index-and-lazy-reference-pattern.md
    └── blog/
        ├── 2026-04-07-mdp01-the-rootstock.md
        ├── 2026-04-07-mdp02-the-session.md
        └── hortora-blog[1-2]-img[1-4].png  (8 files)
```

**State:** Content committed locally. **Not yet pushed to GitHub.** No README.

**Open issue:** `Hortora/spec#1` — Push spec to GitHub and write README.

**What the spec covers** (read before implementing anything):
- v2 entry format: YAML frontmatter + markdown body, one file per entry
- Three-tier retrieval algorithm (by technology → by symptom → full scan)
- Three-level deduplication (Jaccard L1/L2/L3)
- Quality lifecycle: Active → Suspected → Superseded → Retired
- GitHub backend: PR-based submission, CI validation, GitHub Issues as entry IDs
- Federation protocol: canonical / child / peer garden relationships
- Nine-phase implementation roadmap (Phase 1 = data migration, Phase 2 = GitHub backend, etc.)
- Planned repos: `garden-spec` (protocol), `garden-engine` (tooling — this is soredium), domain gardens (`jvm-garden`, `tools-garden`, etc.)

---

### `soredium` — Skills, validators, tooling

**GitHub:** `Hortora/soredium`

Named after the lichen's dispersal unit: a self-contained bundle that carries everything needed to establish a new colony wherever it lands. Soredium is the **tooling and Claude skill repo** for Hortora — validators, CI, MCP server, Obsidian plugin, and Claude skills.

```
soredium/
└── README.md           ← brief description, links to #1
```

**State:** Scaffold placeholder only. No skills, no marketplace, no tooling yet.

**Open issues:**
- `#1` — Scaffold soredium (CLAUDE.md, sync script, basic structure)
- `#2` (epic) — Add forage and harvest garden skills to soredium
  - `#3` — Set up soredium as a standalone Claude skill marketplace
  - `#4` — Add forage skill (CAPTURE, SWEEP, SEARCH, REVISE)
  - `#5` — Add harvest skill (MERGE, DEDUPE)

---

## The Live Garden (Current State — Not Hortora Yet)

The garden is **currently running** at `~/claude/knowledge-garden/` using the `garden` skill from cc-praxis (`~/.claude/skills/garden/`). This is the v1 system. It works and multiple project Claudes depend on it.

```
~/claude/knowledge-garden/
├── CHECKED.md          ← dedupe pair log
├── DISCARDED.md        ← discarded duplicates
├── README.md
├── submissions/        ← 10 unmerged submissions pending
├── tools/              ← git, tmux, playwright, llm-testing, etc. (19 files)
├── quarkus/            ← cdi, config, maven, testing, webauthn, etc. (9 files)
├── java/               ← generics, concurrency, records, maven (6 files)
├── intellij-platform/  ← indexing, rename-refactoring (2 files)
├── java-panama-ffm/    ← native-image-patterns, pty-patterns (2 files)
├── drools/             ← quarkus-testing, rule-builder-dsl (2 files)
├── macos-native-appkit/← appkit-panama-ffm (1 file)
├── beautifulsoup/      ← encoding (1 file)
├── apache-jexl/        ← jexl3 (1 file)
├── approaches/         ← java-dsl-design (1 file)
├── claude-code/        ← settings-json-quirks (1 file)
├── graalvm-native-image/
├── permuplate/         ← (2 files)
└── scelight/           ← (1 file)
```

**104 entries total. Last assigned ID: GE-0114. Drift counter reset 2026-04-09.**

**10 pending submissions** in `submissions/` — not yet merged. Run harvest (or garden MERGE) before doing anything structural.

**GARDEN.md deleted** — it was always empty (never had an index built). Hortora's approach supersedes it.

---

## The Skill Migration Plan

### Why splitting garden into forage + harvest

The `garden` skill conflates two very different activities:

| `forage` | `harvest` |
|----------|-----------|
| Session-time, lightweight | Dedicated maintenance session, heavy |
| CAPTURE, SWEEP, SEARCH, REVISE | MERGE, DEDUPE |
| Happens during work | Never happens mid-work |
| Low context cost | Reads many files |

### The constraint: don't break other Claudes

Multiple project Claudes (starcraft, remotecc, permuplate, quarkusai, etc.) use `garden` via `~/.claude/skills/garden/`. **The installed `garden` skill must not change until the migration is complete.** Other Claudes write submissions to `~/claude/knowledge-garden/submissions/` — `harvest` must process those too.

### Migration sequence

1. **Now (soredium #3):** Set up soredium as a skill marketplace
2. **soredium #4:** Write `forage` skill — extracts CAPTURE/SWEEP/SEARCH/REVISE from garden
3. **soredium #5:** Write `harvest` skill — extracts MERGE/DEDUPE from garden
4. **Later:** When ready to cut over, sync forage+harvest to `~/.claude/skills/` and tell other Claudes
5. **Eventually:** Data migration from `~/claude/knowledge-garden/` to Hortora v2 structure (Phase 1 of spec)

### Compatibility requirements

- `forage` submissions must use the **same file format** as `garden` submissions (so `harvest` can process both)
- `harvest` must process submissions from both `forage` and the legacy `garden` skill
- The GE-ID counter (`GE-0114` was last assigned) needs a home — currently it was in GARDEN.md (now deleted). Decide during #5 implementation — likely a small `metadata.md` or similar.

---

## Key Decisions Made

| Decision | Rationale |
|----------|-----------|
| Skill name stays `garden` for installed skill | Backward compat — other Claudes know it |
| New skills named `forage` (verb) and `harvest` (verb) | Garden is a noun; skills should be verbs |
| forage + harvest live in soredium, not cc-praxis | Hortora is fully self-contained; no dependency on cc-praxis |
| cc-praxis `garden` skill untouched throughout migration | Other project Claudes must not be disrupted |
| GARDEN.md deleted | Was always empty; Hortora's indexed structure supersedes it |
| Soredium has its own marketplace | Users can install from Hortora without knowing cc-praxis |
| cc-praxis may aggregate soredium's marketplace | Idea logged in cc-praxis `docs/ideas/IDEAS.md` — not decided yet |

---

## What To Do Next (in order)

### Immediate — unblocked now

**1. Push spec to GitHub** (`Hortora/spec#1`)
```bash
cd ~/claude/hortora/spec
git push
# Then write README.md
```

**2. Merge pending garden submissions**
```bash
# 10 submissions pending in ~/claude/knowledge-garden/submissions/
# Use garden MERGE (or harvest once written)
```

**3. Scaffold soredium** (`Hortora/soredium#1` + `#3`)
- Write `CLAUDE.md` (project type: skills, sync workflow)
- Copy `scripts/claude-skill` from cc-praxis
- Create `.claude-plugin/marketplace.json`
- Mirror cc-praxis skill structure

**4. Write forage skill** (`Hortora/soredium#4`)
- Extract CAPTURE, SWEEP, SEARCH, REVISE from `~/claude/cc-praxis/skills/garden/SKILL.md`
- Keep submission format identical
- Add `/forage` slash command

**5. Write harvest skill** (`Hortora/soredium#5`)
- Extract MERGE, DEDUPE from `~/claude/cc-praxis/skills/garden/SKILL.md`
- Decide on GE-ID counter location (GARDEN.md is gone)
- Add `/harvest` slash command

### Later — blocked on above

**6. Data migration** (Phase 1 of spec)
- Migrate 104 entries from `~/claude/knowledge-garden/` to v2 structure
- Requires spec Phase 1 implementation in soredium first

**7. GitHub backend** (Phase 2 of spec)
- PR-based submission workflow
- CI validation (`validate_pr.py`, `integrate_entry.py`)

---

## Reference Links

| Resource | Location |
|----------|----------|
| Full design spec | `~/claude/hortora/spec/docs/design/2026-04-07-garden-rag-redesign-design.md` |
| Current garden skill | `~/claude/cc-praxis/skills/garden/SKILL.md` |
| Installed skills dir | `~/.claude/skills/` |
| Live garden data | `~/claude/knowledge-garden/` |
| Hortora Roadmap | https://github.com/orgs/Hortora/projects (GitHub Project) |
| soredium issues | https://github.com/Hortora/soredium/issues |
| spec issues | https://github.com/Hortora/spec/issues |
| Live site | https://hortora.github.io |

---

## Background Reading (in order of importance)

1. **This file** — you're reading it
2. **The spec** — `spec/docs/design/2026-04-07-garden-rag-redesign-design.md` — the complete vision
3. **Founding blog posts** — `spec/docs/blog/2026-04-07-mdp01-the-rootstock.md` and `mdp02-the-session.md` — the why
4. **Current garden skill** — `~/claude/cc-praxis/skills/garden/SKILL.md` — what forage/harvest must cover

---

*Last updated: 2026-04-09 — cc-praxis session covering Hortora decommission in cc-praxis, garden DEDUPE, soredium issue creation, and migration planning.*
