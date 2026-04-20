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
evolution-garden/        ← version paths, deprecation, evolution trajectories
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

## Major Finding: Knowledge-Type-First, Not Technology-Domain-First

**This is a critical architectural correction.**

### What was assumed

Gardens organised by **technology domain** first:
```
jvm-garden     → java, quarkus, kotlin entries
tools-garden   → tools, cli, git entries
python-garden  → python entries
```

### What is actually right

**Knowledge type is the primary organizing axis. Technology domain is metadata.**

A Hibernate gotcha and a Strangler Fig migration pattern are fundamentally different
kinds of knowledge. They need different editorial bars, different curation communities,
different retrieval semantics, different staleness thresholds. Putting them in the same
garden because they "both relate to Java" is the wrong neighbour relationship.

```
# Wrong (technology-first)      # Right (knowledge-type-first)
jvm-garden                      discovery-garden
tools-garden                    patterns-garden
python-garden                   assessment-garden
                                transformation-garden
                                risk-garden
```

### What changes

| Decision | Technology-first | Knowledge-type-first |
|---|---|---|
| Top-level partition | Technology (jvm, tools, python) | Knowledge type (discovery, patterns, assessment) |
| Canonical garden name | `jvm-garden` | `discovery-garden` |
| Domain role | Organizing structure | Metadata / payload filter |
| Curation community | Java people curate jvm-garden | Practitioners curate discovery-garden; architects curate patterns-garden |
| Editorial bar | One bar per technology garden | Different bar per knowledge type |
| Child garden extends | A technology garden | A knowledge type garden |
| Qdrant partition key | domain (java, tools) | knowledge_type (discovery, pattern, assessment) |

### What stays valid

The current `Hortora/garden` git repo is essentially the **discovery-garden** — it has
gotchas, techniques, undocumented. The technology domain directories (java/, tools/) become
metadata/filters rather than structural. The git structure can stay; the conceptual model
changes.

The Qdrant partitioning insight (domain as sharding key within a collection) still holds —
just within `discovery-garden`, domain partitions the collection for performance.

---

## Knowledge Capture Modalities

Knowledge enters the garden through four distinct modalities. All four produce the same
garden entry format. The pipeline and quality mechanism differ.

| Modality | Source | Trigger | Quality mechanism | Volume |
|---|---|---|---|---|
| **Session capture** (forage skill) | Developer insight during coding | Human notices something non-obvious | Human validates before submission | Low, high quality |
| **Mining** (code miner) | Existing codebases | Scheduled scan or triggered | LLM extracts + human review gate | Medium, variable quality |
| **Migration capture** | Migration sessions | Sprint retrospective or live session | Experienced practitioners capture as it happens | Medium, high quality |
| **Support ticket ingestion** | Ticketing system (Zendesk, Jira SM, ServiceNow, Linear) | Batch pipeline or on-close webhook | Aggressive filtering + LLM normalisation + review | High volume, low signal density |

### Which dimension each modality primarily feeds

| Modality | Primary dimension | Secondary |
|---|---|---|
| Session capture | Discovery (gotchas, techniques) | Pattern (techniques) |
| Code mining | Pattern (code examples, architecture) | Discovery (bugs found in review) |
| Migration capture | Transformation + Risk | Discovery (migration-specific gotchas) |
| Support tickets | Discovery + Risk | Temporal (version-specific bugs) |

---

## Support Ticket Ingestion

Support tickets are **customer-sourced knowledge** — different epistemic source from
developer-sourced gotchas. A developer writing a gotcha has deep context on *why*. A
support ticket has the *symptom* (from the customer) and the *resolution* (from the
support engineer). Together they are often more valuable than either alone.

### Pipeline

```
Ticketing system API (Zendesk, Jira SM, ServiceNow, Linear, GitHub Issues...)
        │
        ▼ Pre-filter (cheap, no LLM)
   Keep only:
   - Resolved tickets (no resolution = no knowledge yet)
   - Software issues (not hardware / environment-only)
   - Not already a duplicate of an open ticket
        │
        ▼ Group similar tickets (before any LLM call)
   Cluster by: title similarity + tags + product area
   50 tickets about the same issue → 1 LLM call, not 50
   Use BM25 title match + tag overlap for cheap clustering
        │
        ▼ LLM pipeline (Claude API or Ollama)
   For each cluster:
   - Strip PII (names, companies, emails, account IDs, internal URLs)
   - Generalise (specific → universal):
     "Our UserService fails on prod-us-east-1" →
     "Custom service injection fails when..."
   - Extract: symptom, root cause, fix, why non-obvious
   - Score against 5-dimension editorial criteria
   - Classify: gotcha / technique / undocumented / temporal
   - Draft garden entry in standard format
        │
        ▼ Editorial filter
   Score < 8  → discard (most tickets — customer-specific, already documented)
   Score 8-11 → queue for human review
   Score 12+  → auto-submit with human spot-check
        │
        ▼ Duplicate check (retrieval API)
   Query garden with ticket summary text
   Similarity > threshold → mark as REVISE candidate, not new entry
        │
        ▼ Human review gate (for score 8-11)
   Accept / Edit / Reject
        │
        ▼ Garden entry committed and indexed
```

### Unique challenges

**Ticket grouping is critical.** Without it: 200 LLM calls about the same issue,
200 near-duplicate entries. Cluster first using cheap similarity before touching an LLM.
One cluster = one entry attempt.

**Generalisation is the hardest step.** Support tickets are hyper-specific. The LLM
must abstract from "our prod cluster" to "Quarkus 3.x when X condition holds." The LLM
is very good at this — it is essentially summarisation + abstraction — but the prompt
must be explicit about the generalisation requirement.

**Signal-to-noise ratio is low.** Expect 5–10% of tickets to produce garden-quality
entries. The rest fail the editorial bar: already documented, customer-specific
environment, resolved by config not code, too narrow to be universal. This is fine —
the pipeline is cheap once grouped, and the 10% that pass are high-value real-world
knowledge.

**"Fixed in newer version" tickets become temporal knowledge.** Ticket thread contains
both symptom AND resolution version. These produce temporal/evolutionary entries:
"In Quarkus 3.8.x, this bug exists; fixed in 3.9.0." High value, often missed.

**PII removal is mandatory before any LLM call.** Customer names, company names,
account IDs, internal URLs, email addresses — all must be stripped or pseudonymised
before the ticket content reaches the LLM. Can be done with pattern-matching before
the LLM step.

### Expected entry types from support tickets

| Ticket type | Garden entry type | Dimension |
|---|---|---|
| Unexpected behaviour, resolved by workaround | Gotcha | Discovery |
| Bug fixed in patch version | Temporal note (version X broken, Y fixed) | Temporal |
| "How do I do X?" → resolution teaches a pattern | Technique | Discovery |
| Integration fails in specific combo | Gotcha | Discovery |
| Performance degradation pattern | Risk + Gotcha | Risk / Discovery |
| Migration issue during customer upgrade | Transformation gotcha | Transformation |

---

## Open Questions for Second Opinion

1. Is the 10-dimension taxonomy complete? Are there important dimensions missing?

2. **Knowledge-type-first organisation confirmed?** Should gardens be named by knowledge
   type (discovery-garden, patterns-garden) or by technology domain (jvm-garden,
   tools-garden)? This is the most architecturally significant open question.

3. For migration knowledge specifically: is "Transformation" a distinct dimension or
   is it a sub-type within "Pattern"?

4. What's the right editorial bar for each dimension? Discovery has a clear bar
   ("would a skilled developer be surprised?"). Migration patterns, assessment
   frameworks, and risk knowledge are less clear.

5. How do cross-dimension queries work at retrieval time? How does the service know
   which gardens to fan out to for a given query?

6. Is there a natural order of maturity — start with discovery and patterns, add
   assessment and transformation later? Or does the taxonomy need to be established
   first before any garden is built?

7. For support ticket ingestion: what is the right clustering algorithm before LLM
   processing? How do you prevent the same underlying issue generating duplicate entries
   from different ticket clusters?

8. **Modality × dimension mapping** — should the capture modality be recorded as
   metadata on each entry? (session-captured gotcha vs ticket-sourced gotcha have
   different provenance and possibly different trust levels.)

---

## Academic and Industry Research Findings

*What does the literature say about RAG knowledge capture for software engineering?*

### What evidence says about knowledge types for code generation

**Few-shot retrieval beats raw documentation.** CEDAR (Nashid et al., ICSE 2023)
showed that highly relevant retrieved examples dramatically outperform random
selections — embedding quality and perplexity of examples matter more than count.
GPT-4o improved from 47.1% to 57.5% pass@1 with good few-shot examples; diminishing
returns after 5.

**Mixed knowledge types outperform single source.** Combining examples + gotchas +
Q&A beats any single type. Bug report summarisation research (arXiv 2512.00325) showed
integrating code snippets with bug summaries improves quality by 7.5–58.2% over
text-only.

**Security/vulnerability context is highest-signal.** Structured vulnerability context
reduced GPT-4o vulnerability rate from 41.3% to 5.2% — an 80% reduction. Suggests
risk/failure knowledge dimension has outsized impact on code safety.

**What developers actually search for** (Springer empirical study, 235 engineers,
5 continents): debugging/bug fixing is hardest and most searched, followed by
third-party code reuse, testing, and database queries. Validates priority:
**error/fix knowledge first, then examples, then patterns.**

### Validation of the 10-dimension framework

Basili's Experience Factory (1992, foundational SE knowledge management) categorises
organisational knowledge as: locally calibrated models (→ Pattern), proven processes
(→ Transformation), relevant product components (→ Code examples), and quality
baselines (→ Assessment). The 10 dimensions map onto this established model.

Systematic mapping of SE taxonomies (2017): most knowledge is classified via hierarchy
(53%) or faceted analysis (39%). Suggests garden structure should support faceted
filtering (knowledge type × domain × technology) not just hierarchy.

### The knowledge staleness gap

"Knowledge decay problem" (RAG About It, 2026): knowledge freshness is rarely managed
operationally. No academic paper directly measures knowledge half-life in software,
but evidence suggests API documentation becomes stale in 12–18 months. The research
community treats **outdated knowledge as a security vulnerability**, not a convenience
issue. Current garden design (staleness_threshold field, harvest REVIEW) is ahead of
what most production RAG systems do — this is a differentiator.

### Meta's tribal knowledge work (2026)

Meta used 50+ specialised AI agents to map tribal knowledge in large-scale data
pipelines — reading every file to extract: gotchas, workarounds, technical debt,
decision rationale. Validates: the discovery dimension is real, automated mining is
viable at scale, and the knowledge that most helps AI assistants is the knowledge
that currently lives only in developers' heads.

### Stack Overflow quality findings

StaQC dataset (148K Python, 120K SQL Q&A pairs): high-quality Q&A with runnable code
examples outperforms documentation. However: most low-quality SO questions get ignored
in practice — RAG systems trained on SO need aggressive quality filtering. Validates
the editorial bar approach.

### Key gap identified by research

**Context window management** — research shows 5 retrieved examples is about the
optimum; more is not better. The "recall mode" (return 5–8 full entries for LLM to
read) aligns with this. The LLM reranking approach is validated. What's not yet
designed: how does an AI assistant know WHICH gardens to query for a given coding task?
Proactive knowledge push vs reactive retrieval is an open problem in the literature.

---

## Session Notes

*Running log of decisions and insights as the design evolves.*

**2026-04-16:**

- **Major finding:** knowledge-type-first organisation, not technology-domain-first.
  Technology domain is metadata / filter, not structure. Top-level canonical gardens
  should be named by knowledge type (discovery-garden, patterns-garden) not technology
  (jvm-garden, tools-garden).

- **Code examples garden taxonomy parked.** The taxonomy should follow from the
  knowledge-type framework, not precede it.

- **Support ticket ingestion** identified as fourth capture modality (alongside
  session capture, code mining, migration capture). Customer-sourced knowledge.
  Pipeline: pre-filter → cluster similar tickets → LLM normalise → score → dedup →
  human review. Key challenge: generalisation from specific to universal.

- **Migration as distinct modality** — structured capture during migration projects,
  feeds transformation + risk dimensions primarily.

- **Scope confirmed as beyond Java/Quarkus** — framework must apply to any
  technology, any migration type, any organisation.

- **Research validates core design:** few-shot retrieval beats documentation; mixed
  knowledge types beat single source; 5 examples is optimum; staleness management
  is a differentiator (most RAG systems don't do it); security/risk knowledge has
  outsized impact.

- **TWO FILTERS applied to all 10 dimensions — SETTLED DECISION:**
  Only dimensions passing BOTH survive:
  1. Capture can be automated (session discovery OR code mining OR ticket/source mining)
  2. Retrieving it makes the AI produce better code/advice (genuinely RAG-able)

- **10 dimensions → 5 gardens (SETTLED):**

  | Garden | Primary auto-capture | RAG value |
  |---|---|---|
  | `discovery-garden` | Claude sessions + ticket mining | Non-obvious facts that prevent wasted hours |
  | `patterns-garden` | Code mining + Claude sessions | Copyable working solutions |
  | `evolution-garden` | Release note mining + sessions | Version-specific facts that change code correctness |
  | `risk-garden` | Ticket + post-mortem mining | Documented failure modes |
  | `decisions-garden` | Mine existing ADRs + sessions | Prevents AI suggesting rejected approaches |

- **Dropped (failed one or both filters):**
  - Assessment — not auto-capturable; process not fact, not RAG-able
  - Domain/semantic vocabulary — requires human business knowledge
  - Org-specific constraints — not auto-capturable
  - Organisational/contextual — fails both filters

- **Migration knowledge distributes across the 5** (no migration-garden needed):
  - Migration gotchas → discovery-garden
  - Strangler Fig, Branch by Abstraction → patterns-garden
  - Quarkus 2→3 breaking changes → evolution-garden
  - Dual-write ordering failures → risk-garden

- **Knowledge-type-first confirmed with 5 gardens** — technology domain is a
  metadata payload filter within each garden, not a structural organiser.
  Qdrant collection naming: `discovery_jvm`, `patterns_jvm`, `evolution_jvm` etc.

- **FIELD DESIGN DECISION — domain must be coarse, not fine-grained (REQUIRES MIGRATION):**

  Problem: `domain: quarkus` vs `domain: drools` vs `domain: quarkus-drools` is
  unresolvable. A Drools bug running on Quarkus may be in Drools engine, the
  quarkus-drools integration, or CDI lifecycle — often unknown at capture time,
  sometimes genuinely ambiguous. Fine-grained domain classification cannot be done
  reliably and shouldn't be attempted.

  Solution — separate the two responsibilities `domain` was conflating:

  | Field | Purpose | Values | Example |
  |---|---|---|---|
  | `domain` | Coarse Qdrant partition key ONLY | ~8 broad categories | `jvm` |
  | `stack` | Honest multi-layer "what was running" | Free text, always multi-layer | `"Quarkus 3.9.x, Drools 8.44.0, quarkus-drools"` |
  | `tags` | All relevant technology names | List, multi-layer | `[quarkus, drools, quarkus-drools, cdi]` |
  | `root_cause_layer` | Which layer the bug lives in | Optional, fill when certain | `drools` |

  Domain coarse values: `jvm`, `python`, `tools`, `web`, `data`, `infrastructure`,
  `security`, `cloud`

  Domain migration mapping (existing ~240 entries — scriptable):
  ```
  quarkus, java, drools, java-panama-ffm, casehub-engine, permuplate, apache-jexl → jvm
  python, beautifulsoup → python
  tools, claude-code, intellij-platform, electron, macos-native-appkit, scelight, approaches → tools
  ```

  THREE things require change:
  1. **Existing ~240 garden entries** — `domain` field value migration (scriptable)
  2. **forage SKILL.md** — "Determine the domain" table → coarse domains;
     `root_cause_layer` added as optional field
  3. **submission-formats.md** — entry templates updated; `root_cause_layer` optional

  Directory structure (java/, quarkus/, tools/) stays for GitHub browsability.
  The YAML `domain` field changes to coarse value. Directories and field decouple.

  `root_cause_layer` — fill when certain which layer the bug lives in. Leave blank
  when ambiguous. Entry is fully retrievable without it via stack and tags.

- **Consumption layer identified as design gap** — how does an AI know WHICH gardens
  to query? Proactive vs reactive retrieval not yet designed. Open problem in
  literature too.

- **Structured plan needed** — see `2026-04-16-garden-ecosystem-plan.md`.

- **Area 3 COMPLETE — all 5 garden capture pipelines designed (2026-04-17):**

  | Garden | Primary capture | Secondary |
  |---|---|---|
  | `discovery-garden` | Session (forage) + ticket mining | ✅ Built |
  | `decisions-garden` | Mine existing ADRs + new ADR webhook + session | ✅ Designed |
  | `patterns-garden` | Code mining (JavaParser + LLM minimise) + session | ✅ Designed |
  | `evolution-garden` | Release note/changelog mining + session | ✅ Designed |
  | `risk-garden` | Ticket mining (risk classifier) + post-mortem mining | ✅ Designed |

  **evolution-garden specifics:**
  - Sources: GitHub releases, Maven Central, npm, PyPI, official migration guides
  - Filter: breaking-change | deprecation (action needed) | new-capability. Skip bug fixes,
    internal refactoring, minor additions
  - New YAML fields: `from_version`, `changed_in`, `breaking: bool`, `migration_effort`
  - `staleness_threshold: 1095` (3 years, longer than discovery)
  - Trigger: GitHub release webhook (immediate) + nightly scan (non-webhook sources)

  **risk-garden specifics:**
  - Source 1: ticket mining — same pipeline as discovery, different LLM classifier
    (risk = production harm at scale, not just non-obvious behaviour)
  - Source 2: post-mortem documents — git-tracked markdown first, then GitHub Issues
    with incident labels, then Notion/Confluence via API
  - Generalisation step critical: "our service failed" → "connection pool exhaustion
    under load when exceptions don't release connections"
  - Community vs private split: universal patterns → community; org-specific → child garden
  - New YAML fields: `failure_pattern`, `observed_at_scale`, `severity`, `mitigation`
  - `staleness_threshold: 1825` (5 years — failure patterns are durable)

- **LITERATURE CHECK — discovery entry format validation (2026-04-17):**

  No direct academic validation of the 12-field template exists. Strong indirect support.

  *What is validated:*
  - Core structure (symptom, root cause, fix, context, stack, tags) maps directly to
    Aamodt & Plaza CBR Problem+Solution+Outcome (1994) — 30 years of validation
  - `constraints` field: supported by CBR adaptation and lessons-learned literature
  - Metadata fields (stack, tags): validated by Stack Overflow retrieval research
  - Hybrid structured+unstructured RAG: 13.1% better precision than prose alone

  *Important nuance — WHY fields serve comprehension, not retrieval:*
  - SPLADE/BM25 optimise for term overlap and semantic similarity, NOT reasoning
  - `rationale`, `why_non_obvious`, `alternatives_considered` are largely invisible
    to the retriever — they don't improve finding the right entry
  - But: they significantly help the LLM that READS the retrieved entry — it
    understands why the fix works and can apply context appropriately
  - Conclusion: WHY fields are justified but their value is post-retrieval
    (comprehension phase), not retrieval phase. Keep them optional, not required.

  *Basili / Google SRE warning:*
  - Both explicitly warn over-constrained templates reduce contribution rates
  - Too many required fields → contributors stop contributing
  - Validates current design: WHY fields are OPTIONAL (+1 bonus to score), not required

  *Redundancy risk:*
  - `rationale`, `why_non_obvious`, `alternatives_considered` overlap significantly
  - From retrieval perspective: noise. From comprehension: different readers
  - Watch for consolidation opportunity — three fields saying the same thing is waste

  *The real risk is not format quality but adoption:*
  - Literature says reuse is limited by distribution and process barriers, not structure
  - Format is sound; make it as low-friction as possible to contribute

  *Gap:* no published empirical comparison of template configurations for this use case.
  The format is theoretically grounded but not benchmarked against simpler alternatives.

- **TAXONOMY REVISION — 6 gardens, not 5 (2026-04-18):**

  `examples-garden` added as a distinct 6th garden, separated from `patterns-garden`.
  Rationale: different embedding model (code-specific vs semantic), different editorial
  bar ("any working minimal example" vs "proven architectural pattern"), different
  capture pipeline (code miner vs session/ADR mining), and volume imbalance at scale
  (code miner output would dominate patterns garden, degrading architectural pattern
  retrieval). Surfacing the distinction explicitly is better than hiding it inside
  sub-types.

  | Garden | Knowledge type |
  |--------|---------------|
  | `discovery-garden` | Gotchas, techniques, undocumented behaviours |
  | `patterns-garden` | Architectural, migration, integration patterns (prose) |
  | `examples-garden` | Minimal working code, intent-driven, copy-paste ready |
  | `evolution-garden` | Version-specific facts, breaking changes, deprecations |
  | `risk-garden` | Failure modes, post-mortems, production anti-patterns |
  | `decisions-garden` | ADRs, rejected approaches, settled choices |

- **IMPORTANT DESIGN DETAIL — `decisions-garden` operates at every level of the
  spanning-tree hierarchy (2026-04-18):**

  The apparent weakness of `decisions-garden` (most ADRs are project-specific, not
  universal) is resolved by the federation hierarchy itself. A `decisions-garden` is
  valid and valuable at every node:

  ```
  canonical decisions-garden          ← universal architectural reasoning
      └── quarkus-decisions-garden    ← Quarkus community choices + rationale
          └── acme-decisions-garden   ← ACME Corp's specific ADRs
  ```

  A developer querying their private garden walks the chain automatically — org ADRs,
  then community decisions, then universal reasoning. Each level is appropriately scoped.

  **Auto-capture path per level:**
  - Canonical: mine ADRs broadly from open-source projects across many domains
  - Community: mine that project's own repo (e.g., Quarkus ADRs from quarkus.io/quarkus)
  - Private: mine the org's own ADR directory (fully automated, zero friction)

  **Why this matters:** `decisions-garden` is the garden type that most clearly
  demonstrates the spanning-tree federation model's value. Every other garden type
  (discovery, patterns, examples) is primarily universal. Decisions are the knowledge
  type where the hierarchy adds the most precision — the closer to the leaf, the more
  specific and directly applicable the decision. Do not flatten this to "decisions
  belong only in private gardens" — the community level is where project-level ADRs
  (e.g., why Quarkus chose Vert.x, why Kubernetes chose etcd) live and provide value
  to that project's whole user community.

- **PATTERNS-GARDEN SIGNIFICANTLY EXTENDED (2026-04-18) — observed architectural
  patterns with provenance, suitability, variants, frequency, and trend:**

  The patterns-garden is not just "known design patterns" (GoF, CQRS). It captures
  **observed architectural patterns from real production codebases** — patterns
  discovered by studying how projects actually solve structural problems, with enough
  context for a developer to decide whether and how to apply them.

  **Extended entry format for patterns-garden:**
  - `observed_in: [{project, url, path, first_seen, last_seen}]` — provenance chain
  - `suitability` — when this pattern works, when it doesn't, constraints
  - `variants: [{name, description, tradeoffs}]` — base pattern + named adaptations
  - `variant_frequency: {event-sourced: 31, in-memory: 12}` — which variants are winning
  - `authors: [{github_handle, role: originator|adopter|innovator}]` — developer attribution
  - `stability` — computed: consistent presence across major versions = durable pattern

  **New capture modality: ecosystem mining (richer than code miner)**
  The code miner finds pattern *instances* (builders, multi-datasource configs) and
  minimises them to examples. Ecosystem mining finds the *architectural decisions* of
  a project — what's pluggable, what's abstracted, why — and extracts them as patterns
  with suitability guidance. These are different tasks requiring different LLM prompting.

- **PROJECT REGISTRY — new concept (2026-04-18):**

  A curated index of projects being monitored for pattern discovery. Each entry:
  - Project URL, domain, primary language/framework
  - `last_processed_commit` — for incremental processing and backfill tracking
  - Developer contributors of interest (for attribution)

  The registry is the corpus against which ecosystem mining runs. Starting with
  well-known open-source projects in relevant domains (Hibernate, Quarkus, Spring,
  Kubernetes, etc.).

- **PATTERN EPIDEMIOLOGY — frequency, trend, developer attribution (2026-04-18):**

  Patterns carry adoption analytics: how often a pattern and its variants appear
  across the project registry, who championed it, and whether adoption is growing
  or declining.

  **Trend data design — critical decision:**
  Store trend data as **timestamped events**, not aggregates. Two timestamps on every
  data point:
  - `observed_at` — the git commit date (when the pattern actually appeared in the world)
  - `indexed_at` — when the system processed it

  Using only `indexed_at` (the lazy default) makes historical backfill a migration.
  Separating them makes backfill free — just insert events with historical `observed_at`.

  **Mining strategy — start clean, retrofit history later:**
  - **HEAD scan** — mine current state of all indexed projects (pattern presence + frequency)
  - **Git blame on pattern files** — targeted attribution, cheap (not full history replay)
  - **Event-driven going forward** — webhook/release triggers re-scan, trend accumulates naturally
  - **Recency weighting** — recent adoption outranks old adoption in scoring
  - **Stability signal** — patterns consistent across HEAD of many projects = durable, weight higher
  - **Retrofittable** — full history can be backfilled later via major-version tag sampling;
    idempotent detection makes this safe to run at any time, pause, and resume

  **What you lose by starting clean:** pre-index trend history. Acceptable — "this
  pattern peaked in 2019" requires distinguishing "still used" from "never cleaned up",
  which is too noisy to get right reliably from raw git history.

- **"CODE LIKE X" RETRIEVAL MODE — developer profiles as ranking boost (2026-04-18):**

  Developer coding profiles, inferred from git attribution across the project registry:
  - Which patterns a developer uses most frequently
  - Which variants they prefer
  - Which domains they work in (inferred from project membership)

  **"I want to architect like Sanne Grinovero"** — the retrieval service applies a
  ranking boost based on her profile. Patterns she originated or consistently adopts
  score higher. Her preferred variants surface first. Her decisions-garden entries
  get weighted up. This is a **boost, not a filter** — discovery is preserved.

  Developer profiles are subscribable entities. A respected developer can curate or
  validate their own profile. Others subscribe to use it as a retrieval lens.

  **Monoculture risk:** must be a boost, never an exclusive filter. The system
  surfaces "coding like X" as a strong prior, not as the only retrieval path.

  **Trust signal:** a pattern originated by Sanne Grinovero, adopted by five other
  JVM performance experts, spreading to 40 projects = extremely high-signal provenance.
  Developer reputation cascades into pattern quality scoring.

  **Combined with trend:** "this pattern is growing, and the growth is led by the
  top five JVM performance engineers" — not just a trend, a credibility signal.
