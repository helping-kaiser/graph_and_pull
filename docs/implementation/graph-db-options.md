# ADR-001: Graph Database Selection

**Status**: Accepted
**Date**: 2026-04-10

---

## Context

This project requires a database to store and traverse the social graph: a node catalog (User, Collective, Post, ChatMessage, Item, Hashtag, junction nodes, …) connected by uniform tensor edges (2 user dimensions + system dimensions, with per-category labels — see [edges.md](../primitive/edges.md)). The choice of graph database affects the query language, Rust driver ecosystem, operational complexity, and learning value of the project.

The primary goals are:
1. **Expressiveness** — graph traversal queries should be readable and maintainable
2. **Rust compatibility** — there should be a working Rust client
3. **Local dev simplicity** — should run easily in Docker without heavy dependencies
4. **Learning value** — the project is exploratory; the DB should expose graph concepts clearly

---

## Decision

**Memgraph** — using openCypher as the query language, connected from Rust via the bolt protocol using the `neo4rs` crate.

---

## Alternatives Considered

### Option A: PostgreSQL only (recursive CTEs)

**Pros:**
- No additional infrastructure
- Team familiar with SQL
- Works well for shallow traversals (3-4 hops)

**Cons:**
- Recursive CTEs become hard to read for anything beyond simple `friends of friends`
- Performance degrades at scale — Postgres does index lookups at each hop, not index-free adjacency
- Graph algorithms (shortest path, weighted recommendations) require complex SQL that fights the data model
- Defeats the learning goal of exploring dedicated graph databases

**Verdict:** Rejected. Valid for production at small scale, but not aligned with the exploratory goals of this repo.

---

### Option B: SurrealDB

**Pros:**
- Written in Rust — native SDK, excellent DX for Rust projects
- Multi-model: can do graph, document, and relational in one DB
- Growing community

**Cons:**
- SurrealQL is SQL-flavored with graph extensions (`->` notation) — readable but not as expressive as Cypher for complex graph patterns
- Less mature than Neo4j-family databases for graph-specific workloads
- Fewer graph algorithm primitives out of the box

**Verdict:** Rejected. The SQL-flavored query model is a significant step down from Cypher for graph readability. The Rust-native argument is compelling but not decisive.

---

### Option C: Neo4j

**Pros:**
- Industry standard — invented Cypher, largest ecosystem
- Best learning resources (most Cypher tutorials target Neo4j)
- Neo4j Browser is an excellent visual tool
- Neo4j Aura offers a free cloud tier

**Cons:**
- Requires JVM — Docker image is 600MB+
- The official Java driver; Rust driver (`neo4rs`) is community-maintained
- Slower than Memgraph for in-memory graph workloads
- Enterprise features are paywalled

**Verdict:** Rejected in favor of Memgraph. Neo4j's main advantage is its learning ecosystem — but since Memgraph is fully bolt/Cypher compatible, all Neo4j Cypher resources apply 1:1.

---

### Option D: ArangoDB

**Pros:**
- Multi-model (graph + document + key-value)
- Mature, used in production

**Cons:**
- AQL (ArangoDB Query Language) is a third query language to learn — not Cypher, not SQL
- No official Rust client; HTTP-based access only
- The multi-model approach means graph features are not the primary focus

**Verdict:** Rejected. AQL adds learning overhead without the payoff of Cypher's graph expressiveness.

---

## Why Memgraph over Neo4j

| Criterion | Memgraph | Neo4j |
|---|---|---|
| Query language | openCypher | Cypher (superset of openCypher) |
| Rust driver | bolt → `neo4rs` | bolt → `neo4rs` (same) |
| Docker image | ~200MB | ~600MB+ (JVM) |
| Performance | Faster (in-memory first) | Slower at equivalent scale |
| Graph algorithms | MAGE library | GDS plugin |
| Visualization | Memgraph Lab (good) | Neo4j Browser (excellent) |
| Learning resources | Good, Cypher-compatible | Best in class |

The driver situation is identical — `neo4rs` works with Memgraph because both speak bolt. All Neo4j Cypher documentation applies to Memgraph. The lighter Docker footprint and faster traversal performance make Memgraph the better fit for local development.

---

## Consequences

- All graph queries are written in Cypher
- The `neo4rs` crate is used for the bolt connection
- Memgraph Lab (http://localhost:3000) is available locally for visual graph exploration
- If we ever need to migrate to Neo4j, the Cypher queries are compatible with minimal changes
