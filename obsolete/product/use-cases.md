# TruthDB Use Cases

Status: draft
Last updated: 2026-01-23

This document collects candidate use cases for TruthDB.

It is intentionally broad; it should be refined into a small set of flagship scenarios as the product scope hardens.

## Core database and event-sourcing

### Application event store

- TruthDB acts as the backbone of an event-sourced system.
- Each aggregate is an ordered stream of events.
- Enables replay/time-travel and snapshotting for rebuilds.

### Append-only ledger / audit log

- Immutable, checksummed events.
- Strong auditability for regulated workflows.

### Time-series append store

- Continuous ingestion of telemetry.
- Append-only history (no rewrite/compaction assumptions in the core log).

## Systems and infrastructure

### Durable replication bus / changefeed

- Use partitions as replication logs between systems.
- Consumers can tail and replay deterministically.

### Local-first / edge durability

- Run on edge devices.
- Sync by shipping events later (delta/event-based replication).

## Domain-specific systems

### Financial core / payments ledger (platform-style)

- Transactions represented as atomic events.
- Intended to support strong durability and replay, while keeping domain logic above the WAL.

### IoT / industrial telemetry hub

- Gateways append raw readings.
- Consumers subscribe to event feeds.

## Development and research

### Deterministic recovery testing harness

- Crash recovery, replay ordering, checksum validation.
- Repeatable “power loss” simulations at the log boundary.

## Auditing and provenance

### Tamper-evident journal

- Strong corruption detection.
- Sealed segments + verification (future design).

## Next steps

- Pick 2–3 flagship scenarios.
- Define what is strictly guaranteed (ordering, durability modes, determinism) vs implementation detail.
