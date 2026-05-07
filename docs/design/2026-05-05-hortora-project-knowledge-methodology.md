# Hortora as a Project Knowledge Methodology

**Date:** 2026-05-05  
**Status:** Active design — decisions accumulating, ready for spec when section marked SETTLED  
**Context:** Emerged from examining the overlap between casehub platform protocols and
hortora knowledge gardens.

---

## Immediate Priorities (Before Audit, Before Spec)

These can be done now without waiting for the audit or full spec:

**1. Migrate existing casehub conventions to new format**
Revise all 33 files in `parent/docs/conventions/` to use YAML frontmatter with the
settled schema (id, title, type, scope, applies_to, severity, refs, created). Rename
folder to `protocols/`. Content stays as-is — this is metadata, not a content review.
The content review happens in Audit 2.

**2. Create the `protocol` skill**
New skill in hortora/soredium. Initial operations: CAPTURE, SWEEP, SEARCH, HEALTH, DEEP-SCAN.
DEEP-SCAN can be stubbed for now — the others are needed for the handover integration.

**3. Hook protocol SWEEP into the handover wrap**
Add to the handover checklist alongside forage sweep. Both run while context is full.

---

## Validation Required Before Spec

All structural decisions below are hypotheses until validated against actual content.
**Do not write a spec or implementation plan until this audit is complete.**

### Audit 1 — Canonical garden casehub entries
Scan all entries in `~/.hortora/garden/` that reference casehub concepts
(`WorkItem`, `startCase`, `CaseInstance`, `Channel`, `Commitment`, `Qhorus`, `inputSchema`, `callerRef`).
For each: does it belong in the casehub garden, or is the casehub context incidental and the entry is genuinely universal?

Known count: ~70 quarkus/ entries reference casehub concepts. Expected outcome: most move to casehub garden, some stay canonical with casehub context stripped or noted as incidental.

### Audit 2 — Conventions split
For each of the 33 casehub conventions, classify:
- **Universal Quarkus/Java gotcha** → move to canonical garden as a discovery entry
- **Casehub-specific rule** → becomes a protocol in the casehub garden
- **Duplicate of an existing garden entry** → resolve the contradiction, keep one

Known contradiction: `quarkus-junit-not-junit5.md` convention says use `quarkus-junit`; garden entry GE-20260415-3ce5f3 says use `quarkus-junit5`. One is wrong. Resolve before migration.

### Audit 3 — Folder structure validity
Proposed casehub garden folders:

**Foundation repos** (platform layer — building the platform):
`platform/`, `engine/`, `qhorus/`, `work/`, `ledger/`, `connectors/`, `claudony/`, `repos/`

**Application repos** (application layer — built ON the platform):
`devtown/`, `aml/`, `clinical/`

After audits 1 and 2, check: does every entry have a natural home in this structure, or do edge cases suggest additional folders or a different grouping?

### Audit 4 — repos/ depth check
`parent/docs/repos/` now has 9 files: 6 foundation + 3 application (devtown, aml, clinical — new).
For each: does it stay at family-awareness level (module ownership, what it does NOT do,
dependency graph) or has it crept into class-level design detail?
Trim anything that belongs in the module's own `DESIGN.md`, add a reference in its place.

---

## PLATFORM.md Structure — Foundation vs Application Split

**Problem:** PLATFORM.md is consulted by foundation repo sessions during spec and implementation.
Loading application-tier detail (devtown, aml, clinical) into that context is noise — foundation
repos almost never need to know about applications unless specifically asked (e.g. "how does
devtown use this feature?").

**Rule:** PLATFORM.md covers the foundation layer only. Application tier gets its own document.

```
parent/docs/PLATFORM.md      ← foundation repos only; one line linking to APPLICATIONS.md
parent/docs/APPLICATIONS.md  ← application tier (devtown, aml, clinical); deep-dive links
```

**PLATFORM.md** contains: what the platform is, foundation repo map, capability ownership,
build order, boundary rules, deep-dive links for foundation repos. Ends with:
> "Application tier (devtown, aml, clinical): see [APPLICATIONS.md](APPLICATIONS.md)"

**APPLICATIONS.md** contains: what each application does, how it uses the platform,
dependency on which foundation repos, deep-dive links for application repos.

**Loading rules by context:**
- Foundation repo session (engine, qhorus, work…) → loads PLATFORM.md only
- Application repo session (devtown, aml, clinical) → loads both PLATFORM.md + APPLICATIONS.md
- "How does devtown use X?" query → explicitly fetch APPLICATIONS.md + devtown deep-dive

**Action required in casehub/parent:** Split the restored PLATFORM.md — move application
tier content out to new `docs/APPLICATIONS.md`, leave a single link in PLATFORM.md.

---

## The Core Idea

Hortora is not just a knowledge garden tool. It could be **"the way" for how a project
structures all its knowledge artifacts** — from the moment a new project starts up with
no established best practices.

A new project that adopts the Hortora methodology gets:
- A structured approach to creating and maintaining ADRs
- A structured approach to creating and maintaining platform protocols
- A structured approach to discovery/garden entries
- All three in formats that are tuned for optimal RAG from day one
- A consistent hierarchy model across all three artifact types
- Tooling that guides the creation and maintenance of each type

The analogy: if CLAUDE.md tells an LLM what the project does, Hortora tells the LLM
what the project **knows** — about itself, its technology choices, and its conventions.

---

## The Three Knowledge Layers (Current Thinking)

### Layer 1: Knowledge Gardens (Discovery)
What: Gotchas, techniques, undocumented behaviours discovered during development.  
Audience: Any LLM working with this technology stack.  
Scope: Universal → Community → Project (hierarchical, each level augmenting the parent).  
Current state: Implemented in hortora. Canonical garden live at `~/.hortora/garden/`.

### Layer 2: Platform Protocols (Conventions)
What: Normative rules for how this platform is built — prescriptive, not descriptive.  
Audience: LLMs developing THIS platform specifically.  
Scope: Project-scoped. Possibly hierarchical too (parent platform → child product).  
Current state: Implemented ad-hoc in casehub as `parent/docs/conventions/` + `repos/`.  
Format: Problem + Rule + Example (good, but isolated from garden system).

### Layer 3: Architectural Decisions (ADRs)
What: Settled decisions with rationale — why this approach over the alternatives.  
Audience: LLMs making design decisions on THIS platform.  
Scope: Project-scoped. Hierarchical — child project can inherit or override parent ADRs.  
Current state: Implemented in individual repos (casehub-engine `adr/`, etc.). No cross-project view.

---

## The Hierarchy Principle (Applies to All Three)

Just as knowledge gardens are hierarchical (canonical → community → private), all three
knowledge types could be hierarchical:

```
canonical knowledge   (universal technology facts — anyone's Quarkus garden)
    └── community     (framework/ecosystem conventions — "the Quarkus way")
        └── platform  (platform-level conventions — "the casehub way")
            └── product (product-level overrides — "claudony specifically does X")
```

A child node **augments** the parent by default. Where it **overrides**, the override wins.

Example:
- Canonical: "Quarkus @IfBuildProperty is evaluated at build time only" (discovery garden)
- Platform: "casehub-engine uses RAM-only Quartz — no JDBC store" (platform protocol)
- Product: "claudony's scheduler uses JDBC store for restart durability" (override of parent protocol)

An LLM working on claudony would see: inherit all casehub conventions except where
claudony explicitly overrides.

---

## Why This Matters for RAG

If all three layers use consistent:
- Metadata schema (domain, scope, type, applies_to)
- Physical folder structure (aligned with the hierarchy)
- Entry formats (optimised for embedding and retrieval)

...then a single RAG query can surface the right knowledge regardless of which layer
it lives in, ranked by specificity (product > platform > community > canonical).

The LLM doesn't need to know where to look. It asks one question, gets back a ranked
set of relevant facts, rules, and decisions — each tagged with its layer and scope.

---

## What Hortora Tooling Would Provide for a New Project

A new project adopting the Hortora methodology would get:
1. **Scaffold** — folder structure, INDEX.md templates, metadata schema for all three layers
2. **Capture tools** — forage-style skill for each layer (garden entry, protocol, ADR)
3. **Validation** — schema checks, staleness tracking, hierarchy consistency
4. **Format guidance** — entry formats tuned for RAG at each layer
5. **Integration** — forage SEARCH that queries across all three layers simultaneously

Hortora becomes the methodology that ensures knowledge is captured in the right place,
in the right format, with the right metadata — whether the project is day one or year two.

---

## Settled Decisions

### Physical Location
Project-specific knowledge belongs in the project, not the universal garden.

**Multi-repo project** (has a parent/root coordination repo):
```
parent/docs/protocols/         ← platform-wide rules (rename from conventions/)
parent/docs/repos/             ← family awareness docs (trimmed — no class-level detail)
<child-repo>/docs/protocols/   ← repo-specific rules (new, currently absent)
```

**Single-repo project:**
```
docs/protocols/                ← all protocols for this project
```

### Protocol Types
Two flavours, both live in `protocols/`:

| Type | Example | Content |
|---|---|---|
| `principle` | "Prefer normative layers for interactions" | Directive + references only |
| `rule` | "Never render EVENT content directly" | Directive + minimum example needed to follow it |

Protocols do NOT contain design detail. They reference out to the project docs that do.

### Protocol Entry Format
```yaml
---
id: PP-YYYYMMDD-xxxxxx
title: "Short directive title"
type: principle | rule
scope: platform | repo
applies_to: "which modules / contexts"
severity: critical | important | guidance
refs:
  - path/to/design/detail.md
violation_hint: "optional — plain-language description of what a violation looks like"
created: YYYY-MM-DD
---

One paragraph. The directive. Minimum context to understand why.
No implementation detail — refs: covers that.
```

### Link Integrity — PROTOCOL-REFS.yaml
When a protocol references a file in another project, that project gets a reverse-index
entry. This prevents silent rot when files are renamed or moved.

**`<project>/docs/PROTOCOL-REFS.yaml`:**
```yaml
refs:
  - file: docs/normative-layer.md
    protocols:
      - id: PP-20260505-xxxxxx
        title: "Prefer normative layers for inter-module interactions"
        location: parent/docs/protocols/PP-20260505-xxxxxx.md
```

Maintained automatically by the `protocol` skill. Checked by git-commit hooks in each repo.

### The `protocol` Skill
A new skill (lives in hortora/soredium, parallel to `forage`). Operations:

| Operation | When | What |
|---|---|---|
| `CAPTURE` | Session-time | Write protocol file, update PROTOCOL-REFS.yaml in all referenced projects |
| `SWEEP` | Handover wrap | Scan session for protocol-worthy patterns; surface broken refs from renamed files |
| `SEARCH` | On demand | Find protocols relevant to current work |
| `HEALTH` | On demand | Full integrity scan — verify all refs exist, rebuild PROTOCOL-REFS.yaml if inconsistent |
| `DEEP-SCAN` | One-off, on demand | Analyse one or more repo codebases for patterns not yet recorded as protocols |

`HEALTH` is heavyweight (like `harvest`), run on demand after large refactors or when onboarding a new repo. Not every session.

`DEEP-SCAN` is a dedicated session operation — heavier than SWEEP. Takes one or more repo paths and runs three passes:

1. **Gap finding** — reads the codebase for repeated patterns not captured in any existing protocol. Presents candidates: *"Every CDI async observer delegates to a separate @Transactional bean across 6 modules — not recorded. Add as a protocol?"* Approved candidates run through CAPTURE.

2. **Relevance check** — for each existing protocol, verifies it still applies. If the technology it covers is absent from the codebase → flag for retirement. If no code implements it yet → mark `anticipatory` (future code will need it — keep, don't retire). If actively used → confirmed current.

3. **Violation detection** — scans for code that breaks a recorded protocol. Uses optional `violation_hint` field in the protocol's YAML frontmatter to guide the scan; infers from content if not present. Surfaces: *"2 potential violations of PP-xxxxx — `MessageService.java:147`, `WorkItemService.java:89` — review?"*

`violation_hint` is an optional frontmatter field:
```yaml
violation_hint: "fireAsync() call inside a method annotated @Transactional"
```
Protocols without it are still scanned — less precisely.

### Handover Wrap Integration
Protocol SWEEP runs alongside forage SWEEP in the handover checklist, while context is full:

```
[x] forage sweep     — universal garden (gotchas, techniques, undocumented)
[x] protocol sweep   — project protocols (patterns + ref integrity check)
```

The distinction in sweep: forage asks "universal enough for the garden?", protocol asks
"repeated platform-specific pattern worth codifying?"

### repos/ Boundary
`repos/` files stay high-level: what the module owns, what it explicitly does NOT do,
who depends on it. No class names, no method signatures — those belong in DESIGN.md.
Where current `repos/` files have crept into design detail, trim and add a reference.

---

## Format Gap: Casehub Conventions vs Garden Entries

The casehub conventions are structurally good for human reading but are **not
RAG-optimal** — they lack the machine-readable metadata that makes garden entries
indexable, queryable, and rankable.

**Current casehub convention format:**
```markdown
# Convention: Qhorus EVENT Message Content Is Always Null
**Applies to:** All projects that read or display Qhorus MessageLedgerEntry records
**Severity:** Important — content renders as blank with no error

## The Problem
...
## The Rule
...
## Example
...
```

**Current garden entry format:**
```yaml
---
id: GE-20260505-f60bab
title: "MCP StdioServerParameters command='python3' spawns wrong interpreter"
type: gotcha
domain: tools
stack: "python, mcp"
tags: []
score: 11
verified: true
staleness_threshold: 730
submitted: 2026-05-05
---
```

**What casehub conventions are missing:**
- YAML frontmatter — no machine-readable metadata at all
- Unique IDs — no addressing scheme (would need e.g. `PP-YYYYMMDD-xxxxxx`)
- `type` field — is this a gotcha-derived rule, an architectural constraint, a safety rail?
- `scope` / `applies_to` — present as freetext, not structured metadata
- `severity` — freetext, not a consistent enumeration
- `layer` — no indication of where this sits in the hierarchy (platform vs. product)
- `overrides` — no mechanism to indicate a child entry supersedes a parent
- Staleness tracking — no `submitted`, `staleness_threshold`, `verified` fields

**The consistency requirement:**
Even if the casehub conventions stay in their current location and format for now,
the Hortora methodology should define a canonical format for platform protocols that:
- Uses YAML frontmatter with a consistent schema
- Uses a consistent ID scheme across all three knowledge layers
- Aligns folder structure with the hierarchy model
- Makes entries indexable by the same tooling that indexes garden entries

Casehub's existing 35+ conventions would need migration to this format — but the
content is already there. The work is metadata, not substance.

---

## Open Questions for Brainstorming Session

1. **Are platform protocols a garden type, or a separate artifact class?**  
   The 6-garden taxonomy has `decisions-garden` — does that absorb protocols? Or are
   protocols normative (prescriptive) while decisions are descriptive (why we chose X)?

2. **How does hierarchy work for protocols specifically?**  
   If casehub defines a convention and claudony overrides it, how is the override
   represented? A separate file? A `overrides:` field? A different entry in the same index?

3. **Who owns the canonical protocol layer?**  
   For garden entries, the community owns the canonical layer. For platform protocols,
   there is no "community" — it's project-specific from the start. Does the hierarchy
   collapse to just two levels (platform → product)?

4. **What does the RAG consumption layer look like?**  
   When an LLM queries "what do I know about CDI async events in casehub?", should it
   get: garden entries about CDI gotchas (canonical) + casehub conventions about async
   (platform) + relevant ADRs (decisions) all in one ranked result set?

5. **What's the capture workflow for protocols?**  
   Garden entries are captured via `forage` at session end. Protocols are currently
   written by hand. Should there be a `protocol` capture flow analogous to forage?
   When does a convention get written — immediately when discovered, or after a
   pattern repeats?

6. **How does this affect the current casehub conventions directory?**  
   The 35+ files in `parent/docs/conventions/` are already a good platform protocol
   layer. Do they need reformatting, re-tagging, or just an INDEX that makes them
   RAG-queryable? Or should they migrate into a garden-style structure?

---

## Things This Is NOT (Scope Constraints for Future Brainstorm)

- Not a replacement for CLAUDE.md (which is LLM instructions, not project knowledge)
- Not a code documentation system (jsdoc, javadoc — different purpose)
- Not an onboarding guide for humans (knowledge here is optimised for LLM consumption)
- Not a general wiki (everything here must pass the RAG-value test: does retrieval of
  this entry improve an LLM's output on a real development task?)
