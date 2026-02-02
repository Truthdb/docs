# TruthDB WAL (Write-Ahead Log) — Requirements

Status: draft (promoted from prior agreement)
Last updated: 2026-01-23

This is a design-level requirements specification for the TruthDB Write-Ahead Log (WAL).

The WAL is the durability and ordering substrate. It is not a query engine, not an index, and not a materialized state format.

## 1. Purpose & scope

The WAL is the authoritative durability and ordering mechanism for all TruthDB state changes.

It must support:

- crash safety
- deterministic replay
- high-throughput sequential writes
- corruption detection and safe stop semantics

The WAL is the foundation for:

- event-sourced subsystems
- recovery and replay
- replication/shipping (byte-exact)
- audit/inspection

## 2. Core principles

- Append-only
- Sequential IO optimized
- Crash-consistent
- Deterministic replay
- Minimal abstraction leakage
- Explicit over implicit

## 3. Functional requirements

### 3.1 Append semantics

- Entries MUST be appended sequentially.
- No in-place modification is allowed.
- No compaction/deletion/rewriting occurs inside active WAL segments.

### 3.2 Durability guarantees

The WAL MUST support configurable durability modes:

- `async`: OS-buffered
- `sync`: fsync at commit boundary
- `group_sync`: batched fsync

A committed entry MUST survive:

- process crash
- kernel panic
- power loss (assuming storage honors fsync semantics)

### 3.3 Ordering & identity

Every WAL entry MUST have:

- a monotonically increasing sequence number
- a logical timestamp (not wall-clock dependent)

Ordering MUST be total within a WAL stream.

### 3.4 Entry structure

Each entry MUST contain:

- **Header**:
  - version
  - entry type
  - payload length
  - checksum
- **Payload**:
  - opaque bytes (TruthDB-level encoding)
- **Footer** (optional):
  - duplicate length and/or checksum (torn-write detection)

The WAL MUST NOT interpret payload contents.

### 3.5 Checksums & corruption detection

- Every entry MUST be protected by a fast checksum (xxHash64 was previously discussed).
- Replay MUST:
  - detect partial writes
  - stop safely at the last valid entry
- Silent corruption MUST NOT propagate into state replay.

## 4. Segmentation & file layout

### 4.1 WAL segments

- The WAL MUST be split into fixed-size segments.
- Segment size MUST be configurable (build-time or runtime).
- Rollover MUST be deterministic.

### 4.2 Segment lifecycle

Each segment is in exactly one state:

1. Active (currently being written)
2. Sealed (complete, immutable)
3. Archived (eligible for retention / snapshot pruning)

No segment may transition backwards.

### 4.3 File naming & discovery

- Segment filenames MUST encode:
  - starting sequence number
  - segment index
- The WAL MUST be discoverable via directory scan only.
- No external metadata is required to locate the WAL.

## 5. Replay requirements

### 5.1 Replay determinism

- Given the same WAL bytes, replay MUST produce identical results.
- Replay order MUST be strictly defined.
- Replay MUST be independent of wall-clock time.

### 5.2 Partial replay

The WAL MUST support replay from:

- start
- arbitrary sequence number
- segment boundary

Replay MUST stop cleanly on corruption.

## 6. Concurrency model

### 6.1 Writers

The WAL MUST support:

- single-writer (baseline)
- multi-producer → single-writer queue

No concurrent file writes are allowed.

### 6.2 Readers

The WAL MUST support concurrent readers:

- tailing readers
- snapshot readers

Readers MUST NOT block writers.

## 7. Performance requirements

### 7.1 Write path

The WAL MUST be optimized for:

- large sequential writes
- minimal syscalls
- cache-friendly batching

No allocation on the hot path after initialization.

### 7.2 Backpressure

The WAL MUST expose backpressure signals:

- queue depth
- fsync latency
- disk saturation

TruthDB MAY reject writes rather than buffer unboundedly.

## 8. Integration requirements

### 8.1 Snapshotting

The WAL MUST support coordination with snapshots:

- snapshot at sequence N
- WAL segments < N become reclaimable

The WAL itself does not create snapshots.

### 8.2 Replication & streaming

- The WAL MUST be streamable (byte-exact, order-preserving).
- Remote replication MUST NOT require WAL interpretation.

## 9. Failure semantics

### 9.1 Crash recovery

On startup, recovery MUST:

1. scan segments in order
2. validate entries
3. stop at the first invalid entry
4. expose last committed sequence number

No silent repair attempts are allowed.

### 9.2 Corruption policy

Corruption MUST be:

- detected
- reported
- isolated

The WAL MUST NOT attempt heuristic recovery without explicit operator action.

## 10. Non-goals

The WAL explicitly does not:

- enforce schemas
- perform deduplication
- compact itself
- manage retention policy
- interpret events
- guarantee cross-stream ordering

## 11. Invariants (must always hold)

- append-only
- entries are immutable once committed
- replay is deterministic
- corruption is detectable
- ordering is explicit and total

## Open questions / follow-ups

- Exact binary format and framing (header/footer layout; alignment)
- Commit/transaction boundaries (what “committed” means)
- Checksumming details (what is covered; whether there is a segment-level checksum)
- Segment naming convention and retention policy integration
- io_uring/direct-IO mapping and portability constraints
