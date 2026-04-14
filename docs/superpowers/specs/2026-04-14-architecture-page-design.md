# Architecture Page Design

## Goal

Add a dedicated "Architecture" page to hortora.github.io targeting platform architects and DevOps engineers evaluating Hortora for enterprise deployment. The page explains how Hortora's core systems work — not as a tutorial, but as an engineering evaluation resource.

## Audience

Platform/DevOps architects who need to understand how Hortora handles the hard problems (staleness, integrity, concurrency, retrieval accuracy, scale) before committing to enterprise deployment. They want specifics and guarantees, not a sales pitch.

## Page location

- **File:** `hortora.github.io/docs/architecture.html`
- **Nav label:** "Architecture" — added to the sidebar nav in all four existing docs pages
- **Permalink:** `/docs/architecture.html`
- **Jekyll title:** "Architecture — Hortora"

## Structure

Each of the five sections follows a consistent pattern:
1. Heading as a property claim (not a topic label)
2. Problem statement (1–2 sentences)
3. Solution — how the system solves it
4. **Guarantees** callout: concrete promises
5. **Graceful degradation**: what works at reduced capacity vs. what requires intervention

---

## Section 1: Knowledge Decay

**Heading:** "Knowledge never goes stale silently"

**Problem:** An entry from two years ago looks identical to one written yesterday. AI agents can confidently advise a fix that was correct for Quarkus 3.10 but breaks in 3.40, with no indication anything is amiss.

**Solution — 5-layer enforcement:**

- **Always-on annotation (S2):** Every SEARCH result shows the entry's age. Past-threshold entries get a ⚠️ stale warning with verification date and version (if captured). Approaching-threshold entries get an ℹ️ advisory. This fires even if no maintenance has ever been run.
- **Version capture at write time (S6):** Library/tool entries prompt for `verified_on` (e.g. `quarkus: 3.34.2`) at CAPTURE time. SEARCH uses this to produce a concrete version-gap flag rather than a generic age warning.
- **Rationale capture for high-scoring entries (S3):** Entries scoring ≥12/15 prompt for "why this fix over the obvious alternative" — preserving reasoning that would otherwise be lost with the session.
- **Domain-filtered session review (S1):** At SWEEP end, forage checks entries in the domains worked in that session against their `staleness_threshold`. Expired entries surface for Confirm/Revise/Retire while the developer still has fresh context.
- **Systematic backstop (S5):** `validate_garden.py --freshness` scans all entries and reports the overdue count. `harvest REVIEW` processes all overdue entries across all domains — the guarantee that no entry escapes review indefinitely.

**Guarantees:**
- Every search result shows age — staleness is never invisible
- Entries past `staleness_threshold` are flagged before influencing behaviour
- `last_reviewed` resets the staleness clock when a human explicitly confirms validity
- `harvest REVIEW` provides full-garden coverage across all domains

**Graceful degradation:**
- SWEEP domain filtering requires session activity to identify domains — cold-start sessions skip the spot-check, but S2 annotation still fires
- `harvest REVIEW` requires deliberate scheduling; it is not triggered automatically by CI

---

## Section 2: Submission Integrity

**Heading:** "No entry enters the garden without passing validation"

**Problem:** AI-generated content can be plausible-sounding but wrong, duplicate, or malformed. An unguarded knowledge store accumulates noise that erodes trust faster than it accumulates value.

**Solution — Multi-gate validation pipeline (`validate_pr.py`):**

- **Format validation:** YAML frontmatter must contain all required fields (`id`, `title`, `type`, `domain`, `stack`, `tags`, `score`, `verified`, `staleness_threshold`, `submitted`). Malformed entries fail immediately.
- **Score threshold:** Entries are rated across 5 dimensions — non-obviousness, discoverability, breadth, pain/impact, longevity — and must reach ≥8/15. The dimensions are defined in the protocol spec and enforced by CI.
- **Injection detection:** Entry content is scanned for prompt-injection patterns before it enters the garden. An entry that tries to modify Claude's behaviour at retrieval time is rejected.
- **L1 deduplication:** Jaccard similarity is checked against existing entries in the same domain at submission time. Obvious duplicates are rejected before a human reviewer sees them.
- **CI enforcement:** GitHub Actions runs `validate_pr.py` on every PR against the garden repo. Merge is blocked on any failure. No bypass path exists in the standard workflow.

**Guarantees:**
- No entry merges without passing format validation and the ≥8 score threshold
- CI blocks merge on failure — there is no silent pass
- Every entry carries a permanent, collision-resistant GE-ID (`GE-YYYYMMDD-xxxxxx`) assigned at write time with no central counter
- All entry history is preserved in git — nothing is silently overwritten

**Graceful degradation:**
- L1 Jaccard catches obvious duplicates; close variants can still enter — `harvest DEDUPE` provides L2/L3 semantic deduplication as a periodic maintenance sweep
- Score is self-reported by the submitter; CI enforces the valid range (1–15) but not accuracy — periodic re-scoring improves signal over time

---

## Section 3: Storage Reliability

**Heading:** "Concurrent sessions cannot corrupt garden state"

**Problem:** Multiple AI sessions writing to a shared knowledge store simultaneously can produce partial writes, conflicting commits, and filesystem races — inconsistencies that are hard to detect and harder to recover from.

**Solution — Git as the consistency layer:**

- **Read discipline:** All reads use `git show HEAD:<path>` — never the filesystem directly. A session mid-write does not affect what another session reads.
- **Write discipline:** Every file write is followed immediately by `git add` and `git commit`. No uncommitted garden state persists between skill operations.
- **Conflict recovery:** When two sessions commit simultaneously, the second receives a rejected push. `git rebase HEAD` resolves non-conflicting concurrent writes automatically, without data loss.
- **Human-readable format:** YAML frontmatter + markdown — diffs are readable, history is auditable with `git log` and `git blame`, no binary formats.
- **Sparse blobless clone:** Large gardens use `--filter=blob:none --sparse` at clone time. Clone size stays bounded as the garden grows to thousands of entries.

**Guarantees:**
- Concurrent sessions cannot produce partially-visible entries — the commit is the write
- Every read sees a complete, committed state regardless of concurrent write activity
- Full audit trail: git history records who submitted what, when, from which session
- Entries are never deleted — only deprecated or retired, with content preserved

**Graceful degradation:**
- Rebase recovery handles non-conflicting concurrent writes automatically; two sessions editing the same entry simultaneously require manual conflict resolution (rare — entries are append-only by convention)
- Sparse blobless clone requires a git remote with partial clone support (GitHub, GitLab); local-only gardens fetch all blobs at clone time

---

## Section 4: Retrieval Accuracy

**Heading:** "Relevant entries surface; irrelevant ones don't"

**Problem:** A flat keyword search across hundreds of entries produces two failure modes: false positives (cross-domain noise) and false negatives (missed entries because symptom keywords don't align). Both degrade the garden's value faster than new entries can recover it.

**Solution — 3-tier retrieval with bounded context cost:**

- **Tier 1 — Technology filter:** Every entry is assigned a `domain` at write time. Retrieval scopes to the relevant domain first — cross-domain entries are never surfaced for an unrelated query.
- **Tier 2 — Symptom and label match:** Within the domain, the index is searched by symptom type and label across three dimensions — By Technology, By Symptom/Type, By Label.
- **Tier 3 — Full domain scan:** If tiers 1 and 2 produce no match, a full scan of the domain is performed. No entry in the relevant domain is unreachable regardless of keyword alignment.
- **Index pre-load, bodies on demand:** The `GARDEN.md` index is loaded at session start (ADR-0001 lazy-reference pattern). Entry bodies are only fetched when an entry is selected — context budget cost is proportional to entries read, not garden size.
- **Git-only reads:** Index and entry bodies are always read from `git show HEAD:` — no partial-write races, no stale filesystem cache.

**Guarantees:**
- Technology scoping eliminates cross-domain false positives at query time
- Three tiers ensure no entry in the relevant domain is unreachable
- Context budget cost is bounded — the index pre-load is a single file regardless of garden size
- Index is always committed state — no read can observe a partial write

**Graceful degradation:**
- Tier 3 (full domain scan) degrades in precision as a domain grows beyond ~200 entries — RAPTOR cluster summaries (Phase 8) will address this at scale
- Keyword effectiveness is not automatically measured — entries with thin keywords may be missed until a maintainer updates them; `harvest REVIEW` is the natural point to catch this

---

## Section 5: Federation & Governance

**Heading:** "The protocol scales beyond a single garden"

**Problem:** A single canonical garden becomes a bottleneck at enterprise scale. Organisation-specific knowledge cannot sit alongside public community knowledge. Domain gardens need independent curation cadences. Yet independent forks fragment the protocol.

**Solution — 3-tier federation model:**

- **Canonical garden:** Curated, CI-enforced, publicly reviewed. Source of truth for cross-domain entries. Maintained under the Hortora GitHub organisation.
- **Child garden:** Inherits from a canonical garden and extends it with domain-specific or organisation-specific entries. Validation prevents duplication of canonical entries — a child garden adds without repeating.
- **Peer gardens:** Independent gardens sharing the Hortora protocol. No inheritance — entries are entirely distinct — but the entry format, GE-ID scheme, and retrieval algorithm are compatible.
- **SchemaVer:** Each entry declares its schema version. Mismatches between gardens are detectable at integration time rather than silent.
- **Open protocol:** Entry format, validation pipeline, and retrieval algorithm are not Claude-specific. Any AI assistant implementing the forage/harvest skill convention can participate. The spec is published and versioned independently of any model or vendor.

**Guarantees (current):**
- Entry format and GE-ID scheme are stable and versioned — entries written today will be valid in future schema versions or carry an explicit migration path
- The protocol is vendor-neutral — no lock-in to a specific AI provider or model family
- Enterprise air-gapped deployment is supported today via local-only mode — no GitHub remote required

**Guarantees (Phase 5+, planned):**
- Child garden validation prevents duplication of canonical entries
- Schema version mismatches fail loudly at integration time, not silently at retrieval time
- Compliant gardens can participate in federated retrieval — forage SEARCH can query across gardens in a single session

**Graceful degradation:**
- Federation is Phase 5+ — current production implementation is single-garden. The spec is published; multi-garden tooling is not yet deployed.

---

## Section 6: Deduplication

**Heading:** "Duplicate knowledge is eliminated in layers, not all at once"

**Problem:** A single dedup gate at submission time is either too strict (rejecting valid near-duplicates) or too permissive (letting close variants through). A knowledge garden under active contribution from multiple sessions accumulates semantic drift that point-in-time checks cannot catch.

**Solution — Three-level deduplication with a drift counter:**

- **L1 — Jaccard at PR time (`validate_pr.py`):** Runs automatically on every submission. Compares the incoming entry against all existing entries in the same domain using Jaccard similarity. Catches obvious duplicates and rejects them before any human review. Fires in CI — no operator action required.
- **L2 — Related entry detection (`harvest DEDUPE`):** Periodic maintenance operation that compares all within-domain entry pairs not yet checked against each other. Related entries (similar but legitimately distinct) are identified and cross-referenced — "See also: GE-XXXX" is added to both entries. This improves retrieval coherence without discarding valid knowledge.
- **L3 — Duplicate consolidation (`harvest DEDUPE`):** Within the same sweep, true duplicates (one entry is a subset or copy of another) are surfaced to the maintainer for consolidation. The less complete entry is retired; the more complete entry is preserved. Discarded entries are recorded in `DISCARDED.md` for audit.
- **Drift counter:** `GARDEN.md` tracks `Entries merged since last sweep`. When it reaches the configured threshold (default: 10), `harvest DEDUPE` should be run. The counter is visible, persistent, and incremented by the CI integration step on every merge — it is the discipline mechanism that prevents dedup debt from accumulating silently.

**Guarantees:**
- L1 catches obvious duplicates before merge — no obviously redundant entry enters the garden through the standard workflow
- Every dedup comparison is logged in `CHECKED.md` — pairs are never compared twice, and the full comparison history is auditable
- Discarded entries are recorded in `DISCARDED.md`, not silently deleted — the decision to discard is always traceable
- The drift counter makes dedup debt visible — maintainers always know how many entries have been added since the last sweep

**Graceful degradation:**
- L2/L3 (`harvest DEDUPE`) require a dedicated session with full context budget — they are not run automatically by CI
- The drift counter triggers an obligation, not an enforcement gate — a garden can continue operating above threshold, but the counter makes the debt visible
- Cross-domain duplicates are not checked — entries in different domains are assumed to be distinct by definition

---

## Implementation notes

- Match existing HTML structure: Jekyll frontmatter, `docs-layout` with `docs-sidebar` and `docs-content`
- Add "Architecture" link to sidebar nav in all five docs pages (getting-started, how-it-works, skills-reference, design-spec, and architecture itself)
- Use existing CSS classes — `callout`, `code-block`, `lead` — no new styles required
- Guarantees and Graceful degradation rendered as styled callout boxes matching the existing site design
- Page does not require any new Jekyll collections or config changes
