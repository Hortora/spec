# Garden Ecosystem — Structured Plan

**Date:** 2026-04-16
**Purpose:** Master list of design and build areas. Use this to structure conversations
and ensure nothing gets missed or skipped.

---

## Overview

Seven distinct areas. Each is tracked with status, key decisions outstanding,
and what needs to happen before it can be built or finalised.

| # | Area | Status | Blocking |
|---|---|---|---|
| 1 | Scalable RAG backend | ✅ Designed, ⚠️ validation needed | 3 critical validations |
| 2 | What to capture (knowledge model) | ✅ Settled — 5 gardens | — |
| 3 | Capture modalities and structures | ✅ All 5 pipelines designed | — |
| 4 | Knowledge lifecycle and governance | ❌ Not designed | — |
| 5 | Consumption and integration layer | ❌ Not designed | Area 1 validation |
| 6 | Measurement and evaluation | ❌ Not designed | — |
| 7 | Privacy, security, and deployment | ⚠️ Partially touched | — |

---

## Area 1: Scalable RAG Backend

**What it is:** The retrieval infrastructure — Qdrant + SPLADE + Ollama + Quarkus native
`garden-retrieval` service. Designed in detail.

**Reference:** `docs/design/2026-04-16-garden-retrieval-service.md`

**Status:** Architecture decided. Three critical validations outstanding before building.

### What's decided

- Qdrant (self-hosted, open-source) for hybrid vector search
- SPLADE (`prithivida/Splade_PP_en_v1`, Apache 2.0) via ONNX Runtime for sparse vectors
- Ollama (`nomic-embed-text`) for dense embeddings
- LangChain4J + Quarkus native image for the retrieval service
- Two retrieval modes: recall (LLM downstream, 5–8 full entries) and precision (human,
  1–3 reranked cards)
- Per-garden Qdrant instance; domain as partition key within collections
- Knowledge-type-first collection naming (discovery-garden not jvm-garden)
- Federation via service-to-service HTTP; each garden runs its own Qdrant + service

### Critical validations (must do before building)

1. **SPLADE ONNX in Quarkus native image** — JNI + GraalVM on macOS ARM. If it fails,
   precision mode falls back to Qdrant-ranked results without reranking.
2. **LangChain4J dual-client** — LangChain4J (dense) + Qdrant Java client (sparse)
   in the same application, no state conflicts.
3. **IDF modifier + SPLADE** — `"modifier": "idf"` was designed for BM25 TF vectors.
   With SPLADE output (already semantically weighted), interaction may be unexpected.
   May need to disable IDF modifier when using SPLADE.

### Key review question for second opinion

Is SPLADE appropriate for short technical entries (50–200 lines), or should BM25 be
benchmarked first? Research suggests mixed knowledge types > single type, but doesn't
compare BM25 vs SPLADE for this specific content profile.

---

## Area 2: What to Capture (Knowledge Model)

**What it is:** What types of knowledge go into the garden ecosystem.

**Reference:** `docs/design/2026-04-16-knowledge-dimensions.md`

**Status:** ✅ SETTLED. Two filters applied (auto-capturable + RAG-able). 10 dimensions
reduced to 5 gardens. Knowledge-type-first organisation confirmed.

### The 5 gardens (SETTLED)

Two filters applied to all candidate dimensions:
1. **Capture must be automated** — session discovery, code mining, or ticket/source mining. No human manual effort.
2. **Must improve AI output when retrieved** — genuinely RAG-able, not process/methodology.

| Garden | Auto-capture source | RAG value | Build status |
|---|---|---|---|
| `discovery-garden` | Claude sessions + ticket mining | Non-obvious facts that prevent wasted hours | ✅ Existing Hortora garden (rename + restructure) |
| `patterns-garden` | Code mining + Claude sessions | Copyable working solutions | ⚠️ Code miner designed, not built |
| `evolution-garden` | Release note mining + sessions | Version-specific facts that change code correctness | ❌ Not started |
| `risk-garden` | Ticket + post-mortem mining | Documented failure modes | ❌ Not started |
| `decisions-garden` | Mine existing ADRs + sessions | Prevents AI suggesting already-rejected approaches | ❌ Not started |

### What was dropped and why

| Dropped | Reason |
|---|---|
| Assessment | Not auto-capturable; process not fact; not RAG-able |
| Domain / semantic vocabulary | Requires human business knowledge to capture |
| Org-specific constraints | Not auto-capturable |
| Organisational / contextual | Fails both filters |
| Transformation (as standalone) | Distributes: gotchas → discovery, patterns → patterns, facts → temporal, failures → risk |

### Qdrant collection naming (knowledge-type-first)

Technology domain is a metadata filter within each garden, not a structural organiser.

```
discovery_java        discovery_tools       discovery_python
patterns_quarkus      patterns_java
evolution_java        evolution_quarkus
risk_java             risk_quarkus
decisions_jvm
```

### Major finding to validate

**Knowledge-type-first organisation, not technology-domain-first.** Gardens should
be named by knowledge type (discovery-garden, patterns-garden) not technology
(jvm-garden, tools-garden). Technology domain is metadata / filter, not structure.
This changes the entire garden federation architecture.

### Key questions to resolve

1. Is the 10-dimension taxonomy complete? What does the literature say is missing?
2. Which dimensions are highest priority? Research says: discovery + risk have
   outsized impact; examples > documentation; mixed types beat single type.
3. Which dimensions become first-class gardens vs metadata fields vs sub-types?
4. What is the editorial bar for each dimension? (Discovery is clear; others are not.)
5. How do cross-dimension queries fan out and merge?

### Research task outstanding

Web + Google Scholar research partially done (see knowledge dimensions doc).
Key remaining question: is there a validated taxonomy of software engineering
knowledge types that we can adopt rather than invent?

---

## Area 3: Capture Modalities and Structures

**What it is:** The four ways knowledge enters the garden, and the different structures
each requires.

**Reference:** `docs/design/2026-04-16-knowledge-dimensions.md` (modalities section),
`docs/design/2026-04-16-code-examples-garden.md` (code mining design, parked)

**Status:** Four modalities identified. Session capture is built. Others are designed
at varying levels. Code examples garden structure is parked.

### The four modalities

| Modality | Status | What's designed | What's missing |
|---|---|---|---|
| **Session capture** (forage skill) | ✅ Built | Full CAPTURE/SWEEP/SEARCH/REVISE workflow | Extends to new dimensions when taxonomy agreed |
| **Code mining** | ⚠️ Designed (parked) | CLI tool concept, JavaParser AST, LLM minimisation | Structure of code examples garden (parked) |
| **Migration capture** | ⚠️ Concept only | Identified as distinct modality | How sessions are structured, tooling |
| **Support ticket ingestion** | ⚠️ Pipeline designed | Full pipeline: filter → cluster → LLM → score → review | Integration with specific ticket systems, PII handling |

### Code examples garden (parked)

Structure of the code examples garden (design patterns / use cases / integrations)
is parked. The taxonomy must follow from the knowledge-type framework (Area 2),
not precede it. Specifically: does "design patterns" map to the Pattern dimension?
Does "use cases" map to a new sub-type? Resolve Area 2 taxonomy first.

### Key questions to resolve

1. What is the right structure for the code examples garden? (Blocked on Area 2)
2. How is migration capture structured? Is it a variant of session capture, or a
   distinct workflow? Does it have its own garden, or use the same gardens with
   metadata flagging migration context?
3. What ticket systems should support ticket ingestion support first? What's the
   MVP pipeline (read-only export? webhook? API polling?)
4. How does a modality record its provenance on an entry? Should entries carry
   `captured_via: session|mined|migration|ticket`? Does this affect trust or retrieval?

---

## Area 4: Knowledge Lifecycle and Governance

**What it is:** How entries are created, updated, reviewed, retired. Who curates
what. How quality is maintained at community scale.

**Status:** ❌ Not designed. Some foundation exists (staleness_threshold, harvest REVIEW,
PR workflow) but community governance at scale is not designed.

### What exists

- `staleness_threshold` field on each entry — staleness-aware retrieval
- `harvest REVIEW` — systematic staleness sweep (manual, per session)
- PR-based submission — CI validates score, format, L1 dedup
- `validate_garden.py --freshness` — count overdue entries

### What's missing

**Governance at community scale:**
- Trusted contributor model (auto-approve for high-reputation contributors)
- Disputed knowledge handling (two entries contradict each other)
- Dimensional editorial board (who owns the assessment dimension? the patterns dimension?)
- Contribution incentives (why would community contributors bother?)
- Versioning (when a technology changes, how do many related entries get updated together?)

**Staleness at scale:**
- Research confirms: outdated knowledge is treated as a security vulnerability in
  high-reliability systems. Current design (manual staleness review) doesn't scale
  to community volume.
- Automated staleness detection — can you detect when an entry is stale from
  external signals? (new library version released, CVE published, deprecation notice)
- "Watch" integration (Phase 9: `_watch/` CI) — monitors upstream for changes
  that should trigger entry review

**Entry retirement:**
- Entries are currently never deleted — deprecated, not removed
- At 1M entries, the retired/deprecated corpus becomes significant
- How does retrieval handle deprecated entries? Suppress? Warn? Different weight?

### Key questions to resolve

1. What is the governance model for community canonical gardens?
2. Can staleness detection be partially automated (version release webhooks, CVE feeds)?
3. How are contradicting entries resolved?
4. What does "trusted contributor" look like — reputation system? Domain vouching?

---

## Area 5: Consumption and Integration Layer

**What it is:** How AI assistants discover and use the garden. The interface between
the retrieval service and the LLM doing the coding.

**Status:** ❌ Not designed. This is identified as a gap in the academic literature too.

### What exists

- MCP server (garden_search tool) — AI assistants can query the garden via MCP
- forage SEARCH skill — manual search during Claude Code sessions
- The retrieval modes (recall / precision) — designed for the right consumer

### What's missing

**Garden discovery:**
- How does an AI assistant know which gardens exist and which to query?
- `garden-config.toml` tells the service where gardens are, but the AI doesn't
  know what the garden contains without querying it
- A "garden manifest" — what domains, what entry types, what coverage — so the
  AI can decide whether to query before spending retrieval latency

**Proactive vs reactive retrieval:**
- Current model: AI asks garden when it has a question (reactive)
- Could be: garden pushes relevant knowledge when context suggests relevance (proactive)
  — e.g., when the AI sees `@PreUpdate` in context, proactively surface the Hibernate
  lifecycle entry without being asked
- Research is open on which is better; proactive risks noise, reactive misses

**Context window management:**
- Research confirms: 5 examples is about optimal, more causes degradation
- How are retrieved entries packaged for the LLM? Just raw markdown? Structured summary?
- When you retrieve from multiple gardens (upstream chain + local), how do you
  budget context across them?

**The "which garden" routing problem:**
- "I want to code a Builder pattern" → patterns-garden
- "My Hibernate callback isn't firing" → discovery-garden
- "How do I migrate from Quarkus 2 to 3?" → transformation-garden + evolution-garden
- Who decides? The AI? A router? The user? The garden-config?

### Key questions to resolve

1. How does an AI assistant know which gardens to query for a given coding task?
2. What does the garden manifest look like (so the AI can self-route)?
3. Proactive or reactive retrieval — or both with different triggers?
4. How are retrieved entries from multiple gardens merged and budgeted for context?

---

## Area 6: Measurement and Evaluation

**What it is:** How you know the garden is actually helping. Entry-level and garden-level
metrics. Feedback loops.

**Status:** ❌ Not designed. Phase 9 design has retrieval_count and helpfulness_score
as concepts, but no pipeline.

### What's missing

**Entry-level metrics:**
- `retrieval_count` — how many times this entry has been retrieved
- `helpfulness_score` — did the AI (or human) act on this entry?
- `resolution_rate` — for gotcha entries, how often does the user report the fix worked?

**Garden-level metrics:**
- Coverage gaps — what queries are consistently returning zero results?
- Freshness distribution — what % of entries are past their staleness_threshold?
- Retrieval quality — are high-scoring entries being retrieved? Low-scoring ones too?
- Entry age distribution — is the garden growing stale overall?

**Feedback loops:**
- When an LLM uses a retrieved entry, was it helpful? How to close this loop?
- MCP server could log query + result → did the conversation continue productively?
- Session capture could ask: "did the garden help with this?" at session end

**A/B testing:**
- With garden vs without garden — does code quality improve?
- Known to be hard to measure empirically (code quality is subjective)
- Proxy: do developers who use the garden make fewer regressions? Resolve more
  tickets first-time?

### Key questions to resolve

1. What are the minimum viable metrics for a v1 garden?
2. How does the MCP server close the feedback loop without asking the user explicitly?
3. Can retrieval quality be measured automatically (did the LLM use the retrieved entry
   in its response)?

---

## Area 7: Privacy, Security, and Deployment

**What it is:** Who can access what. Air-gapped deployments. Organisational privacy.
PII handling in automated pipelines.

**Status:** ⚠️ Partially touched. Per-garden isolation is designed. PII handling in
support ticket pipeline is identified. Full security model not designed.

### What exists

- Per-garden Qdrant instances — isolation by garden
- Child garden model — private knowledge stays in child, never pushed to canonical
- PII handling identified as required step in support ticket pipeline

### What's missing

**Retrieval API access control:**
- Who can query which gardens?
- Public canonical gardens: open to all
- Private child gardens: authenticated only
- How does the MCP server authenticate when querying an upstream garden?

**Air-gapped deployment:**
- No external services (Qdrant self-hosted ✅, Ollama local ✅, SPLADE ONNX local ✅)
- Support ticket ingestion in air-gapped: LLM must be local (Ollama + codellama/mistral)
- Code miner in air-gapped: Claude API not available, local LLM reduces quality

**PII pipeline (support tickets, mining):**
- Required: strip before any LLM call
- Pattern-matching: names, emails, account IDs, internal URLs, company names
- LLM-assisted: subtler PII (project codenames, internal references)
- Audit: what was stripped, for GDPR compliance

**Organisational knowledge leakage:**
- Child garden operators: can they accidentally submit entries that expose
  internal architecture to the canonical (public) upstream?
- The PR workflow provides a gate, but humans can still accidentally include
  sensitive context in otherwise valid entries

### Key questions to resolve

1. What is the authentication model for garden retrieval APIs?
2. How does air-gapped deployment affect support ticket and code miner quality?
3. What is the PII stripping pipeline — pattern-matching only, or LLM-assisted?

---

## Sequencing and Dependencies

```
Area 2 (Knowledge model / taxonomy)
  ├── must be agreed before Area 3 can be finalised (code examples structure)
  ├── must be agreed before Area 5 can be designed (routing requires knowing dimensions)
  └── informs Area 4 (different governance per dimension)

Area 1 (RAG backend)
  ├── 3 critical validations before building
  └── must be built before Area 5 can be designed (routing uses retrieval service)

Area 4 (Lifecycle / governance)
  └── can be designed independently, implemented after Area 1 is built

Area 6 (Measurement)
  └── requires Area 1 to be running (need real queries to measure)

Area 7 (Privacy / security)
  └── can be designed independently, must be implemented before production
```

### Recommended conversation sequence

1. **Finalise Area 2 taxonomy** — agreement on knowledge-type-first, dimension list,
   editorial bars. This unblocks everything else.

2. **Validate Area 1** — the three critical technical validations (SPLADE native image,
   dual-client, IDF modifier). Run a prototype. Unblocks building.

3. **Design Area 5** — consumption layer and routing. Arguably highest user-facing
   value. Requires Area 2 taxonomy and Area 1 validation.

4. **Design Area 4** — governance model. Can happen in parallel with Area 1 validation.

5. **Design Area 3** — modality structures (now that Area 2 is agreed). Code examples
   garden, migration capture, ticket ingestion pipelines.

6. **Design Area 6** — measurement. Needs Area 1 running.

7. **Design Area 7** — privacy/security. Must be complete before any production launch.

---

## What Else Might Be on This List

*Things that may be missing — to be confirmed or discarded.*

- **Multi-modal knowledge** — diagrams, screenshots, stack traces as structured data,
  video recordings of debugging sessions. Not yet discussed.

- **Cross-garden contradiction detection** — when two gardens have entries that
  contradict each other (one says "always use X", another says "never use X").
  Phase 9 concept, not designed.

- **Knowledge as training data** — long-term: garden entries as fine-tuning data for
  smaller specialist models. Karpathy pattern applied to the garden. Not yet discussed.

- **The cold-start / bootstrapping problem** — how do you seed a new domain garden
  quickly? Code miner helps but has limitations. Is there a faster path?
  Not yet designed.
