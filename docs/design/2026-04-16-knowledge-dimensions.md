# Knowledge Dimensions — Garden Ecosystem Design

**Date:** 2026-04-16
**Status:** Draft — taxonomy under active design
**Context:** Broad perspective on what an AI coding assistant needs to work effectively
across any software domain, including systems and application migrations.

---

## Starting Point: What We Have

Two gardens designed so far:

1. **Knowledge garden** — non-obvious technical facts (gotchas, techniques, undocumented).
   Short markdown entries, prose-centric, editorial bar: "would this genuinely surprise a
   skilled developer?" Universal applicability across any technology stack.

2. **Code examples garden** — minimal use-case-driven code snippets (draft, taxonomy
   underdesigned). Code-centric, intent-driven retrieval ("I want to do X").

These cover one dimension well (discovery knowledge) and one dimension partially
(pattern knowledge). A broader perspective reveals many more.

---

## The Full Taxonomy of Knowledge Dimensions

### Dimension 1: Discovery Knowledge ✅ (current garden)

What's non-obvious, undocumented, or surprising about a technology. The editorial bar
("would this surprise a skilled developer?") travels well across any domain. Applies
equally to Java, Python, infrastructure, databases, cloud platforms.

Types: gotcha, technique, undocumented.

### Dimension 2: Pattern Knowledge ⚠️ (underdesigned)

Recurring solutions to recurring problems. Much broader than "code examples":

| Pattern type | Description | Example |
|---|---|---|
| Design patterns | GoF, enterprise, reactive | Builder, Factory, Observer |
| Architectural patterns | System structure | CQRS, Event Sourcing, Saga |
| **Migration patterns** | How to move from A to B | Strangler Fig, Branch by Abstraction, Big Bang Rewrite, Parallel Run, Dual Write |
| Integration patterns | System boundaries | Anti-corruption layer, Gateway, Adapter |
| Data patterns | Data movement and consistency | Change data capture, event sourcing, dual write |
| Testing patterns | How to verify correctness | Contract testing, property-based, golden master |
| Security patterns | Security-by-design | Zero-trust, defence in depth |

Migration patterns are a **first-class knowledge category**, not a subset of code examples.

### Dimension 3: Decision Knowledge

What was chosen and why, what was explicitly rejected. Travels across any scale of
decision — technology choice, architectural choice, migration strategy choice.

Current form: ADRs (Architecture Decision Records). Could be RAG-indexed so an AI
assistant knows "we already considered and rejected this approach."

Key property: prevents the AI suggesting exactly what was already evaluated and dismissed.

### Dimension 4: Assessment Knowledge ← new, important for migrations

How to evaluate the state of a system **before** acting on it.

Examples:
- "10 questions to ask before decomposing a monolith"
- "How to identify bounded contexts in a legacy codebase"
- "How to measure migration readiness"
- "How to estimate strangler fig effort"

An AI needs this knowledge BEFORE writing any code on a migration engagement. Without
it, it gives technically correct advice for the wrong starting point.

### Dimension 5: Transformation / Migration Knowledge ← new, distinct from patterns

How to move from state A to state B at any scale — application migrations, infrastructure
migrations, data migrations. Process and sequence, not just structure.

| Sub-type | Description |
|---|---|
| Application migration | Monolith → microservices, framework migration, language port |
| Infrastructure migration | On-prem → cloud, container adoption, database migration |
| Data migration | Schema evolution, ETL, zero-downtime cutover, dual-write phase |
| Team migration | Org structure change, ownership transfer, knowledge handover |

Key questions this answers: In what order do you tackle things? How do you maintain
consistency during cutover? What does rollback look like? What's the minimum viable
first slice?

Distinct from pattern knowledge: **patterns are structural; transformation knowledge is
process and sequence.**

### Dimension 6: Domain / Semantic Knowledge

What things mean in a specific context. Business vocabulary, regulatory concepts,
industry-specific models.

Examples:
- "When we say 'case' we mean a CaseHub workflow entity, not a general concept"
- "In banking, 'settlement' means T+2 net settlement, not gross"
- "A 'user' is the authenticated principal, not the end customer"

Without this, an AI produces technically correct code that is semantically wrong for
the domain. An AI coding a trading system that doesn't know domain vocabulary makes
errors that compile and test correctly but violate business rules.

### Dimension 7: Constraint Knowledge

What cannot be done, or must be done, for non-technical reasons.

| Constraint type | Examples |
|---|---|
| Regulatory | GDPR: all PII encrypted at rest; HIPAA: audit log every access |
| SLA / performance | Response time < 200ms under contract; 99.9% uptime obligation |
| Security mandates | All secrets in vault; no plaintext credentials ever |
| Compliance | SOC2: full audit trail; PCI-DSS: card data never on application tier |
| Operational | 3 replicas → no local cache; blue-green → backward-compatible migrations |

These are as architectural as any technical decision, but often not captured anywhere
a coding assistant can find them.

### Dimension 8: Temporal / Evolutionary Knowledge

How things change over time. Version migration paths, deprecation trajectories,
end-of-life timelines.

Examples:
- "Quarkus 2.x → 3.x: what breaks, what's newly available"
- "Java 11 → 21: virtual threads, pattern matching, sequenced collections"
- "This library will be unsupported in 18 months — avoid new adoption"
- "Spring Boot 2 → 3: Jakarta EE namespace migration required"

An AI that knows evolution trajectories gives **forward-compatible advice** rather than
recommending patterns that will need migration in 12 months.

### Dimension 9: Risk / Failure Knowledge

What goes wrong and how it manifests. Distinct from gotchas:

| | Gotcha | Risk / failure knowledge |
|---|---|---|
| Scope | Universal (any project using this tech) | Contextual (this class of decision) |
| Type | "This API behaves unexpectedly" | "This architectural choice leads to outages" |
| Source | Discovering a non-obvious behaviour | Pattern from post-mortems and incidents |
| Example | "@PreUpdate fires at flush not persist" | "Dual-write migrations fail at ordering assumptions 70% of the time" |

Failure knowledge codifies collective learning from production incidents and migration
post-mortems. High value for migrations where the same failure modes repeat.

### Dimension 10: Organisational / Contextual Knowledge

What's true about a specific team, history, and context. Not universally applicable —
specific to one organisation or project.

| Sub-type | Examples |
|---|---|
| Team conventions | "We use constructor injection always"; "Tests must use @QuarkusTest" |
| Failure history | "The March outage was caused by X pattern in our context" |
| Ownership | "Team A owns payment, Team B owns auth" |
| Operational context | "We run 3 replicas, 512MB heap, blue-green deploys" |
| API contracts | "The payment service returns 202 not 200 on success" |

This is the least shareable dimension (org-specific) but highest precision — the AI
knows what's actually true for your environment, not just what's generally true.

---

## The Matrix

```
                    Universal                    Organisational
                    (community gardens)          (child/private gardens)
                    ─────────────────────────    ──────────────────────────
Technical           Discovery (gotchas,           Failure history,
                    techniques, undocumented)     API contracts

Structural          Pattern (design, arch,        Team decisions / ADRs,
                    migration, integration)       "Why not X" decisions

Process             Assessment (evaluation        Project conventions,
                    frameworks, readiness)        team practices
                    Transformation (migration
                    process, sequencing)

Semantic            Domain vocabulary             Org-specific vocabulary,
                    (industry patterns)           project ubiquitous language

Evolutionary        Temporal (version paths,      Org technology roadmap,
                    deprecation, EOL)             migration timeline

Operational         Risk (failure modes,          Operational constraints,
                    recovery patterns)            SLAs, compliance
```

**Left column:** shareable, broadly applicable, community-curated, editorial bar for quality.
**Right column:** not shareable beyond the org, high precision, captured in child gardens.

---

## Implications for Garden Structure

### Current assumption (wrong at scale)

Gardens organised by **technology domain** first:
```
java/
quarkus/
tools/
python/
```

### Better: organised by **knowledge type** first, technology within

```
discovery-garden/       ← gotchas, techniques, undocumented (any technology)
patterns-garden/        ← design, architectural, migration, integration patterns
assessment-garden/      ← evaluation frameworks, readiness checklists
transformation-garden/  ← migration processes, transformation strategies, sequencing
risk-garden/            ← failure modes, recovery patterns, post-mortems
temporal-garden/        ← version paths, deprecation, evolution trajectories
```

Code examples fit **inside** `patterns-garden` — not a top-level concept.

Each garden is queryable independently or together. A migration engagement query
might hit assessment-garden (readiness), transformation-garden (strategy),
patterns-garden (Strangler Fig), and discovery-garden (Java gotchas) simultaneously.

### Why the federation model handles this

Each specialised garden:
- Has its own Qdrant collection optimised for its retrieval semantics
- Has its own editorial bar (migration patterns ≠ code gotchas)
- Has its own curation community (architects for patterns, practitioners for gotchas)
- Can be queried independently or in combination

The `garden-retrieval` service fans out to multiple gardens when a query spans
knowledge types, merges results via RRF, and returns a coherent ranked set.

---

## Migrations: A Cross-Cutting Example

A migration engagement requires at least five dimensions simultaneously:

| Phase | Dimensions needed |
|---|---|
| **Before** | Assessment (is this system ready?), Temporal (what does target look like in 2 years?) |
| **Strategy** | Pattern (which migration pattern?), Decision (what have others chosen and why?) |
| **Execution** | Transformation (what order, what sequence?), Discovery (what gotchas exist in this stack?) |
| **Risk** | Risk (what typically goes wrong?), Constraint (what can't we do?) |
| **After** | Temporal (is target state forward-compatible?), Domain (did semantic correctness survive?) |

No single garden serves this. The federation of specialised gardens, with unified
retrieval, is the only architecture that works at this scope.

---

## What's Settled vs What Needs Design

### Settled ✅

- Infrastructure: Qdrant + SPLADE + Ollama + Quarkus native + LangChain4J
- Federation: per-garden service stack, service-to-service upstream queries
- Partitioning: domain as sharding key at Qdrant level
- Two retrieval modes: recall (LLM downstream) and precision (human downstream)
- Git as source of truth; Qdrant as derived search index

### Needs design ⚠️

- **Top-level taxonomy agreed** — the 10 dimensions above are a starting point, not a
  final answer. Which dimensions become first-class gardens? Which are sub-types within
  a garden? Which are metadata fields on entries rather than separate gardens?

- **Code examples garden taxonomy** — "design patterns / use cases / integrations" was
  proposed but may be wrong. The taxonomy should follow from the broader dimension
  framework, not precede it. **Parked pending taxonomy agreement.**

- **Retrieval semantics per dimension** — migration pattern retrieval is different from
  gotcha retrieval. Assessment knowledge retrieval is different from code examples
  retrieval. Each dimension may need tuned retrieval parameters (limit, mode, model).

- **Editorial bar per dimension** — the "would a skilled developer be surprised?" bar
  works for discovery knowledge. What's the bar for transformation knowledge? For risk
  knowledge? For assessment knowledge?

- **Cross-dimension queries** — when a query spans multiple gardens (migration + gotchas
  + patterns), how are results merged? Simple RRF across all? Weighted by dimension?

- **Organisational garden structure** — how does a company structure its private child
  gardens across these dimensions? One per dimension? One per project? One per team?

---

## Open Questions for Second Opinion

1. Is the 10-dimension taxonomy complete? Are there important dimensions missing?

2. Should gardens be organised by knowledge type (proposed above) or by technology
   domain (current assumption)? Or is a matrix (type × domain) the right structure?

3. For migration knowledge specifically: is "Transformation" a distinct dimension or
   is it a sub-type within "Pattern"?

4. What's the right editorial bar for each dimension? Discovery has a clear bar.
   Migration patterns, assessment frameworks, and risk knowledge are less clear.

5. How do cross-dimension queries work at retrieval time? How does the service know
   which gardens to fan out to for a given query?

6. Is there a natural order of maturity — start with discovery and patterns, add
   assessment and transformation later? Or does the taxonomy need to be established
   first before any garden is built?
