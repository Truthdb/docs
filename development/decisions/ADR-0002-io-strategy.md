# ADR-0002: IO Strategy and Async Runtime Boundary

Status: proposed (needs implementation alignment)
Last updated: 2026-01-23

## Context

TruthDB’s core performance and correctness goals depend on explicit durability boundaries and predictable IO behavior.

Prior research/discussion (including TigerBeetle’s approach) highlights that “general async runtimes” can be awkward for database-grade storage engines:

- They often assume network-style readiness semantics rather than explicit flush/durability semantics.
- Combining io_uring, fsync ordering, and backpressure correctly can fight the runtime’s abstractions.

At the same time, Rust’s ecosystem strongly supports Tokio for non-storage concerns (RPC, orchestration, general service structure).

## Decision (proposed)

Treat storage IO as its own explicit subsystem with clear boundaries.

- The WAL write path should be owned by a single-writer component with explicit batching and durability controls.
- Multi-producer → single-writer queue is preferred over concurrent file writes.
- Reader tails/snapshot readers must not block the writer.

We may still use Tokio in the overall service, but the storage core should not rely on “ambient runtime behavior” for correctness.

## Consequences

### Positive

- Makes durability boundaries explicit and testable.
- Improves predictability (batching, flush ordering, backpressure).
- Allows swapping implementation approaches (blocking + dedicated thread, io_uring loop, direct IO) behind a stable interface.

### Negative / trade-offs

- More plumbing up-front (queues, backpressure reporting, shutdown coordination).
- Potentially less ergonomic than “just async everything”.

## Alternatives considered

- **Fully Tokio-async file IO in the hot path**
  - Pros: simpler integration.
  - Cons: hard to reason about fsync ordering/backpressure and file IO semantics across platforms.

- **Pure blocking IO everywhere**
  - Pros: simplest semantics.
  - Cons: may limit throughput/latency unless carefully structured.

## Next steps

- Define the storage interface boundary (writer API, durability modes, error model).
- Decide whether the first implementation is:
  - blocking IO + dedicated thread, or
  - an io_uring-based loop (Linux-only) with portability fallback.
- Add benchmarks for sequential append + group sync.

## Related

- WAL requirements: `development/specs/WAL.md`
