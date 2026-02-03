# TruthDB Storage (Single-File) — Draft Spec

Status: draft (working doc)
Last updated: 2026-02-02

This document is a collaborative spec for TruthDB’s **single-file storage layout**. The goal is one file that contains **everything**: WAL, metadata, data pages, free-space tracking, and snapshots.

This is intentionally verbose so design choices are explicit and reviewable.

---

## 1. Goals

- **Single file** contains the entire database state.
- **Preallocated file** for predictable IO and layout.
- **WAL is the source of truth**, but still inside the same file.
- **Crash safety** with deterministic recovery.
- **High-throughput sequential writes** on the hot path.
- **Corruption detection** and safe stop at last valid state.
- **Simple, explicit layout** that can evolve via versioning.

## 2. Non-goals (for v0)

- Multi-file segment management.
- Advanced compaction/GC strategies.
- Multi-writer concurrency (see section 3.1).
- Cross-node replication details.

## 3.1 Concurrency model (v0)

- **Multiple producers → single writer**: application threads enqueue write requests.
- **Single writer thread/task** serializes all file IO (WAL, data pages, superblock).
- **No concurrent file writes** are allowed.

### Rationale

- Ensures **total ordering** for deterministic replay.
- Avoids interleaving/torn writes across threads/processes.
- Simplifies durability and recovery logic.
- Keeps performance predictable with batching/group commit.

---

## 3. High-level layout (single file)

The file is divided into fixed, page-aligned regions. The intent is:

1) **Fixed header** (small, constant-size)
2) **Dual superblocks** (A/B, for atomic commits)
3) **WAL region** (append-only, sequential)
4) **Data region** (COW pages / records)
5) **Metadata region** (schemas, indexes, manifests)
6) **Free-space region** (allocator state)
7) **Snapshot region** (optional, or just snapshot roots)

This is a single file. “Regions” are logical ranges inside a **preallocated** file.

---

## 4. Rationale for the single-file layout

**Why one file?**
- Simplifies deployment and operational handling.
- Avoids cross-file consistency problems.
- Enables atomic state switching via superblocks.

**Why fixed regions?**
- Easier to reason about and debug.
- Allows explicit IO patterns (sequential WAL, random data reads).
- Enables future direct-IO alignment without redesign.

**Why preallocation?**
- Predictable IO and layout (no runtime file growth).
- Fewer fragmentation surprises.
- Fast, explicit failure when capacity is exhausted.
- Simplifies allocator bookkeeping (fixed maximum ranges).

---

## 5. File header (fixed-size)

### 5.1 Structure
- `magic` (8 bytes) — identifies TruthDB file
- `version` (u32)
- `page_size` (u32) — **4K** (chosen for mixed workloads and OS page alignment)
- `file_uuid` (16 bytes)
- `created_salt` (16 bytes)
- `superblock_a_offset` (u64)
- `superblock_b_offset` (u64)
- `header_checksum` (u64)

### 5.2 Rationale
- **Magic+version** allow forward/backward compatibility checks.
- **Page size** is a global invariant for all regions.
- **UUID/salt** support uniqueness and corruption diagnostics.
- **Superblock offsets** avoid hard-coding positions.
- **Checksum** catches torn or corrupted headers early.

---

## 6. Superblocks (A/B)

Two copies of a small “commit record” are kept. Only one is active.

### 6.1 Structure
- `generation` (u64) — monotonically increasing
- `active` (u8) — which copy is latest
- `wal_head` (u64)
- `wal_tail` (u64)
- `last_committed_seq` (u64)
- `metadata_root` (u64)
- `data_root` (u64)
- `allocator_root` (u64)
- `snapshot_root` (u64, optional)
- `checksum` (u64)

### 6.2 Rationale
- **A/B copies** give atomic commit without complex journaling.
- **Generation** allows last-good detection after crash.
- **Root pointers** make the data layout versioned and COW-friendly.
- **Checksum** ensures we never accept a torn superblock.

---

## 7. WAL region

The WAL is the authoritative ordered log of all state changes. In a fixed-size file, the WAL is a **ring buffer** with a configurable capacity.

### 7.1 Entry format (v0)
- Entry header:
  - `entry_type` (u16)
  - `entry_version` (u16)
  - `payload_len` (u32)
  - `seq_no` (u64)
  - `logical_ts` (u64)
  - `header_crc` (u64)
  - `payload_crc` (u64)
- Payload bytes
- Footer (**required**):
  - `payload_len` (u32) duplicate
  - `payload_crc` (u64) duplicate

Entry types (v0):
- `record` — data/metadata changes
- `commit` — marks a batch as durable/visible

### 7.2 Rationale
- **Seq number** gives total ordering.
- **Logical timestamp** allows replay ordering independent of wall-clock.
- **Checksums** detect corruption/torn writes.
- **Footer (required)** makes partial write detection more robust.

### 7.3 Recovery rule
- Scan WAL sequentially from `wal_head`.
- Stop at first invalid entry.
- `last_committed_seq` becomes last valid entry.

### 7.4 Retention rule

- WAL entries may be overwritten only after a snapshot/root makes them reclaimable.
- The writer must refuse new writes if the ring would overwrite required history.

---

## 8. Data region (copy-on-write)

All mutable data is written via copy-on-write (COW):
- New pages are appended into free space.
- WAL records the new root.
- Superblock switches to new root on commit.

### Rationale
- COW avoids in-place corruption risk.
- WAL + superblock commit gives a single, deterministic recovery path.

---

## 9. Metadata region

Contains schema definitions, indexes, and manifest structures.

- Stored as pages like data region.
- Root pointer stored in superblock.

### Rationale
- Metadata is part of the authoritative state, so it must be versioned and recoverable.

---

## 10. Allocator / free-space

Free-space tracking is stored as pages in a dedicated region.

v0 choice:
- **Bitmap pages** (simple, fast, predictable)

### Rationale
- Keeping allocator state inside the file enables full self-containment.

---

## 11. Snapshot region (optional)

Snapshots are optional in v0. If used:
- A snapshot is just a root pointer stored periodically.
- Snapshot roots are stored in a **dedicated snapshot region**.
- Snapshot cadence: **hybrid** (time-based minimum + WAL-usage threshold).

### Rationale
- Keeps snapshot logic simple while remaining compatible with WAL-first recovery.

---

## 12. Commit model

Proposed v0 commit sequence:
1) Append WAL `record` entries describing new roots/changes.
2) fsync WAL (sync or group_sync mode).
3) Append WAL `commit` entry for the batch.
4) Write new superblock copy.
5) fsync superblock.

**Commit boundary is the superblock switch.**
If crash happens before superblock write, WAL may contain entries not yet committed.

---

## 12.1 Ingestion → storage write path (v0)

This section defines how incoming messages become durable storage writes.

### Write path

1) **Ingress** (network/task) validates request and enqueues a write job.
2) **Batcher** groups jobs by size/time (hybrid) and assigns a `batch_id`.
3) **IO writer** (single thread/task) performs:
  - write new data/metadata pages (COW)
  - append WAL `record` entries for the batch
  - append WAL `commit`
  - update superblock

### Durability modes (v0)

- **async**: no fsync; OS buffers control durability.
- **sync**: fsync per batch.
- **group_sync**: fsync after **N=32** batches or **T=5ms**, whichever comes first.

### Backpressure

- If WAL ring is near full, ingress must apply backpressure.
- Hybrid: **block up to a timeout**, then reject with a retryable error until snapshot frees space.

---

## 12.2 Observability (storage + network)

Observability must cover both storage IO and the ingress/network layer so operators can diagnose saturation and latency.

### Storage metrics/logs

- WAL ring usage (% full), head/tail positions
- fsync latency (p50/p95/p99)
- batch sizes and batch flush intervals
- snapshot duration and frequency
- allocator bitmap occupancy and fragmentation hints
- backpressure events (count, duration)

### Network/ingress metrics

- request rate, queue depth, and enqueue latency
- rejected requests (backpressure timeouts)
- end-to-end write latency (ingress → commit)

### Tracing/logging guidelines

- Structured logs with `batch_id`, `commit_seq`, and request IDs
- Trace spans for: enqueue → batch → WAL write → fsync → commit
- Error logs for WAL corruption detection and recovery stop points

---

## 12.3 Performance considerations

- **Single-writer throughput** depends on batching. Larger batches reduce fsync cost per entry.
- **Preallocation** improves predictability by avoiding file growth and fragmentation.
- **Page size** affects IO efficiency and cache behavior (4K vs 8K tradeoff).
- **COW overhead** adds write amplification but keeps recovery simple and safe.
- **Sequential WAL writes** are the primary hot path; keep them contiguous and aligned.

### io_uring considerations

- Linux-only; **no cross-platform support is required**.
- Direct IO (if used) requires aligned buffers and IO sizes to page boundaries.
- Durability ordering must be explicit in the submission chain (write → fsync).
- Shutdown must flush the ring and verify completions before returning.

Risks to watch:
- Small batch sizes can cap throughput due to fsync latency.
- Overly small pages increase metadata overhead.
- Overly large pages waste IO for small updates.

---

## 12.4 Security considerations

- **Checksums** (header + payload) prevent silent corruption from propagating.
- **Dual superblocks** reduce risk of accepting torn metadata.
- **Deterministic replay** avoids ambiguity during recovery.

Checksum algorithm (v0): **xxHash64** (chosen for speed; corruption detection only).

Risks to watch:
- Bit-rot on long-lived storage without periodic verification.
- Weak or inconsistent checksum coverage across regions.
- Unsafe recovery paths that attempt “repair” without operator intent.

---

## 13. Open decisions (need your input)

None currently.

---

## 14. Byte-level layout map (v0)

All offsets are **page-aligned (4K)**. Sizes shown are defaults and can be adjusted by config at file creation.

### 14.1 Fixed areas

- **Header**: 1 page (4K)
- **Superblock A**: 1 page
- **Superblock B**: 1 page

### 14.2 Regions (default sizes within 10GB file)

Let total file size = 10GB.

- **WAL ring**: 5% (min 256MB, max 1GB). For 10GB: **512MB**.
- **Data region**: 66% (~6.6GB)
- **Metadata region**: 8% (0.8GB)
- **Allocator bitmap region**: 2% (0.2GB)
- **Snapshot region**: 2% (0.2GB)
- **Reserved/expansion**: 17% (~1.7GB)

These ratios are defaults and **configurable at file creation**.

### 14.3 Region ordering (v0)

1) Header (4K)
2) Superblock A (4K)
3) Superblock B (4K)
4) WAL ring
5) Data region
6) Metadata region
7) Allocator bitmap region
8) Snapshot region
9) Reserved/expansion

### 14.4 Notes

- Offsets are derived at file creation and stored in the header.
- Reserved space is for future format growth without migration.

---

## 14.5 Storage configuration (v0)

The storage subsystem is configured at startup. Suggested keys:

- `storage.path` — file path for the single storage file
- `storage.size_gb` — preallocated file size (default 10)
- `storage.page_size` — fixed at 4096
- `storage.wal_ratio` — default 0.05 (min 256MB, max 1GB)
- `storage.metadata_ratio` — default 0.08
- `storage.snapshot_ratio` — default 0.02
- `storage.allocator_ratio` — default 0.02
- `storage.reserved_ratio` — default 0.17
- `storage.group_sync_batches` — default 32
- `storage.group_sync_ms` — default 5
- `storage.backpressure_timeout_ms` — timeout before reject
- `storage.snapshot_min_interval_ms` — time-based cadence floor
- `storage.snapshot_wal_threshold` — usage threshold for snapshot trigger

---

## 15. WAL payload schema (v0)

The WAL entry header/footer is fixed as defined in section 7. The **payload** depends on `entry_type`.

### 15.1 `record` payload

Represents a single logical change (data/metadata/allocator updates).

Fields:

- `batch_id` (u64) — groups multiple records before a `commit`
- `subsystem` (u16) — identifies which vertical (data, metadata, allocator, etc.)
- `flags` (u16)
- `target_root` (u64) — new root pointer for the subsystem (or 0 if none)
- `payload_bytes` (opaque) — subsystem-specific encoding

### 15.2 `commit` payload

Marks a batch as durable and visible.

Fields:

- `batch_id` (u64)
- `commit_seq` (u64)
- `commit_flags` (u16)

### 15.3 Recovery rules

- Replays `record` entries only if a matching `commit` is present for the `batch_id`.
- Uncommitted batches are ignored.

---

## 16. Open decisions (need your input)

None currently.

---

## 17. Next steps

1) Add storage config (paths, file size, region ratios) to config.
2) Define on-disk structs/constants (header, superblock, WAL entry).
3) Implement file creation + preallocation + layout initialization.
4) Implement WAL ring append + checksum + required footer verification.
5) Implement recovery scan to last valid commit.
6) Add minimal tests for header/superblock/WAL parsing.
7) Implement allocator bitmap operations (alloc/free, persist bitmap pages).
8) Implement snapshot creation + retention (hybrid cadence).
9) Wire ingestion path to storage writer (batching + group_sync).
10) Add basic metrics/logs for WAL usage, snapshot cadence, backpressure events.

---

## 18. Benchmarking

Benchmarking is required to validate and tune:

- batch sizes and group_sync defaults (N, T)
- WAL ring size ratios
- snapshot cadence thresholds
- allocator performance under churn

Use representative workloads (small writes, bursty streams, mixed read/write) and capture latency percentiles and throughput.

