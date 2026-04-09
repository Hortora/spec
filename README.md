# Hortora / spec

Open protocol specification for **Hortora** — a governed, federated knowledge garden for AI coding assistants.

Hortora solves a specific problem: developers rediscover the same non-obvious bugs, workarounds, and undocumented behaviours repeatedly across projects. Hortora captures them once, organises them with a quality lifecycle, and federates them across machines and teams via GitHub-backed canonical gardens.

---

## What's in this repo

### Design

[`docs/design/2026-04-07-garden-rag-redesign-design.md`](docs/design/2026-04-07-garden-rag-redesign-design.md)

The full protocol specification. Covers:

- **Entry format** — YAML frontmatter + markdown body, one file per entry
- **Retrieval algorithm** — three-tier (by technology → by symptom → full scan)
- **Deduplication** — three-level Jaccard similarity (L1/L2/L3)
- **Quality lifecycle** — Active → Suspected → Superseded → Retired
- **GitHub backend** — PR-based submission, CI validation, GitHub Issues as entry IDs
- **Federation protocol** — canonical / child / peer garden relationships
- **Implementation roadmap** — nine phases from data migration to hosted federation

### Architecture Decision Records

[`docs/adr/`](docs/adr/)

- [ADR-0001](docs/adr/0001-index-and-lazy-reference-pattern.md) — Index-and-lazy-reference pattern for session continuity

### Blog

[`docs/blog/`](docs/blog/)

The founding narrative. Also published at [hortora.github.io/blog](https://hortora.github.io/blog).

---

## Related repositories

| Repo | Purpose |
|------|---------|
| [Hortora/soredium](https://github.com/Hortora/soredium) | Claude skills, validators, and tooling |
| [Hortora/hortora.github.io](https://github.com/Hortora/hortora.github.io) | Public website |

---

## Status

Specification: **draft** — phased implementation underway via [soredium](https://github.com/Hortora/soredium).
