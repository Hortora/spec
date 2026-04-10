# 0003 — GE-ID scheme: date + random hex, replacing sequential counter

Date: 2026-04-10
Status: Accepted

## Context and Problem Statement

Garden entries need stable, unique identifiers (GE-IDs). The original scheme
used a sequential counter (GE-0001, GE-0002, …) stored in GARDEN.md. This
requires coordination: every submission must read the counter, increment it,
and write it back — creating race conditions under concurrent submissions and
coupling the ID assignment mechanism to a centralised bottleneck.

## Decision Drivers

* No coordination required between concurrent submitters
* IDs must remain human-referenceable (speakable, embeddable in skill prompts)
* Must work across local mode and GitHub PR mode without an external service
* Federation-safe: gardens owned by different teams must not collide
* Git is fine for small-team gardens; future large-scale models will differ

## Considered Options

* **Sequential counter in GARDEN.md** — current scheme; GE-0001, GE-0002, …
* **GitHub Issue number as ID** — create an issue per submission, issue # = GE-ID
* **UUID** — full 128-bit random ID
* **Date + 6 random hex chars** — e.g. GE-20260410-a3f7c2

## Decision Outcome

Chosen option: **Date + 6 random hex chars**, because it requires no
coordination, is human-referenceable, has negligible collision probability at
small-team garden scale, and is federation-safe by construction.

Format: `GE-YYYYMMDD-xxxxxx` where `xxxxxx` is 6 lowercase hex characters
generated at submission time.

Existing entries (GE-0001 through GE-0172) retain their sequential IDs.
The validator accepts both formats. No migration of existing entries.

### Positive Consequences

* No counter to synchronise — IDs are generated locally without coordination
* Collision probability < 0.003% at 1,000 submissions/day per garden
* Date component provides human-readable provenance
* Works identically in local mode and GitHub PR mode
* Federation-safe: probability of cross-garden collision on the same day
  is negligible (16.7M possibilities per day)

### Negative Consequences / Tradeoffs

* IDs are not sequential — no implied ordering between entries
* Two ID formats coexist in the garden during transition (old: GE-NNNN,
  new: GE-YYYYMMDD-xxxxxx)
* At truly global scale (thousands of submissions/day to a single garden),
  a different model will be needed — accepted, as that scenario implies
  federation rather than a single shared garden

## Pros and Cons of the Options

### Sequential counter in GARDEN.md

* ✅ Simple, human-readable ordering
* ❌ Requires coordination — race conditions under concurrent submissions
* ❌ Centralised bottleneck as submission rate grows

### GitHub Issue number as ID

* ✅ Issue number is externally visible and trackable
* ❌ GitHub Issues don't scale to millions of entries
* ❌ Creates a 1:1 coupling between workflow tooling and ID scheme
* ❌ Existing gap (garden at GE-0172, issues start at #1) requires workaround

### UUID

* ✅ Zero collision risk
* ✅ No coordination needed
* ❌ Not human-readable or speakable
* ❌ Ugly in file names, skill prompts, and conversation references

### Date + 6 random hex chars

* ✅ No coordination needed
* ✅ Human-referenceable and speakable
* ✅ Date component provides context
* ✅ 50% collision threshold ~5,000 submissions/day — far beyond single garden scale
* ❌ Not strictly ordered (entries on same day are unordered relative to each other)
* ❌ Two ID formats during transition

## Links

* Supersedes the sequential counter approach established at garden inception
* [ADR-0002](0002-ci-script-location-and-soredium-visibility.md) — CI script location (related: same git-based coordination model)
