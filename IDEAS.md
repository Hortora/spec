# Idea Log

Undecided possibilities — things worth remembering but not yet decided.
Promote to an ADR when ready to decide; discard when no longer relevant.

---

## 2026-04-10 — Garden staleness: entries decay without context or review mechanism

**Priority:** high  
**Status:** active

Garden entries capture WHAT (symptom, root cause, fix) but not WHY this fix over alternatives, or WHEN the knowledge expires. The `staleness_threshold` field exists but is never enforced — entries past threshold look identical to current ones. The risk: AI agents act on stale, decontextualized knowledge with full confidence, amplifying bad practices across sessions silently. "Red hat bureaucracy with agentic powers."

**Context:** Feedback from external reviewer. Discussed 2026-04-10 in hortora session. Core tension: a knowledge garden is only as good as its decay management.

**Proposed solutions and likelihood:**

**1. Active staleness enforcement in forage SWEEP** — when sweeping a session, flag entries past `staleness_threshold` for the user to confirm or revise. Implementation: ~1 day. *Likelihood of solving the problem: HIGH — directly closes the loop between stale entries and human review. Main risk: users ignore the flags.*

**2. Skepticism annotation in forage SEARCH results** — when surfacing a garden entry, append "(entry from YYYY-MM-DD, verify still applies to [stack version])" for entries older than N months. Implementation: hours. *Likelihood: MEDIUM-HIGH — adds appropriate epistemic humility at the point of use, cheap to implement, but doesn't remove stale entries.*

**3. Require "why not alternatives" field on high-scoring entries** — entries scoring 12+ require a brief note on why this fix over the obvious alternative. Implementation: validator change + skill update. *Likelihood: MEDIUM — improves quality for new entries but doesn't fix existing 172. Adds friction that may reduce submission rate.*

**4. Provenance links — entry links to session ADR or issue that prompted it** — so Claude can assess context rather than treating the entry as timeless fact. Implementation: convention in submission format. *Likelihood: LOW-MEDIUM — sessions don't persist, links rot; ADR links are more viable but most entries don't have ADRs.*

**5. Periodic review harvest mode** — a dedicated harvest operation that resurfaces all entries past staleness threshold and prompts for confirm/revise/retire. Implementation: ~2 days. *Likelihood: HIGH if actually run regularly; MEDIUM in practice — depends on discipline to run it.*

**6. Version-pinned confidence decay** — entries for versioned libraries get a "confidence: high until vX.Y, low after" marker. forage SEARCH surfaces this. Implementation: schema change + skill update. *Likelihood: MEDIUM — good for library-specific gotchas, useless for methodology/pattern entries.*

**Promoted to:**
