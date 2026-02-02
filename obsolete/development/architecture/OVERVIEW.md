# TruthDB Architecture Overview

Status: draft (promoted from prior agreements)
Last updated: 2026-01-23

This document is the canonical high-level architecture baseline for TruthDB.

It separates:

- **Agreed direction**: decisions we intend to treat as “the plan” unless explicitly changed.
- **Current implementation**: what the repo does today.

If a section is aspirational, it is labeled as such.

## What TruthDB is (agreed direction)

TruthDB is a high-performance, WAL-centric, event-sourced data system designed for:

- long-lived state
- deterministic replay
- strong durability guarantees
- predictable performance

It is inspired by systems like TigerBeetle and Kafka (log-first design), but is not a clone.

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

See: `development/decisions/ADR-0002-io-strategy.md`.

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

See:

- `development/specs/INSTALL-DEBIAN.md`
- `development/specs/INSTALLER-BOOT-AND-UX.md`

## Current implementation notes (what exists today)

TruthDB currently exists as:

- a Rust service (`truthdb/`) running under systemd
- an installer chain split across repos (`installer/`, `installer-kernel/`, `installer-iso/`)
- an org release pipeline that version-locks ISO inputs by tag

For authoritative “what happens during install”, prefer the specs under `development/specs/` and the build workflows in the code repos.

## Open questions (to turn into specs/ADRs)

- Exact WAL binary format (framing, commit boundaries, entry types)
- Snapshot format and lifecycle
- Replication protocol and what layer owns it
- Final on-disk layout and how it evolves
- What becomes part of product-facing guarantees vs internal implementation
