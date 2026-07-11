# Data Region, Snapshots, And WAL Reclamation

## Status
Planned.

## Problem

TruthDB cannot run indefinitely.

The WAL is a fixed-size ring buffer. Every write appends to the tail, but the head never advances. Once the ring fills, the engine returns `WalFull` and all writes fail. There is no way to reclaim WAL space because the entire engine state depends on replaying every WAL entry from head to tail on startup.

This is the single biggest architectural blocker in the system today.

## What Exists Today

The storage file already has preallocated regions for data, snapshots, an allocator bitmap, and metadata. None of them are used. The relevant fields in `FileHeader` (`data_offset`, `data_size`, `snapshot_offset`, `snapshot_size`, `allocator_offset`, `allocator_size`) point at real disk space, and `Superblock` has `data_root`, `snapshot_root`, and `allocator_root` fields, but nothing reads or writes those regions.

Current behavior:

- `Engine::new()` replays **all** WAL entries from `wal_head` to `wal_tail` to rebuild in-memory state
- `append_wal_entry()` advances `wal_tail` but never touches `wal_head`
- Free space is calculated as `wal_size - (wal_tail - wal_head)`, so advancing `wal_head` would immediately reclaim space
- Both superblocks are always written identically — the A/B alternation pattern is not functional
- `EngineState` (indexes, documents, schemas) lives entirely in memory

## Goal

Make TruthDB capable of running indefinitely by:

1. Persisting engine state to the data region (checkpoint)
2. Recording checkpoint markers in the snapshot region
3. Advancing `wal_head` past entries that are covered by a completed checkpoint
4. Fixing the superblock A/B pattern so checkpoint commits are atomic

## Design

### Core Cycle

The steady-state cycle is:

1. Normal operations append WAL entries as today
2. A checkpoint trigger fires (WAL usage threshold, entry count, or explicit request)
3. Engine state is serialized and written to the data region
4. A snapshot descriptor is written to the snapshot region recording what was checkpointed
5. The active superblock is updated to point at the new snapshot and advance `wal_head`
6. The alternate superblock is written with the same state
7. WAL space behind the new `wal_head` is now reclaimable

### Data Region

The data region is the durable equivalent of the in-memory `EngineState`.

For v1, the serialization strategy is a flat bincode blob:

- Serialize `EngineState` (indexes, documents, schemas) into a contiguous byte buffer
- Write that buffer to the data region at page-aligned offsets
- Record the offset and length in the snapshot descriptor

Postings (inverted index structures) are **not** serialized. They are derived data built from documents and are rebuilt on startup after loading the checkpoint.

Why flat blob and not a B-tree:

- The data region is a checkpoint, not a random-access store
- Checkpoints are written sequentially, not updated in place
- A B-tree page structure adds complexity without benefit at this stage
- The flat blob is simpler to verify and debug

The data region supports two checkpoint slots (A and B) to enable copy-on-write safety. A new checkpoint is always written to the inactive slot. If the write fails or the process crashes mid-write, the previous checkpoint in the other slot remains valid.

### Page Allocator

The allocator bitmap region tracks which pages in the data region are in use.

For v1:

- Each bit represents one 4 KB page in the data region
- Allocation uses first-fit contiguous scan
- The bitmap is small (data region size / 4096 / 8 bytes) and fits in a few pages
- The bitmap is persisted alongside the snapshot descriptor

The allocator does not need to be sophisticated for v1 because checkpoints are bulk writes that replace the entire previous checkpoint. Fragmentation is not a concern when there are only two alternating slots.

### Snapshot Descriptors

The snapshot region holds small checkpoint markers. These are not the data itself — they are bookmarks that say:

- "At sequence number N, the full engine state was written to data region offset X, length Y, in slot A/B"
- Generation number (matches superblock generation)
- Checksum of the serialized data
- Timestamp

A snapshot descriptor fits in a single page. The snapshot region holds two descriptors (one per slot), matching the two data region checkpoint slots.

### Crash-Safe Snapshot Creation

The write sequence for a checkpoint is:

1. Serialize engine state to a buffer
2. Compute checksum of the serialized data
3. Determine the target slot (opposite of the currently active slot)
4. Write serialized data to the data region at the target slot's offset
5. `fdatasync`
6. Write the snapshot descriptor to the snapshot region
7. `fdatasync`
8. Write the new superblock to the **next** superblock (A or B, alternating) with:
   - Incremented generation number
   - Updated `wal_head` (advanced past checkpointed entries)
   - Updated `data_root` pointing to the new data
   - Updated `snapshot_root` pointing to the new descriptor
   - Updated `last_committed_seq` reflecting the checkpoint boundary
9. `fdatasync`
10. Write the same state to the other superblock as a backup
11. `fdatasync`

If the process crashes at any point before step 9 completes, the previous superblock still points at the previous valid checkpoint. Recovery uses the superblock with the highest valid generation.

### WAL Reclamation

After a successful checkpoint:

- `wal_head` is advanced to the sequence number following the last checkpointed entry
- No data is zeroed or overwritten — the ring simply treats the space between old head and new head as free
- Free space calculation (`wal_size - (wal_tail - wal_head)`) immediately reflects the reclaimed space
- Future WAL entries can wrap around and reuse that space

### Modified Startup Flow

Current startup:

1. Open file, validate header and superblocks
2. Replay all WAL entries from `wal_head` to `wal_tail`
3. Engine is ready

New startup:

1. Open file, validate header and superblocks
2. Pick the superblock with the highest valid generation
3. If `snapshot_root` is nonzero:
   a. Read the snapshot descriptor from the snapshot region
   b. Validate the descriptor's checksum and generation
   c. Read the serialized engine state from the data region
   d. Validate the data checksum
   e. Deserialize into `EngineState`
   f. Set `next_seq_no` and `next_doc_id` from the checkpoint state
4. Replay only WAL entries from `wal_head` to `wal_tail` (these are the entries *after* the checkpoint)
5. Rebuild postings from the loaded documents
6. Engine is ready

The key improvement: if a checkpoint covers entries 1–10000 and WAL only contains entries 10001–10050, startup replays 50 entries instead of 10050. This also means startup time stays bounded regardless of total history.

### Superblock A/B Fix (Prerequisite)

The current code writes both superblocks identically on every WAL append. This must be fixed before snapshots can work.

Required changes:

- Track which superblock is "active" (use the `active` field that already exists in the `Superblock` struct)
- On normal WAL appends: write only the active superblock
- On checkpoint: write the new state to the *inactive* superblock first, then make it active
- On recovery: compare both superblocks, pick the one with the highest valid generation
- The `generation` field already exists and is incremented — it just needs to be used for selection

### Checkpoint Triggers

V1 should support at least:

- **WAL usage threshold**: trigger when WAL is N% full (e.g., 75%)
- **Manual trigger**: an explicit engine command to force a checkpoint

Future triggers (not v1):

- Time-based (every N seconds)
- Entry-count-based (every N entries)
- Idle-based (checkpoint when write load drops)

## Implementation Phases

### Phase 1: Superblock A/B Fix

Fix the dual-superblock pattern so that recovery chooses the superblock with the highest valid generation. Stop writing both identically.

Files: [storage.rs](truthdb/crates/truthdb-core/src/storage.rs), [storage_layout.rs](truthdb/crates/truthdb-core/src/storage_layout.rs)

### Phase 2: Page Allocator

Implement the bitmap allocator for the data region. Read/write the bitmap to the allocator region. Support allocate and free for contiguous page ranges.

Files: new `allocator.rs`, [storage.rs](truthdb/crates/truthdb-core/src/storage.rs)

### Phase 3: Engine State Serialization

Add `serialize` and `deserialize` methods for `EngineState`. Use bincode. Exclude postings. Include a version tag and checksum in the serialized format.

Files: [engine.rs](truthdb/crates/truthdb-core/src/engine.rs)

### Phase 4: Snapshot Write Path

Implement the checkpoint write sequence: serialize state, allocate data region pages, write data, write snapshot descriptor, update superblock, advance `wal_head`.

Files: [storage.rs](truthdb/crates/truthdb-core/src/storage.rs), [engine.rs](truthdb/crates/truthdb-core/src/engine.rs)

### Phase 5: Modified Startup / Recovery

Change `Engine::new()` to check for a valid snapshot, load it, then replay only the remaining WAL entries. Rebuild postings after loading.

Files: [engine.rs](truthdb/crates/truthdb-core/src/engine.rs), [storage.rs](truthdb/crates/truthdb-core/src/storage.rs)

### Phase 6: WAL Reclamation

After a successful checkpoint, advance `wal_head` in the superblock. Verify that free space calculation and ring wrap logic work correctly with the new head position.

Files: [storage.rs](truthdb/crates/truthdb-core/src/storage.rs)

### Phase 7: Checkpoint Triggers

Add WAL usage threshold trigger and manual checkpoint command. Wire the trigger into the engine's write path.

Files: [engine.rs](truthdb/crates/truthdb-core/src/engine.rs), [dispatcher.rs](truthdb/crates/truthdb-core/src/dispatcher.rs)

### Phase 8: Integration Tests

- Write until WAL is nearly full, trigger checkpoint, verify writes can continue
- Crash simulation: kill mid-checkpoint, verify recovery uses previous valid checkpoint
- Startup with checkpoint: verify state matches, verify only post-checkpoint WAL entries are replayed
- Round-trip: write → checkpoint → more writes → restart → verify all data present
- Ring wrap: fill WAL, checkpoint, reclaim, fill again past the original tail position

## Potential Challenges

- **Alignment discipline**: all data region writes must be page-aligned (4 KB) because of `O_DIRECT`. The serialized blob will need padding.
- **Checkpoint size**: if engine state grows larger than the data region, the system must fail clearly rather than corrupt data. Need a size check before attempting a checkpoint.
- **Atomicity of the superblock flip**: the entire correctness model depends on the superblock write being atomic at the page level. A single 4 KB page write with `O_DIRECT` and `fdatasync` should provide this on Linux, but it must be validated.
- **Postings rebuild cost**: rebuilding postings from all documents on startup adds time. For v1 this is acceptable. Later, postings could be serialized too.
- **Concurrent reads during checkpoint**: the current global mutex means checkpoint serialization blocks all reads. For v1 this is acceptable. Later, a read snapshot or COW mechanism could allow concurrent access.

## Success Criteria

1. TruthDB can run indefinitely under sustained write load without hitting `WalFull`
2. Checkpoints are crash-safe — a crash mid-checkpoint does not lose previously committed data
3. Startup time is proportional to WAL entries *after* the last checkpoint, not total history
4. All existing tests continue to pass
5. The system can survive a fill → checkpoint → fill → checkpoint cycle repeatedly

## Non-Goals For V1

- Incremental / delta checkpoints (full state each time is fine for v1)
- Concurrent checkpoint writes (global mutex is acceptable)
- Postings serialization (rebuild from documents is acceptable)
- Compaction or garbage collection within the data region
- Multiple checkpoint history (only the latest checkpoint matters)
- Configurable serialization format (bincode only for v1)

## Open Questions

1. Should the checkpoint command be exposed as a client-facing API, or only triggered internally?
2. What WAL usage percentage should be the default trigger threshold?
3. Should we add a `--no-checkpoint` startup flag for debugging/testing?
4. Should the serialized format include a schema version migration path from the start, or defer that?
