# TruthDB Use Cases

Status: draft
Last updated: 2026-01-23

This document collects candidate use cases for TruthDB.

It is intentionally broad; it should be refined into a small set of flagship scenarios as the product scope hardens.

## AI, vector search, and retrieval-augmented generation

### RAG knowledge base

- TruthDB stores document chunks with their embedding vectors alongside structured metadata.
- Similarity search retrieves the most relevant context for LLM prompt assembly.
- No external vector database required — retrieval, filtering, and full-text search happen in one system.

### Semantic search over business data

- Dense vector embeddings enable "meaning-based" search beyond keyword matching.
- Hybrid queries combine vector similarity with exact filters (e.g., "find similar support tickets from the last 30 days for tenant X").
- Useful for product catalogs, knowledge bases, internal documentation, and customer support.

### Embedding versioning and re-indexing

- When embedding models change, TruthDB can re-embed from the authoritative WAL without data loss.
- Full provenance: every vector traces back to the source document event and model version.
- Rolling cutover between old and new embedding indexes (same pattern as search reindexing).

### AI agent memory and context store

- AI agents can persist conversation history, tool outputs, and retrieved context as events.
- Vector search over agent memory enables retrieval of relevant past interactions.
- Event-sourced history provides full audit trail of what the agent "knew" at each step.

## Core database and event-sourcing

### Application event store

- TruthDB acts as the backbone of an event-sourced system.
- Each aggregate is an ordered stream of events.
- Enables replay/time-travel and snapshotting for rebuilds.

### Append-only ledger / audit log

- Immutable, checksummed events.
- Strong auditability for regulated workflows.

### Time-series append store

- Continuous ingestion of telemetry.
- Append-only history (no rewrite/compaction assumptions in the core log).

## Systems and infrastructure

### Durable replication bus / changefeed

- Use partitions as replication logs between systems.
- Consumers can tail and replay deterministically.

### Local-first / edge durability

- Run on edge devices.
- Sync by shipping events later (delta/event-based replication).

## Domain-specific systems

### Financial core / payments ledger (platform-style)

- Transactions represented as atomic events.
- Intended to support strong durability and replay, while keeping domain logic above the WAL.

### IoT / industrial telemetry hub

- Gateways append raw readings.
- Consumers subscribe to event feeds.

## Development and research

### Deterministic recovery testing harness

- Crash recovery, replay ordering, checksum validation.
- Repeatable “power loss” simulations at the log boundary.

## Auditing and provenance

### Tamper-evident journal

- Strong corruption detection.
- Sealed segments + verification (future design).

## Next steps

- Pick 2–3 flagship scenarios.
- Define what is strictly guaranteed (ordering, durability modes, determinism) vs implementation detail.
- Evaluate vector/RAG retrieval as a flagship differentiator scenario.
