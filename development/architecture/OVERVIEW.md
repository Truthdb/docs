# TruthDB Architecture Overview

Status: draft (promoted from prior agreements)
Last updated: 2026-01-23

This document is the canonical high-level architecture baseline for TruthDB.

It separates:

- **Agreed direction**: decisions we intend to treat as “the plan” unless explicitly changed.
- **Current implementation**: what the repo does today.

If a section is aspirational, it is labeled as such.

## What TruthDB is (agreed direction)

TruthDB is a high-performance, WAL-centric, **hybrid data system** designed for:

- long-lived state
- deterministic replay
- strong durability guarantees
- predictable performance
- unified retrieval across structured, full-text, and vector data

It is inspired by systems like TigerBeetle and Kafka (log-first design), but is not a clone.

TruthDB is hybrid by design: a single system that serves as an event store, a search engine, a relational database, and a vector database. Rather than requiring separate infrastructure for each retrieval modality, TruthDB aims to support structured queries, full-text search, and vector similarity (for AI/RAG workloads) under one WAL, one consistency model, and one operational surface.

## Core principles (agreed direction)

- **Event-sourced at the core**: state is derived from an ordered event log.
- **WAL is the source of truth**: materialized state and indexes are rebuildable optimizations.
- **No hidden background magic**: durability boundaries and derivations are explicit.
- **Observable layers**: it should be possible to inspect and reason about durability, ordering, and replay.

## Storage model (agreed direction)

- Append-only segmented WAL
- Snapshots as an optimization, never the source of truth
- Deterministic replay from WAL bytes

Example conceptual layout:

- `/var/lib/truthdb/`
  - `wal/` (segments)
  - `snapshots/`
  - `metadata/` (manifest/schema)

The exact layout is a spec-level decision and should be aligned with the implementation before being treated as canonical.

## IO strategy (agreed direction; details TBD)

- Prefer sequential IO and explicit durability boundaries.
- Consider direct IO where practical.
- Avoid “accidental semantics” from generic async runtimes in the storage core.

See: `../decisions/ADR-0002-io-strategy.md`.

## Compute model (agreed direction)

- One primary executable with explicit roles selected by config (e.g. ingest, compute, query, replay).
- Deterministic processing requirements for core state derivation:
  - avoid wall-clock time inside deterministic replay paths
  - avoid hidden randomness

## WASM integration (agreed direction)

WASM is the intended plugin/sandbox/determinism boundary.

- What runs in WASM (intended): event processors/reducers/validators/custom operators.
- WASM must not have direct access to filesystem/network/clock.
- Access is via explicit host calls.

Status: aspirational (not fully implemented in current repo).

## Deployment model (agreed direction)

- Bare metal first (predictable IO, known hardware).
- Appliance-style installation path via a dedicated installer ISO.

## Installer and boot strategy (agreed direction vs current)

Agreed direction:

- Keep the UEFI stage minimal (“dumb loader”).
- The installer environment (Linux kernel + initramfs/userspace) owns UI, branding, and installation logic.
- Conservative disk safety posture and explicit confirmation before destructive actions.
- Quiet boot is a goal (no console spam; show a splash early).

Current implementation:

- The project ships a UEFI-bootable installer ISO built by the `installer-iso` pipeline.
- The installer is currently console-only (not a graphical UI).
- Local installer-iso builds now have explicit `dev` and `release` input modes, while tagged releases remain the authoritative path.

See:

- `../specs/INSTALL-DEBIAN.md`
- `../specs/INSTALLER-BOOT-AND-UX.md`

## Current implementation notes (what exists today)

TruthDB currently exists as:

- a Rust service (`truthdb/`) running under systemd
- an installer chain split across repos (`installer/`, `installer-kernel/`, `installer-iso/`)
- an org release pipeline that builds installer ISOs from the latest published dependency releases
- local containerized helper scripts that can either iterate on workspace artifacts or approximate the release input set

For authoritative “what happens during install”, prefer the build workflows in the code repos. Historical installer docs remain under `../specs/`.

## Vector and embedding support (agreed direction)

TruthDB should natively support dense vector embeddings as a first-class field type, with approximate nearest neighbor (ANN) indexing and similarity search. This enables RAG and AI-assisted retrieval without external vector databases.

Key design points:

- Vectors are stored in the WAL as part of document events, covered by the same durability guarantees.
- ANN indexes (e.g., HNSW) are derived structures, rebuildable from WAL replay — consistent with the principle that the WAL is the source of truth.
- Hybrid queries combine vector similarity, full-text relevance, and structured filters in a single request.
- TruthDB does not generate embeddings; embedding is the client's responsibility. TruthDB stores, indexes, and retrieves vectors.

Status: roadmap (not yet implemented). See: `../../features/VECTOR_DATABASE_AND_RAG.md`.

## Open questions (to turn into specs/ADRs)

- Exact WAL binary format (framing, commit boundaries, entry types)
- Snapshot format and lifecycle
- Replication protocol and what layer owns it
- Final on-disk layout and how it evolves
- What becomes part of product-facing guarantees vs internal implementation
- ANN index algorithm selection (HNSW vs DiskANN vs IVF) and integration with io_uring storage path
