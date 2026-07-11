# How TruthDB Differs From PostgreSQL (Conceptual)

Status: draft
Last updated: 2026-03-27

This is a conceptual comparison intended for orientation.
It is not a performance claim.

## PostgreSQL WAL (what it is)

PostgreSQL uses a WAL primarily as a **physical redo log** for a mutable storage engine.

- The “truth” is the heap/index pages (mutable on disk).
- WAL exists to make torn writes recoverable and to guarantee crash recovery.
- WAL records are tightly coupled to the storage engine layout (page formats, MVCC machinery, index structures).

Implications:

- WAL is not a domain log.
- WAL is not meant to be portable across different storage engines.
- “Replay” is the engine’s redo procedure to restore pages.

## TruthDB WAL (intended direction)

TruthDB’s direction is “log-first”:

- The WAL is intended to be the **authoritative ordered substrate**.
- Derived state (tables/indexes/snapshots) is rebuildable from WAL bytes.
- Correctness is defined in terms of ordering + deterministic replay.

Implications:

- The WAL is treated as a product-level invariant (ordering, durability modes, corruption detection).
- Replay is a core capability, not just crash recovery.

## Summary

- PostgreSQL: mutable engine + WAL as recovery mechanism.
- TruthDB (intended): WAL as source of truth + derived state as optimization.
- PostgreSQL + pgvector: vector search bolted onto a row-oriented engine via an extension.
- TruthDB (intended): vector search as a native retrieval modality alongside full-text and structured queries, with vectors stored in the WAL and ANN indexes treated as rebuildable derived state.

## Vector search: pgvector vs TruthDB (intended direction)

PostgreSQL supports vector similarity search through `pgvector`, a third-party extension. While functional, it inherits PostgreSQL's storage model constraints:

- Vectors are stored as column values in heap pages, subject to MVCC overhead and TOAST compression.
- The HNSW or IVFFlat index is a standard PostgreSQL index — it must be vacuumed, can bloat, and rebuilds are expensive.
- Vector search is a query operator, not a first-class retrieval path. Hybrid queries (vector + full-text + structured) require manual score fusion in application code or complex SQL.

TruthDB's intended direction:

- Vectors are WAL events — they follow the same append, replay, and snapshot lifecycle as all other data.
- The ANN index is a derived structure, rebuildable from the WAL. Model upgrades mean re-embedding and cutover, not ALTER TABLE + REINDEX.
- Hybrid queries (vector + text + structured) are a single query operation with built-in score fusion.
- No extension installation, no TOAST overhead, no MVCC bloat on immutable embeddings.

See:

- `VECTOR_DATABASE_AND_RAG.md`
- `WAL.md`
- `ADR-0001-wal-is-source-of-truth.md`
