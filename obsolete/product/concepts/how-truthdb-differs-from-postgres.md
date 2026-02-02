# How TruthDB Differs From PostgreSQL (Conceptual)

Status: draft
Last updated: 2026-01-23

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

See:

- `development/specs/WAL.md`
- `development/decisions/ADR-0001-wal-is-source-of-truth.md`
