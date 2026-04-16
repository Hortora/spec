# Code Examples Garden — Design Spec

**Date:** 2026-04-16
**Status:** Proposed
**Companion to:** `2026-04-16-garden-retrieval-service.md`

---

## Problem

A developer says "I want to support multiple databases and switch between them for my
JPA persistence layer." They need a working, minimal, copyable code example — not an
explanation of why something is non-obvious. The knowledge garden serves the explanation
use case. The code examples garden serves the pattern use case.

Without a code miner, the garden fills slowly by hand. With one, point it at any
Quarkus project and it extracts, minimises, and submits examples automatically.

---

## Two Components

1. **Code examples garden** — a separate knowledge garden of minimal, use-case-driven
   code snippets. Same infrastructure as the knowledge garden (git + Qdrant +
   garden-retrieval service) with different entry format and retrieval semantics.

2. **Code miner** — a tool that points at a codebase, discovers pattern instances, uses
   an LLM to rewrite them as minimal teachable examples, and submits them to the garden.

---

## Code Examples Garden

### Entry Format

```yaml
---
id: CE-20260416-aabbcc
title: "Multiple JPA datasources with runtime switching"
use_case: "Support multiple databases and switch between them for JPA persistence, configurable per tenant or context"
type: example
domain: quarkus
stack: "Quarkus 3.x, Panache, PostgreSQL, MySQL"
tags: [jpa, datasource, multi-tenancy, configuration]
language: java
difficulty: intermediate
submitted: 2026-04-16
mined_from: "github.com/example/project (anonymised)"   # optional provenance
---

## Multiple JPA datasources with runtime switching

```java
// application.properties
quarkus.datasource.db-kind=postgresql
quarkus.datasource."users".db-kind=mysql
quarkus.datasource."users".jdbc.url=${USERS_DB_URL}

// Repository using named datasource
@ApplicationScoped
public class UserRepository {

    @Inject
    @DataSource("users")
    AgroalDataSource usersDataSource;

    public List<User> findAll() {
        try (var conn = usersDataSource.getConnection()) {
            // query
        }
    }
}
```

**Dependencies:**
```xml
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-hibernate-orm-panache</artifactId>
</dependency>
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-jdbc-mysql</artifactId>
</dependency>
```

**Key points:**
- Each named datasource gets its own `quarkus.datasource."name".*` namespace
- Inject by name using `@DataSource("name")` qualifier
- Default datasource (no name) still available alongside named ones
```

### How it differs from the knowledge garden

| | Knowledge garden | Code examples garden |
|---|---|---|
| Entry ID | `GE-YYYYMMDD-xxxxxx` | `CE-YYYYMMDD-xxxxxx` |
| Purpose | Explain non-obvious behaviour | Provide something to copy |
| Content | Prose + optional code | Minimal code + brief notes |
| Key field | `title` (the weird thing) | `use_case` (what I want to do) |
| Editorial bar | Non-obvious, hard to discover | Any valid working minimal example |
| Multiple variants | One entry per gotcha | One entry per variant (PG version, MySQL version) |
| Entry length | 50–200 lines | 20–50 lines |
| Retrieval goal | Find the right explanation | Find 3–5 adaptable variants |

### Retrieval semantics

The `use_case` field is the primary retrieval surface. Queries are natural language intent
("I want to switch between multiple databases") matched against use case descriptions.
SPLADE term expansion handles: "databases" → "PostgreSQL", "MySQL", "MongoDB", "datasource",
"multi-tenancy".

Return 4–6 variants in recall mode (multiple database types, multiple pattern approaches).
The LLM selects and adapts the most relevant one for the user's context.

Dense embedding: `nomic-embed-text` (NL queries against NL use-case descriptions —
general model is appropriate; code-specific model helps if queries contain code fragments).
Consider benchmarking `jinaai/jina-embeddings-v2-base-code` (Apache 2.0) for comparison.

### Qdrant collection

Separate collection `code_examples` in the same Qdrant instance, or a separate Qdrant
instance. Same `garden-retrieval` service handles both — different collection name in
`garden-config.toml`:

```toml
[[gardens]]
name = "quarkus-examples"
path = "~/.hortora/quarkus-examples"
retrieval_url = "http://localhost:8091"
```

---

## Code Miner

### Purpose

Point at any codebase. Discover instances of patterns. Extract the relevant code.
Use an LLM to rewrite it as the minimal teachable example. Submit to the code examples
garden.

Without the miner: garden fills by hand, slowly.
With the miner: point at any Quarkus project (internal or open-source) and get dozens
of examples in minutes.

### Pipeline

```
Input: git repo URL or local directory path
       + optional focus: "builder patterns" / "all patterns" / "datasource configs"

Step 1 — DISCOVER (static analysis, no LLM)
  Parse Java AST with JavaParser (Apache 2.0)
  Identify pattern instances:
    - Builder pattern: classes with build() method returning the class type,
                       chain methods returning 'this'
    - Multi-datasource: @DataSource annotations, multiple datasource configs
    - Configuration DSL: @ConfigMapping, @ConfigProperty hierarchies
    - Test patterns: @QuarkusTest, @TestTransaction, QuarkusTransaction usage
    - Reactive: Uni<T>, Multi<T>, chain operators
    - REST client: @RegisterRestClient, @RestClient injection
    - Custom pattern rules (user-extensible)

Step 2 — EXTRACT (static analysis, no LLM)
  For each discovered instance:
    - Extract the minimal class/method + direct dependencies
    - Resolve imports actually used (JavaParser symbol solver)
    - Identify required annotations, interfaces, parent classes
    - Estimate example size (skip if inherently large/complex)
    - Skip if tightly coupled to internal infrastructure (heuristics:
      company package names, internal base classes, private registries)

Step 3 — MINIMISE (LLM)
  Send extracted code to LLM with structured prompt:

  "You are extracting a minimal, teachable code example from production code.

  Pattern detected: {pattern_name}
  Original code:
  {extracted_code}

  Rewrite this as the smallest self-contained example that demonstrates
  the pattern. Requirements:
  - Under 40 lines of code
  - No company-specific names, URLs, or internal dependencies
  - Generic but realistic variable/class names
  - Keep all annotations and framework-specific details (they matter)
  - Add a one-line comment if the key insight isn't obvious from the code
  - Output the use_case description: one sentence starting with 'I want to...'
  - Output the required Maven dependencies (groupId:artifactId only)

  Output format: JSON with fields: code, use_case, title, dependencies, tags"

  LLM: Claude API (Anthropic) or Ollama (local, for air-gapped environments)

Step 4 — SCORE AND FILTER (heuristic)
  Reject if:
    - Code > 50 lines after minimisation
    - use_case description is vague ("I want to use Java")
    - Duplicate use_case already in garden (semantic check via retrieval API)
    - LLM output fails JSON parse or required fields missing

Step 5 — HUMAN REVIEW GATE (optional)
  Interactive CLI: show each candidate, Accept / Edit / Reject / Skip
  --auto flag: skip review, submit all passing candidates
  --dry-run flag: show what would be submitted, write nothing

Step 6 — SUBMIT
  Write CE-YYYYMMDD-xxxxxx.md to garden directory
  Run validate_pr.py (reused from knowledge garden pipeline)
  Git commit + PR (or direct to main for trusted sources)
```

### What makes a good minimal example

The LLM minimisation prompt targets these properties:

| Property | What it means | How enforced |
|----------|---------------|-------------|
| Self-contained | Compilable with only declared dependencies | LLM strips internal imports |
| Generic | No company names, URLs, or secrets | LLM renames, removes |
| Focused | Demonstrates exactly one pattern | Extract one class/method, not a file |
| Short | < 40 lines of code | LLM instruction + post-filter |
| Annotated | Framework annotations preserved | LLM instruction (they matter for Quarkus) |

### Technology

**AST parsing:** JavaParser (open source, Apache 2.0, Java-native). Handles Java
and Kotlin. For multi-language support: tree-sitter (Rust, Python bindings available,
handles 40+ languages including Java, Kotlin, TypeScript, Python).

**LLM integration:**
- **Claude API** (Anthropic): best minimisation quality, requires API key
- **Ollama** (local): `llama3`, `mistral`, `codellama` — air-gapped environments,
  slightly lower quality for code rewriting
- Pluggable: same prompt, different backend

**Duplicate detection before submission:**
Call the code examples garden's retrieval API:
```
GET /api/search?q={use_case}&mode=precision&limit=3
```
If top result similarity > 0.85: skip as likely duplicate.

### Deployment

Standalone CLI tool: `garden-miner` (Quarkus CLI extension or standalone Java JAR).

```bash
# Mine all builder patterns from a project
garden-miner mine ./my-quarkus-project \
  --pattern builder \
  --garden-url http://localhost:8091 \
  --review        # interactive review before submit

# Mine everything, auto-submit
garden-miner mine https://github.com/quarkusio/quarkus-quickstarts \
  --all \
  --auto \
  --dry-run       # show what would be submitted

# Mine specific file or package
garden-miner mine ./src/main/java/com/example/persistence \
  --pattern datasource,configuration
```

### Pattern Library (extensible)

Initial patterns to detect in Java/Quarkus:

| Pattern | AST heuristic |
|---------|--------------|
| Builder/fluent DSL | Class with `build()` → self-type, chain methods return `this` |
| Multi-datasource | `@DataSource` annotation OR multiple `quarkus.datasource.*` keys |
| Config mapping | `@ConfigMapping` interface with nested config groups |
| QuarkusTest setup | `@QuarkusTest` class with `@TestProfile` or custom config |
| REST client | `@RegisterRestClient` + `@RestClient` injection |
| Panache repository | Extends `PanacheRepository<T>` with custom queries |
| Reactive chain | `.onItem()`, `.onFailure()`, `Uni.combine()` chains |
| MCP tool | `@McpTool` annotation with `@McpToolParam` parameters |
| Event bus | `@ConsumeEvent`, `EventBus.publish()` patterns |

Pattern library is user-extensible via config file: define AST patterns as JSON rules.

---

## Integration with Knowledge Garden

The code examples garden and knowledge garden are complementary:

```
Knowledge garden          Code examples garden
─────────────────         ────────────────────
"Why is @PreUpdate        "Show me how to configure
not firing?"              multiple datasources"

Explains a gotcha         Provides a pattern to copy

Prose + optional code     Minimal code + brief notes

Captured by:              Captured by:
  forage CAPTURE            garden-miner (automated)
  (manual, human)           OR forage-like skill
                            (prompted during coding)
```

Both gardens are queried via the same `garden-retrieval` service (different collections).
A coding assistant queries both in parallel: gotcha garden for "why is X broken",
code examples garden for "show me how to do Y."

---

## Open Questions

1. **Pattern detection quality.** AST heuristics will produce false positives (not
   every class with a `build()` method is a builder worth capturing). Human review
   gate mitigates this; tuning pattern rules over time improves it.

2. **Minimisation quality.** LLM rewriting quality varies. Claude API produces better
   minimal examples than smaller local models. Trade-off between quality and
   air-gapped operation.

3. **Provenance and licensing.** Mining from open-source projects: what licence
   restrictions apply to extracted code examples? Apache 2.0 and MIT projects are
   clearly fine. GPL projects need care. `mined_from` frontmatter field records
   provenance; human review gate allows licence check.

4. **Multi-language support.** JavaParser handles Java/Kotlin. tree-sitter handles
   40+ languages. If the garden needs TypeScript, Python, or Go examples, tree-sitter
   is the right path. Not needed for initial implementation.

5. **Incremental mining.** Running the miner on the same repo twice should not
   re-submit duplicates. Duplicate detection via retrieval API handles this, but
   depends on garden being populated from the first run.

6. **Code example staleness.** Unlike gotcha entries (API-specific, can go stale),
   pattern examples change less often. `staleness_threshold` could be longer
   (1460 days / 4 years) for patterns vs 730 days for API-specific examples.
