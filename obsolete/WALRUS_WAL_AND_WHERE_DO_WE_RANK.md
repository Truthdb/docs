# Walrus WAL And Where Do We Rank

## Question 1

Ok, id like you to have a look at this reddit thread: <https://www.reddit.com/r/rust/comments/1o0hbtz/walrus_a_1_million_opssec_1_gbs_write_ahead_log/> and be ready to answer questions.

## Answer 1

I’ve reviewed it and I’m ready.

Main takeaways:

- The core dispute in the thread is not “is it fast?” but “what durability contract does it really provide?” The top criticism is that the original design acknowledged writes before durable completion and relied on `mmap`/background flush behavior in ways that could lose data silently on faults. Source: [Reddit thread](<https://www.reddit.com/r/rust/comments/1o0hbtz/walrus_a_1_million_opssec_1_gbs_write_ahead_log/>)
- The author explicitly conceded that, at that point, it was “fast first, durable second,” and said they planned fail-stop handling for flush errors plus an opt-in synchronous write mode and stronger on-disk checks. Source: [Reddit thread](<https://www.reddit.com/r/rust/comments/1o0hbtz/walrus_a_1_million_opssec_1_gbs_write_ahead_log/>)
- The repo now appears to have moved beyond that October 2025 post: the current README describes a broader distributed streaming system, `io_uring` support on Linux, dual storage backends, and explicit fsync modes, with the latest release shown as `v0.3.0` on November 27, 2025. Source: [GitHub repo](<https://github.com/nubskr/walrus>)

One important framing point: the Reddit discussion is mainly about the earlier WAL/durability claims, while the current repo README reflects a later, expanded system. So if you ask me about “Walrus,” I’ll distinguish between:

- the October 2025 Reddit/WAL debate
- the later repo state shown on GitHub

Sources:

- <https://www.reddit.com/r/rust/comments/1o0hbtz/walrus_a_1_million_opssec_1_gbs_write_ahead_log/>
- <https://github.com/nubskr/walrus>

## Question 2

In the thread, im thermaldouble. The user valarauca14 answers my question and has other points in the thread. How are we faring here in thruthdb compared to the thread's points and comments?

## Answer 2

Short version: compared to the Walrus thread, TruthDB is in a much better place on the biggest correctness traps, but it is not yet “finished storage-engine hardening.”

The biggest difference is that the main `valarauca14` critique does not hit us the same way, because we do not use `mmap` at all. TruthDB’s storage path is Linux-only, `io_uring`-based, uses `O_DIRECT`, and registers both the file and fixed buffers in [direct_io.rs](/Users/fgos/projects/truthdb/truthdb/crates/truthdb-core/src/direct_io.rs#L77) and [direct_io.rs](/Users/fgos/projects/truthdb/truthdb/crates/truthdb-core/src/direct_io.rs#L99). So the whole `mmap` / `SIGBUS` / `msync` / page-cache-sync mess from that Reddit exchange mostly does not apply to us. On your `thermaldouble` question specifically, we also ended up following the “do it yourself, not `tokio-uring`” direction: Tokio is used for networking, but the disk path is our own raw `io_uring` wrapper, not a general async-runtime storage integration. That said, the storage calls still run under a global engine mutex and `submit_and_wait(1)` blocks the Tokio worker thread in [client_listener.rs](/Users/fgos/projects/truthdb/truthdb/crates/truthdb-core/src/client_listener.rs#L53), [dispatcher.rs](/Users/fgos/projects/truthdb/truthdb/crates/truthdb-core/src/dispatcher.rs#L57), and [direct_io.rs](/Users/fgos/projects/truthdb/truthdb/crates/truthdb-core/src/direct_io.rs#L266), so we have avoided the worst runtime mismatch but not yet built the proper dedicated storage actor.

On durability semantics, we are also better than the Walrus version being criticized there. TruthDB does not ack a write and fsync later in the background. `Engine::execute()` persists first and only applies the event after that in [engine.rs](/Users/fgos/projects/truthdb/truthdb/crates/truthdb-core/src/engine.rs#L41), [engine.rs](/Users/fgos/projects/truthdb/truthdb/crates/truthdb-core/src/engine.rs#L88), and `append_wal_entry()` writes the WAL record, updates superblocks, and calls `sync_data()` before returning in [storage.rs](/Users/fgos/projects/truthdb/truthdb/crates/truthdb-core/src/storage.rs#L286). So the “background fsync silently losing acknowledged writes” complaint from the thread is not our current design.

Integrity-wise, we are ahead of that Walrus discussion too. We checksum the file header and both superblocks in [storage_layout.rs](/Users/fgos/projects/truthdb/truthdb/crates/truthdb-core/src/storage_layout.rs#L31), [storage_layout.rs](/Users/fgos/projects/truthdb/truthdb/crates/truthdb-core/src/storage_layout.rs#L181), and each WAL record has a header checksum, payload checksum, and matching footer redundancy in [storage_layout.rs](/Users/fgos/projects/truthdb/truthdb/crates/truthdb-core/src/storage_layout.rs#L280). Recovery and replay verify those checks before trusting entries in [storage.rs](/Users/fgos/projects/truthdb/truthdb/crates/truthdb-core/src/storage.rs#L363), [storage.rs](/Users/fgos/projects/truthdb/truthdb/crates/truthdb-core/src/storage.rs#L393), and [storage.rs](/Users/fgos/projects/truthdb/truthdb/crates/truthdb-core/src/storage.rs#L481). We also use `xxHash64`, not FNV-1a, in [storage_layout.rs](/Users/fgos/projects/truthdb/truthdb/crates/truthdb-core/src/storage_layout.rs#L399). That is much closer to the checksum direction suggested in the thread.

Where we are still weak:

- We do not currently turn a storage sync failure into fail-stop behavior. We propagate the error back to the caller, but we do not latch the engine into a poisoned/read-only state or crash deliberately. That is an inference from [storage.rs](/Users/fgos/projects/truthdb/truthdb/crates/truthdb-core/src/storage.rs#L286) and [engine.rs](/Users/fgos/projects/truthdb/truthdb/crates/truthdb-core/src/engine.rs#L88): errors return, but nothing forces shutdown after a serious storage fault.
- Our “two superblocks” are not actually used as a robust dual-copy recovery scheme yet. On open, we require both superblocks to validate and then we take state from `superblock_a` in [storage.rs](/Users/fgos/projects/truthdb/truthdb/crates/truthdb-core/src/storage.rs#L165) and [storage.rs](/Users/fgos/projects/truthdb/truthdb/crates/truthdb-core/src/storage.rs#L186). That means the redundancy exists on disk, but recovery does not yet choose “latest valid generation.”
- We are not yet using the more advanced kernel/page-state tricks `valarauca14` mentioned (`sync_file_range`, `PAGE_IS_SOFT_DIRTY`, `mlock`). But that is mostly because our current design intentionally bypasses the page-cache path with `O_DIRECT`, not because we forgot they exist.
- Architecturally, we are still simple: one engine mutex, one-entry-at-a-time `io_uring` submissions, no batching/group commit/storage thread. That is correctness-friendlier than the original Walrus criticism, but not where a serious high-end engine ends up.

So the blunt answer is: from the Reddit thread’s perspective, TruthDB is already better than the criticized Walrus design on `mmap`, ack semantics, and basic on-disk validation. The biggest remaining gaps are post-I/O-failure policy and proper superblock recovery logic, not the `tokio-uring` question.

Source for the thread context: [Reddit thread](https://www.reddit.com/r/rust/comments/1o0hbtz/walrus_a_1_million_opssec_1_gbs_write_ahead_log/)
