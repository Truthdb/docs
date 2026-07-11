# IO_URING Storage Plan

## Purpose

This document describes what it would take to move TruthDB to a Linux-only `io_uring` disk I/O architecture.

The goal is not to "sprinkle in async I/O." The goal is to redesign the disk storage path so TruthDB can use Linux `io_uring` deliberately and correctly on the hot write path.

## Why This Matters

TruthDB is intended to be extremely fast.

If that remains a core product requirement, then the storage path should move away from synchronous `std::fs` I/O and toward a Linux-native design that:

- reduces syscall overhead
- batches writes efficiently
- supports explicit completion handling
- preserves strong durability semantics

`io_uring` is the most relevant Linux technology for that direction.

## Current State

Today, TruthDB does not use `io_uring`.

Current storage characteristics:

- file I/O is synchronous
- storage uses `std::fs::File`
- reads and writes use `Read`, `Seek`, and `Write`
- WAL appends currently depend on mutable file-position state

This is simple and acceptable for the current development phase, but it is not the target architecture if maximum storage performance is a serious goal.

## Design Principle

This plan is only about disk access.

It is not a networking plan, and it is not a runtime-wide rewrite plan.

The intended direction is:

- Linux only
- disk path only
- `io_uring` only as the target disk I/O model
- focus first on the WAL path
- use the aggressive path rather than a conservative buffered path

The WAL is already the architectural source of truth, so it is the correct place to begin.

## Main Architectural Changes Required

### 1. Introduce A Storage Backend Abstraction

TruthDB needs a clear boundary between:

- engine logic
- WAL/data persistence logic

The current code writes directly through one concrete storage implementation.

That needs to become a backend boundary with at least:

- open/create storage
- append WAL record
- read WAL records for replay
- durability boundary handling
- close/shutdown

This boundary is still required even if the intended end state is `io_uring` only, because the current code must be refactored out of direct `File` usage before the aggressive Linux path can take over cleanly.

## 2. Move To Offset-Based I/O

The current storage code uses `Seek` and a mutable file cursor.

That is the wrong shape for `io_uring`.

For `io_uring`, TruthDB should issue reads and writes using explicit file offsets. This avoids shared file-position state and makes async submission/completion much safer and clearer.

This means:

- WAL append logic must compute exact write offsets
- WAL replay logic must read from explicit offsets
- correctness must not depend on a mutable global file cursor

## 3. Introduce A Single-Writer Storage Actor

TruthDB should have exactly one owner of the WAL write path.

Recommended model:

- many producers can submit write requests
- one storage task owns the file descriptors and the `io_uring` ring
- that storage task serializes WAL appends

This matches the existing WAL direction very well:

- total write ordering stays explicit
- no concurrent file writes are needed
- batching becomes possible without giving up correctness

## 4. Redefine Durability In Terms Of Submission And Completion

With synchronous I/O, durability is easy to reason about:

- write
- flush
- continue

With `io_uring`, TruthDB must reason about:

- submission queue entries
- completion queue entries
- write completion
- fsync or fdatasync completion
- commit acknowledgment timing

TruthDB should not acknowledge a durable commit until the relevant completion path has finished successfully.

That implies:

- explicit request IDs or sequence tracking
- explicit completion handling
- clear mapping from "write accepted" to "write durable"

## 5. Commit To Linux-Only Disk I/O

TruthDB disk I/O should be treated as Linux only.

TruthDB should:

- target Linux as the runtime that matters
- treat `io_uring` as the required disk I/O model
- fail clearly if the required Linux features are unavailable

Non-Linux development convenience is not the design driver for this plan.

## 6. Add Capability Detection

TruthDB must not assume all Linux systems support the same `io_uring` features.

At startup, the `io_uring` backend should verify:

- kernel support level
- ring setup success
- any optional features needed by the chosen mode

If those checks fail, TruthDB should fail clearly at startup rather than silently downgrade to a different disk path.

## 7. Redesign Batching Around Group Commit

The biggest early performance win is likely not "async reads everywhere."

The first serious win is:

- batched WAL appends
- group commit
- fewer fsync boundaries

So the `io_uring` backend should be designed around:

- multiple writes staged together
- one durability boundary for a batch
- completion fan-out back to waiting writers

This fits the existing configuration direction around group sync.

## Target Backend Shape

The target backend should be intentionally aggressive.

Required direction:

- `O_DIRECT`
- explicit offsets
- one ring per storage actor
- one WAL writer
- registered file descriptors
- registered buffers / fixed buffers
- aligned preallocated I/O memory
- write + fsync completion tracking
- no buffered file I/O path as the target design

This means TruthDB must take ownership of the ugly parts instead of delegating them to the page cache:

- alignment discipline
- buffer lifecycle discipline
- direct-I/O-safe write sizing
- explicit durability boundaries
- explicit batching

`SQPOLL` and `IOPOLL` may still be staged later, but `O_DIRECT`, registered files, and registered buffers are part of the intended direction, not optional extras.

## Suggested Runtime Model

High-level model:

1. server/engine sends a WAL append request to the storage actor
2. storage actor reserves sequence number and offset
3. storage actor submits write requests to `io_uring`
4. storage actor optionally links or follows with fsync/fdatasync
5. storage actor waits for completions
6. storage actor updates durable sequence boundary
7. storage actor replies to waiting engine task

This keeps the correctness story explicit.

## Required Refactors In The Existing Code

The likely refactor sequence is:

### Phase 1: Storage Boundary Cleanup

- separate storage API from storage file internals
- remove direct engine assumptions about concrete storage internals
- make WAL append/read operations explicit API calls

### Phase 2: Offset-Based WAL Logic

- replace file-cursor-dependent logic with offset-based logic
- make replay logic offset-driven
- keep synchronous backend working

### Phase 3: Storage Actor

- introduce a dedicated storage task
- serialize WAL appends through that task
- move durability acknowledgment into that task

### Phase 4: Linux `io_uring` Backend

- add Linux-only backend implementation
- open files and manage ring lifecycle
- submit write/read/fsync requests
- handle completions
- enforce direct-I/O alignment rules

### Phase 5: Group Commit

- batch multiple write requests
- amortize fsync cost
- return acknowledgments only after durability boundary completes

### Phase 6: Aggressive Resource Registration

- move file descriptors into registered file tables
- move hot-path I/O buffers into registered buffers
- eliminate avoidable per-request resource setup

### Phase 7: Replay And Recovery Validation

- verify WAL replay still behaves identically
- validate crash-recovery semantics
- validate partial-write handling under the new backend

### Phase 8: Benchmarking

- build a proper Linux benchmark harness
- compare synchronous backend vs `io_uring`
- measure:
  - append throughput
  - tail latency
  - fsync latency
  - replay throughput

## Configuration Direction

TruthDB should eventually support explicit storage mode configuration, but the end-state design target is `io_uring`.

```toml
[storage]
backend = "io_uring"
```

During transition there may still be temporary compatibility modes, but that is migration scaffolding, not the product direction.

## Correctness Risks

This work has real correctness risk and should be treated carefully.

Major risk areas:

- acknowledging writes before durability is real
- losing total write ordering
- mishandling completion ordering
- offset/accounting bugs around WAL wrap or segment boundaries
- broken shutdown semantics with in-flight requests
- direct-I/O alignment violations
- misuse of registered buffers or registered files

The storage backend must remain boringly correct before it tries to be clever.

## Performance Reality Check

Using `io_uring` does not automatically make TruthDB the fastest database in the world.

Performance will still depend on:

- WAL format
- batching strategy
- fsync policy
- memory layout
- allocator behavior
- indexing strategy
- query engine design
- CPU cache behavior
- storage device characteristics

`io_uring` is an enabling technology, not a magic result.

## Verification Constraint

`io_uring` needs a native Linux kernel for runtime verification.

In practice:

- building TruthDB in Docker on macOS is still useful
- `linux/arm64` on Apple Silicon is a valid runtime candidate if the Linux VM/container runtime exposes `io_uring`
- container runtime policy can still block `io_uring` with `EPERM` even on `linux/arm64`
- running `linux/amd64` under emulation on macOS is not authoritative for `io_uring`
- emulated containers can fail at `io_uring_setup` with `ENOSYS` before TruthDB logic even starts
- final verification for this work should happen on a real Linux host or VM with native kernel support

## Recommended First Milestone

The first milestone should be:

- Linux-only experimental backend
- WAL append path only
- explicit offset-based direct I/O writes
- one storage actor
- batchable write queue
- fsync completion tracking
- `O_DIRECT`
- registered files
- registered buffers
- strict alignment rules enforced
- fail-fast startup if requirements are not met

That is the smallest milestone that is still architecturally meaningful.

## Explicit Non-Goals For V1

The first `io_uring` milestone should not attempt:

- a networking rewrite
- rewriting the entire database around `io_uring` on day one
- broad host-OS compatibility
- performance marketing claims before Linux benchmarks exist

## Success Criteria

The first `io_uring` storage milestone is successful if:

1. TruthDB can run on Linux with an `io_uring` backend
2. the WAL append path remains correct and replayable
3. durability semantics are explicit and tested
4. direct I/O requirements are enforced rather than bypassed
5. Linux benchmarks show measurable improvement on the WAL path

## Conclusion

TruthDB can move to `io_uring`, but only through a deliberate storage redesign.

The right path is:

- abstract storage
- make WAL I/O offset-based
- introduce a single-writer storage actor
- add a Linux-only aggressive `io_uring` disk backend
- use `O_DIRECT`
- use registered files and registered buffers
- benchmark and validate before claiming success

That is a serious project, but it is a coherent one.
