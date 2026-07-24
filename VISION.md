# TruthDB Vision

Last updated: 2026-07-24

This document is the story of TruthDB: what it is, the architecture that makes
it possible, and the decisions it is committed to. It assumes no prior
knowledge of the project. Current status and the order of upcoming work live in
[ROADMAP.md](ROADMAP.md); everything else in this repository is detail under
this document.

## What TruthDB is

Most organizations that need a database end up running four or five: a
relational database for transactions, a search engine for full-text queries,
an event streaming platform to move changes between systems, sometimes a
specialized ledger for money, and glue to hold it all together. Each system
has its own storage, its own replication, its own backup story, and its own
idea of what happened when.

TruthDB is one system that does those jobs: a relational database compatible
with SQL Server's wire protocol and SQL dialect, a full-text search engine, an
event streaming platform, a double-entry ledger, and domain-specific features
built on top of them (time-dependent views for pension administration being
the first).

The point is not that one box is cheaper than five, although it is. The point
is a property that a fleet of separate systems cannot have at any price: **a
single ordered timeline**. In TruthDB, one transaction can update a table,
emit an event to a stream, and move money in a ledger — and every query, every
subscriber, and every point-in-time view sees those effects in the same order,
because there is exactly one write-ahead log and everything in the system is
derived from it. Keeping separate systems consistent with each other is a
distributed-systems research problem. Inside TruthDB it is a property of the
architecture.

Hence the name. One source of truth.

## Why one timeline matters

Three concrete examples of what the single log buys, one per major feature
area:

- **Streams.** Kafka guarantees event ordering only within one partition;
  ordering across partitions is the gap that entire stream-processing stacks
  exist to paper over. A TruthDB change stream is a filtered view of one
  totally ordered log, so ordering across all tables, topics, and databases is
  exact and free.
- **Ledger.** TigerBeetle (the reference design for modern accounting
  databases) commits chains of transfers atomically across accounts and
  ledgers without two-phase commit — which is possible precisely because it
  has a single log. TruthDB has that log, so ledger operations can be atomic
  with each other *and* with ordinary SQL updates in the same transaction.
- **Time travel.** "The world as of 2019-03-31" is a single consistent cut,
  because one log defines one clock for everything. A pension audit that joins
  employment history, fund prices, and regulation tables as of a past date
  gets an exactly consistent answer, not an approximately consistent one.

## The architecture: a small substrate, parallel engines

TruthDB is a small, hardened storage substrate with independent engines above
it:

```
┌────────────┬──────────┬───────────┬──────────┬────────────────┐
│ Relational │  Search  │  Streams  │  Ledger  │   Verticals    │
│ (SQL/TDS)  │          │           │          │ (temporal, …)  │
├────────────┴──────────┴───────────┴──────────┴────────────────┤
│                        THE SUBSTRATE                          │
│   one data file · one write-ahead log · commit protocol       │
│   crash recovery · checkpoints · backup · replication         │
└───────────────────────────────────────────────────────────────┘
```

**The substrate** owns the single data file and its space allocator, the
write-ahead log, the transaction commit protocol, crash recovery, checkpoints,
backup, and replication. It is deliberately small, and it is where all of the
system's durability and correctness machinery is concentrated.

**The engines** are parallel and independent. Each has its own data model, its
own query surface, and potentially its own wire protocol: the relational
engine speaks TDS (the SQL Server protocol) so that tools like SSMS and
standard JDBC drivers work unmodified; a streams engine can speak the Kafka
protocol; a ledger engine can speak TigerBeetle-style command batches. Engines
do not know about each other, and no engine's data model is forced through
another's.

This pattern — several independent state machines running over one shared,
replicated log — is established in the systems literature (Meta's Delos, the
Tango paper) and is a cousin of FoundationDB's layer architecture. Its payoff
is that **every engine inherits durability, crash recovery, backup, and
replication from the substrate for free.** A new engine defines its log record
types and how to replay them; the substrate does everything else. Replication
in particular ships raw log bytes, so it replicates every engine — current and
future — without engine-specific code.

## The substrate contract

The boundary between substrate and engines is precise, because getting it
wrong is how storage systems break. The recurring failure mode in this class
of system is an invariant that is computed independently in two places — two
components each deciding "what is committed," or "what may be deleted from the
log" — which eventually disagree. The substrate exists to compute the
dangerous invariants exactly once. Four things are shared, and only these:

1. **The log and the data file.** One write-ahead log, one data file, one
   allocator, one log-sequence-number space. Every durable byte of every
   engine lives here.
2. **The commit protocol.** Transaction begin, commit, and rollback are
   substrate-level, even for engines that mostly use single-event
   transactions. Because there is one log, a commit record spanning engines is
   trivial — atomicity between a table update and a ledger transfer comes at
   near-zero cost, rather than being impossible to retrofit across per-engine
   commit systems.
3. **Log truncation and checkpoint floors.** Each engine reports the oldest
   log position it still needs; the substrate computes the minimum and owns
   log truncation and checkpointing. No engine keeps private durable state
   that the checkpoint system cannot see.
4. **Naming and enumeration.** One catalog records that every object exists —
   tables, indexes, topics, ledgers — even though it knows nothing about their
   internals. Backup manifests, monitoring, security grants, and
   `DROP DATABASE` enumerate the world once, not once per engine.

Everything else is parallel: data models, query languages, planners, wire
protocols. Cross-engine query features — temporal views over relational
tables, full-text predicates inside SQL — are deliberate, per-pair
integrations added when a use case demands them, never an architectural
obligation.

## Databases are namespaces

TruthDB supports multiple databases as **naming, security, and organizational
containers** — and deliberately nothing more. A database groups objects of any
engine (tables, topics, ledgers, indexes), scopes name resolution
(`db.schema.object`, `USE`), carries options and permissions, and appears in
`sys.databases`. Queries and transactions across databases work with no extra
machinery, because underneath there is still one log and one catalog.

This is a considered position, not an omission. "Multiple databases" can mean
three levels of separation, each roughly ten times the cost of the previous:

| Level | Grants | Requires | Breaks |
|---|---|---|---|
| 1. Naming | namespaces, security, cross-db queries and transactions | catalog and session work | nothing |
| 2. Restore | restoring one database to a point in time while others run | a log per database (SQL Server) or filtered replay of a tagged shared log (Oracle Multitenant) | cross-database consistency at restore |
| 3. Failover | replicating and failing over each database independently | a log stream per database, replication machinery multiplied | cross-database transaction atomicity |

TruthDB commits to level 1 and rejects levels 2 and 3 — rejects, not defers —
because the rest of the vision replaces their use cases:

- The reason to want per-database restore is "put this database back to
  14:02." A system with temporal views answers `AS OF 14:02` — query the past
  without rewinding anyone. Streams answer with replay. A ledger answers with
  compensating entries, and would call physical rewind a bug. **The past is
  queryable, not restorable.**
- Per-database failover fragments the timeline that is the product. Failover
  is whole-instance: one consensus domain, one truth.

The industry's history calibrates the trade. Postgres shares one log but walls
databases off so completely that cross-database queries are impossible. SQL
Server gives every database its own log and pays with internal two-phase
commit and a decade in which cross-database transactions were unsupported
under its own high-availability feature. Oracle spent enormous engineering on
pluggable databases over a shared log and still kept failover whole-instance.
Every system that made the recovery or failover unit smaller than the
transaction unit bought a permanent seam. TruthDB keeps the two units
identical, so the seam cannot exist.

## Decisions this vision fixes

Rejected permanently:

- Per-database or per-engine logs. One write-ahead log, one LSN space.
- Per-database restore and per-database failover. Restore, replication, and
  failover are instance-granular.
- Engine-private durable state invisible to the substrate's checkpoint and
  truncation machinery.
- Forced unification of query surfaces. Engines are parallel; integrations
  are chosen per pair.

Required of future work:

- **Log records carry a container/engine tag**, so that filtered change-stream
  subscriptions ("subscribe to changes in `hr`"), per-domain retention, and
  any future log-filtering feature are possible. The tag is cheap to carry
  from the beginning and expensive to retrofit into a shipped log format.
- **Log retention is tiered.** The in-place log is the hot tier; archived log
  segments are the warm tier serving stream retention and deep temporal
  history, governed by per-namespace retention policies over one physical log.
- **Isolation is resource governance, not storage separation.** Engines and
  tenants sharing one substrate contend for log bandwidth, memory, and I/O;
  the answer is admission control and scheduling, never splitting the
  storage.
- **If scale-out beyond one machine ever comes, the sharding axis is
  partition-shaped (by key), not database-shaped.** Databases are namespaces,
  not units of placement.
