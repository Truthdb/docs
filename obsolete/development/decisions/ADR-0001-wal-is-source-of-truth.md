# ADR-0001: WAL is the Source of Truth

Status: accepted (architectural direction)
Last updated: 2026-01-23

## Context

TruthDB is intended to be an event-sourced, log-first system.
We want deterministic replay, observable durability boundaries, and rebuildable derived state.

Traditional databases often treat a mutable storage engine as the “truth” and use a WAL as a recovery mechanism.
TruthDB’s direction is different: the WAL is the durable ordered substrate that other structures derive from.

## Decision

The TruthDB WAL is the **authoritative source of truth**.

- Materialized state (tables), indexes, and snapshots are derived structures.
- Snapshots exist only as performance optimizations and must be reproducible from WAL bytes.
- Correctness is defined in terms of WAL ordering + deterministic replay.

## Consequences

### Positive

- Deterministic rebuild of derived state is a core capability.
- Replication/shipping can be byte-exact and order-preserving.
- Debugging/auditing becomes log-centric and inspectable.

### Negative / trade-offs

- Requires careful definition of “commit” and durability semantics.
- Requires strict replay determinism (no wall-clock or hidden nondeterminism in core derivations).
- May require more explicit lifecycle management (segment sealing, snapshot coordination, retention).

## Related

- Specs: `development/specs/WAL.md`
- Comparisons: PostgreSQL WAL vs TruthDB; TigerBeetle WAL comparison (in gptinput)
