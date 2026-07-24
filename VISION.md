# TruthDB Vision

Status: agreed direction
Last updated: 2026-07-24

This is the story of TruthDB: what it is, the architecture that makes it possible,
and the decisions we have committed to. Everything else in this repository is
detail under this document. The previous generation of docs (now in `obsolete/`)
described the same direction in draft form; this document supersedes them.

## What TruthDB is

TruthDB is one system that does the jobs organizations currently deploy four or
five systems to do: a relational database (SQL Server-compatible), a search
engine (Elasticsearch heritage), an event streaming platform (Kafka's job), a
financial ledger (TigerBeetle's job), and domain verticals built on top of them
(temporal views for pension firms being the first).

The differentiator is not that one box is cheaper than five. It is the one
property a fleet of separate systems can never have: **a single ordered
timeline**. In TruthDB, a transaction can update a table, emit an event, and
move money in a ledger — and every engine, every subscriber, and every
point-in-time view sees those effects in one consistent order, because there is
exactly one log. Between separate systems, that consistency is a distributed
systems research problem. Inside TruthDB, it is a property of the architecture.

Hence the name. One source of truth.

## The architecture: a small substrate, parallel engines

TruthDB is a small, hardened substrate with independent engines above it. The
pattern is that of Meta's Delos and the Tango paper — multiple state machines
over one shared log — and cousin to FoundationDB's layers.

```
┌────────────┬──────────┬───────────┬──────────┬────────────────┐
│ Relational │  Search  │  Streams  │  Ledger  │   Verticals    │
│ (SQL/TDS)  │          │ (future)  │ (future) │ (temporal, …)  │
├────────────┴──────────┴───────────┴──────────┴────────────────┤
│                        THE SUBSTRATE                          │
│   one data file · one ring WAL · group commit · recovery      │
│   checkpoints · backup/PITR · physical replication · catalog  │
└───────────────────────────────────────────────────────────────┘
```

**The substrate** owns the single data file and its allocator, the ring WAL,
the commit protocol, crash recovery, checkpoints, backup, and replication. It
is built: stages 1–18 of the relational plan produced a substrate with
ARIES-style recovery, full and log backups with point-in-time restore, and
physical WAL-shipping replication with synchronous commit, readable standbys,
and manual failover.

**The engines** are parallel and independent. Each has its own data model, its
own query surface, and potentially its own wire protocol. The relational engine
speaks TDS to SSMS and JDBC. A future streams engine can speak the Kafka
protocol. A future ledger engine can speak TigerBeetle-style batches. Engines
do not know about each other, and no engine's data model is forced through
another's.

The payoff of this shape: **every new engine inherits durability, crash
recovery, backup, and replication for free.** An engine defines WAL entry types
and redo rules; the substrate does the rest. Physical replication ships bytes —
it replicated the search engine without a line of search-aware code, and it
will replicate streams and ledgers the same way.

This is not hypothetical. TruthDB already runs two engines over the shared log
today — the relational engine and the search engine — with distinct WAL entry
families, recovered and replicated by common machinery.

## The substrate contract

Experience with two engines taught us precisely where the boundary belongs.
The project's hardest defects all had one shape: **an invariant derived twice
on two sides of a boundary.** The substrate exists to derive the dangerous
invariants exactly once. Four things are therefore shared, and only these:

1. **The log and the data file.** One WAL, one file, one allocator, one LSN
   space. Every durable byte of every engine lives here.
2. **The commit protocol.** Transaction begin/commit/rollback records and group
   commit are substrate-level, even for engines that only ever use
   single-event transactions. Because there is one log, a commit record
   spanning engines is trivial — cross-engine atomicity (a table update and a
   ledger transfer committing together) stays available at near-zero cost,
   rather than being impossible to retrofit across per-engine commit systems.
3. **Truncation and checkpoint floors.** Each engine reports what it still
   needs from the log; the substrate computes the minimum and owns ring
   truncation and checkpointing. No engine keeps private durable state the
   checkpoint system cannot see — engine-private snapshots were where the
   hardest replication bugs lived, and they are prohibited going forward.
4. **Naming and enumeration.** One catalog knows that every object exists —
   tables, indexes, topics, ledgers — even though it knows nothing about their
   internals. Backup manifests, replication monitoring, security grants, and
   `DROP DATABASE` all need to enumerate the world once, not per engine.

Everything else is parallel: data models, query languages, planners, wire
protocols. Cross-engine query features (temporal views over tables, full-text
predicates in SQL) are deliberate per-pair integrations, added when a vertical
demands them — never a day-one architectural obligation.

## One timeline

The single log is not an implementation convenience; it is the product. Three
consequences, one per future engine:

- **Streams.** Kafka orders events only within a partition; cross-partition
  ordering is the pain point entire stream-processing stacks exist to paper
  over. TruthDB's change stream is a filtered subscription over one totally
  ordered WAL — ordering across all tables, topics, and databases is free and
  exact.
- **Ledger.** TigerBeetle's model — atomic linked events across accounts and
  ledgers, no two-phase commit — works precisely because there is one log.
  TruthDB has that log.
- **Temporal.** "The world as of 2019-03-31" is a single consistent cut across
  every table and engine, because one LSN space is one clock. For pension-firm
  audits, a bitemporal view that joins employment history, fund prices, and
  regulation tables is exactly consistent, not approximately.

## Databases are namespaces

TruthDB supports multiple databases as **naming, security, and organizational
containers** — and deliberately nothing more. A database groups objects of any
engine (tables, topics, ledgers, indexes), scopes name resolution
(`db.schema.object`, `USE`), carries options and permissions, and appears in
`sys.databases`. Cross-database queries and transactions work with zero extra
machinery, because underneath there is still one log and one catalog.

We examined the alternatives in depth. Per-database granularity comes in three
levels, each roughly ten times the cost of the previous:

| Level | Grants | Requires | Breaks |
|---|---|---|---|
| 1. Naming | namespaces, security, cross-db queries and transactions | catalog + session work | nothing |
| 2. Restore | restore one database to a point in time | per-db logs (SQL Server) or tagged-log filtered replay (Oracle Multitenant) | cross-db consistency at restore |
| 3. Failover | per-database replication and failover | per-db log streams, N× replication machinery | cross-db transaction atomicity (the SQL Server availability-group seam) |

TruthDB commits to level 1. Levels 2 and 3 are rejected, not deferred, because
the vision itself replaces their use cases:

- The reason to want per-database restore is "rewind `hr` to 14:02." A system
  with temporal views answers `AS OF 14:02` — query the past without rewinding
  anyone. The streams engine answers with replay. The ledger engine answers
  with compensating entries and would call physical rewind a bug. **The past
  is queryable, not restorable.**
- Per-database failover fragments the timeline that is the product. Failover
  is whole-instance: one consensus domain, one truth — the TigerBeetle
  position, and the one stage 18 already built.

For calibration: Postgres shares one WAL but walls databases off from each
other (no cross-database queries); SQL Server gives every database its own log
and pays with internal two-phase commit and a decade of cross-database/
availability-group incompatibility; Oracle spent enormous engineering on
pluggable databases over a shared log and still kept failover whole-instance.
Every system that split the recovery or failover domain finer than the
transaction domain bought a seam. TruthDB keeps the two domains identical, so
the seam cannot exist.

## Decisions this vision fixes

Rejected permanently:

- Per-database or per-engine logs. One WAL, one LSN space, forever.
- Per-database restore and per-database failover. Restore, replication, and
  failover are instance-granular.
- Engine-private durable state invisible to the substrate's checkpoint and
  truncation machinery.
- Forced unification of query surfaces. Engines are parallel; integrations are
  chosen per pair.

Required of future work:

- **WAL records carry a container/engine tag.** Filtered change-stream
  subscriptions ("subscribe to `hr`"), per-domain retention, and any future
  log-filtering feature all need it. It is cheap to carry from the start of
  multi-database work and expensive to retrofit into shipped log formats.
- **Log retention becomes tiered.** The ring WAL is the hot tier; archived log
  segments are the warm tier for streams retention and deep temporal history.
  The FULL-recovery-model log archiver (copy-out-before-truncate) is the seed
  of this mechanism, generalized with per-namespace logical retention policies
  over one physical log.
- **Isolation is resource governance, not storage separation.** Multiple
  engines and tenants on one substrate contend for the ring, memory, and I/O;
  the answer is admission control and scheduling (Oracle Resource Manager's
  road), never splitting the storage.
- **If scale-out beyond one box ever comes, the sharding axis is
  partition-shaped (by key), not database-shaped.** Databases are namespaces;
  they are not units of placement.

## Where we are, and the road

Built: the substrate, and the relational engine through stage 18 of
[RELATIONAL_DB_PLAN.md](RELATIONAL_DB_PLAN.md) — SQL surface with TDS/SSMS
compatibility, ACID transactions with row versioning, backup with PITR, and
replication/HA. The search engine runs as the second (legacy) engine and will
be converged with the relational surface (`CREATE FULLTEXT INDEX`,
`CONTAINS()`) per the plan's stage 19 outline.

Ahead, in no committed order: multiple databases as namespaces, the streams
engine, the ledger engine, the temporal vertical, and vector search for
AI/RAG workloads. Each will get its own plan document with the rigor of the
relational plan. Each starts from the same first question: *what are its WAL
entry types and redo rules?* — because that is all the substrate needs to know.
