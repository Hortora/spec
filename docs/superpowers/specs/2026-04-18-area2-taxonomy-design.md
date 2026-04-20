# Area 2 — Knowledge Taxonomy Design

**Date:** 2026-04-18
**Status:** Approved — ready for implementation planning
**Supersedes:** Knowledge model in `2026-04-16-garden-ecosystem-plan.md` (5 gardens → 6)
**Context:** Part of the 7-area garden ecosystem build sequence. Area 2 must be locked
before Area 5 (consumption/routing) can be designed.

---

## Overview

The garden ecosystem organises knowledge by **type**, not by technology domain. Technology
is metadata — a filter applied within a garden, not a structural organiser. Six canonical
garden types, each valid at every level of the spanning-tree federation hierarchy.

---

## The 6 Gardens

| Garden | Knowledge type | Primary capture |
|--------|---------------|-----------------|
| `discovery-garden` | Gotchas, techniques, undocumented behaviours | Session (forage) + ticket mining |
| `patterns-garden` | Observed architectural patterns — provenance, suitability, variants | Ecosystem mining + delta analysis |
| `examples-garden` | Minimal working code, intent-driven, copy-paste ready | Code miner |
| `evolution-garden` | Breaking changes, deprecations, version-specific facts | Release note mining + session |
| `risk-garden` | Failure modes, production anti-patterns, post-mortems | Ticket mining + post-mortem mining |
| `decisions-garden` | ADRs, rejected approaches, settled choices | ADR mining + session |

### Editorial Bars

Each garden has a single-sentence filter applied at submission time:

| Garden | Editorial bar |
|--------|---------------|
| `discovery-garden` | Would a skilled developer familiar with the technology still have spent significant time on this? |
| `patterns-garden` | Would a practitioner facing this class of problem reach for something more complex or less elegant without this pattern? |
| `examples-garden` | Is this minimal, working, and demonstrating a real use case — not a toy? |
| `evolution-garden` | Does this describe a breaking change, deprecation, or capability shift that would change code correctness for someone on that version? |
| `risk-garden` | Has this failure mode caused production harm at meaningful scale, and is the mechanism universal enough to recur in other systems? |
| `decisions-garden` | Does this capture the reasoning behind a choice clearly enough that someone facing the same choice could apply it — including what was rejected and why? |

`examples-garden` has a deliberately low bar — the code miner populates it at volume,
quality comes from completeness (compiles, demonstrates intent) not editorial judgment.
All others require human review.

---

## Federation — Spanning-Tree at Every Level

All 6 gardens are valid at every node of the hierarchy:

```
canonical <garden>          ← universal, community-curated
    └── community <garden>  ← project/framework community (e.g. Quarkus)
        └── private <garden>← org-specific, child of community
```

`decisions-garden` demonstrates this most clearly:

```
canonical decisions-garden          ← universal architectural reasoning
    └── quarkus-decisions-garden    ← Quarkus community choices + rationale
        └── acme-decisions-garden   ← ACME Corp's specific ADRs
```

Auto-capture sharpens at each level:
- **Canonical:** mine ADRs broadly from open-source projects across many domains
- **Community:** mine that project's own repo (Quarkus, Spring, Kubernetes, etc.)
- **Private:** mine the org's own ADR directory — fully automated, zero friction

Do not flatten `decisions-garden` to "private only". Community-level decisions gardens
(e.g. "why Quarkus chose Vert.x") are legitimate, valuable, and fully auto-capturable
from the project's own repository.

---

## Patterns-Garden — Extended Model

The patterns-garden captures more than named design patterns. It captures **observed
architectural patterns from real production codebases** — patterns discovered by studying
how projects actually solve structural problems, with enough context to decide whether
and how to apply them.

### Extended Entry Format

Beyond the standard garden entry fields:

```yaml
observed_in:
  - project: serverless-workflow
    url: https://github.com/serverlessworkflow/specification
    path: impl/core/src/main/java/io/serverlessworkflow/impl/
    first_seen: "2022-03-14"
    last_seen: "2026-04-01"
  - project: quarkus-drools
    url: https://github.com/kiegroup/kogito-runtimes
    path: ...
    first_seen: "2023-01-08"

suitability: |
  Works when evaluation logic must be swappable at runtime without recompilation.
  Less suitable when state is simple and unlikely to change.

variants:
  - name: in-memory
    description: Evaluator holds all state in-process
    tradeoffs: Fast, no serialisation overhead; not distributable
  - name: event-sourced
    description: Evaluator state reconstructed from event log
    tradeoffs: Distributable, auditable; higher latency, complex recovery

variant_frequency:
  in-memory: 12
  event-sourced: 31
  distributed: 4

authors:
  - github_handle: fabian-martinez
    role: originator        # introduced the pattern
  - github_handle: sanne-grinovero
    role: innovator         # introduced the event-sourced variant

stability: high             # computed: consistent across 3+ major versions of 5+ projects
```

### Relationship to examples-garden

`patterns-garden` is prose-heavy architectural description. `examples-garden` is code.
They are distinct — different embedding models, different editorial bars, different
capture pipelines, different retrieval intent. Do not merge them.

---

## Pattern Discovery Pipeline

Two mechanisms. Session-capture as a discovery seed was considered and rejected —
it depends on a human already knowing what they found, making it not true discovery.

### Mechanism 1: Structural Clustering (Primary)

1. Extract structural features from all indexed projects: interface counts, abstraction
   layers, injection points, dependency graphs, extension point signatures
2. Cluster projects by structural similarity
3. Candidates = clusters that don't match any known pattern signature
4. LLM examines cluster members: "what structural decision do these share and why?"
5. Human validation gate → confirmed candidates join the known pattern library
6. Known library grows → future clustering only surfaces truly novel candidates
   (virtuous loop: signal improves as the garden grows)

### Mechanism 2: Delta Analysis (Secondary — provides origin story)

1. Monitor major version changes across indexed projects (git tags)
2. When a project introduces a new abstraction layer between versions, LLM reads
   the diff: "what architectural decision was made here?"
3. Cross-reference with structural clustering: does this delta match a known cluster?
4. If novel → candidate pattern with exact origin (project, version, date, author)

### Combined Provenance

Clustering finds *what*. Delta finds *when and where*. Git blame finds *who*.
A confirmed pattern enters the garden with full provenance:

```
Structural cluster: 8 projects share this abstraction
    → Delta: first introduced in serverless-workflow v1.4, 2022-03-14
    → Git blame: @fabian-martinez
    → LLM: named "Pluggable Evaluator with Swappable Persistence"
    → Human validation: approved
    → patterns-garden entry with complete provenance chain
```

---

## Project Registry

A curated index of projects monitored for pattern discovery.

### Per-project record

```yaml
project: serverless-workflow
url: https://github.com/serverlessworkflow/specification
domain: jvm
primary_language: java
frameworks: [quarkus, spring]
last_processed_commit: abc1234
notable_contributors:
  - fabian-martinez
  - sanne-grinovero
```

`last_processed_commit` enables incremental processing and makes historical backfill
safe — backfill is just processing commits before the current marker.

---

## Trend Data Architecture

### Critical design decision: timestamped events, not aggregates

Every trend data point carries two timestamps:

```yaml
observed_at: "2022-03-14"   # git commit date — when the pattern appeared in the world
indexed_at: "2026-04-18"    # when the system processed it
```

Using only `indexed_at` makes historical backfill a painful migration.
Separating them makes backfill free — insert events with historical `observed_at`.

### Mining strategy

| Step | What | Cost | When |
|------|------|------|------|
| HEAD scan | Pattern presence + frequency across all indexed projects | Low | Initial index + periodic |
| Git blame on pattern files | Author attribution (targeted, not full history replay) | Low | At pattern discovery |
| Event-driven | Re-scan on new commits/releases | Low | Continuous |
| Recency weighting | Recent adoption outranks old in scoring | n/a | At query time |

### Stability signal

Patterns consistent across the HEAD of many projects are durable. Stability of
presence across many projects' HEAD = proven utility, weighted higher than frequency
alone.

### Retrofitting historical trend data

Start clean. Historical trend can be backfilled later via major-version tag sampling:

```
for each project in registry:
    git log --tags --simplify-by-decoration   # major versions only
    for each version tag:
        git checkout <tag>
        run pattern detection
        insert event with observed_at = tag.date   # idempotent
        git checkout HEAD
```

Pattern detection is idempotent — safe to run, pause, resume, or re-run at any time.
Full history for large projects is expensive; monthly sampling or major-version-only
sampling gives the trend shape at a fraction of the cost.

---

## Developer Profiles and "Code Like X"

### Profile structure (inferred from git attribution)

```yaml
developer: sanne-grinovero
domains: [jvm, data-grids, performance]   # inferred from project membership
patterns_authored:
  - pattern_id: pluggable-evaluator
    role: innovator
    variant_preferred: event-sourced
patterns_adopted:
  - pattern_id: cqrs
    frequency: 8
    variant_preferred: event-sourced
```

Profiles are inferred passively from git attribution. Developers can optionally
validate or curate their own profile.

### Retrieval personalisation

"Code like Sanne Grinovero" applies a **ranking boost**, not a filter:
- Patterns she originated or consistently adopts score higher
- Her preferred variants surface first
- Her decisions-garden entries get weighted up
- Other patterns remain discoverable — monoculture prevented

Profiles are subscribable. Others can adopt a developer's profile as a retrieval lens.

### Credibility cascade

A pattern originated by domain experts and adopted by respected peers is higher
signal than frequency alone. Developer reputation flows into pattern quality scoring.

---

## What This Design Enables

- **"How does project X solve Y?"** — query patterns-garden with project filter
- **"Is this pattern growing or declining?"** — trend data query
- **"Who invented this and where did it spread?"** — provenance chain
- **"Code like Sanne Grinovero"** — developer profile retrieval boost
- **"Which JVM experts are adopting this pattern?"** — developer attribution + trend
- **"What patterns is this project known for?"** — project registry + pattern attribution

---

## Deferred to Later Areas

| Topic | Area |
|-------|------|
| How the consumption layer routes queries across gardens | Area 5 |
| Structural clustering algorithm detail | patterns-garden implementation plan |
| Project registry schema and tooling | patterns-garden implementation plan |
| Developer profile schema | patterns-garden implementation plan |
| `submission-formats.md` updates for new entry types | implementation |
| Qdrant collection naming per garden | Area 1 validation |

---

## Relationship to Existing Design Docs

| Doc | Relationship |
|-----|-------------|
| `2026-04-16-garden-ecosystem-plan.md` | Master plan — Area 2 now supersedes its taxonomy section |
| `2026-04-16-knowledge-dimensions.md` | Session notes and research — this spec is the locked design |
| `2026-04-16-garden-retrieval-service.md` | Area 1 — retrieval stack unchanged by this taxonomy |
