# TruthDB Roadmap

Last updated: 2026-07-24

Status of the vision described in [VISION.md](VISION.md), and the shape of
what comes next. Legend: ✅ built and tested · 🟡 partially done · ❌ not
started. Items below the "Ahead" line are in no committed order; each will get
its own detailed plan document before work starts, with the rigor of
[RELATIONAL_DB_PLAN.md](RELATIONAL_DB_PLAN.md).

## Built

### The substrate

- ✅ Single-file storage on io_uring with an internal space allocator.
- ✅ Write-ahead log with group commit; the log is the source of truth, all
  other state is derived and rebuildable.
- ✅ ARIES-style crash recovery (redo/undo) and checkpoints.
- ✅ Backup: full backups, log backups, point-in-time restore, FULL/SIMPLE
  recovery models with copy-out-before-truncate log archiving.
- ✅ Physical replication: WAL shipping over TLS with authenticated handshake,
  replication slots, synchronous commit, readable standbys, standby
  restartpoints, and manual promotion with epoch (timeline) fencing.
- ✅ Replication monitoring: `sys.dm_repl_slots`, `sys.dm_repl_replica_states`,
  and a CLI status command.

### The relational engine

Detailed history and remaining items: [RELATIONAL_DB_PLAN.md](RELATIONAL_DB_PLAN.md).

- ✅ T-SQL surface: DDL, DML, joins, aggregates, subqueries, views,
  procedures, functions, triggers, with SQL Server-compatible error numbers.
- ✅ TDS wire protocol: SSMS query windows, JDBC, and other standard SQL
  Server clients work unmodified.
- ✅ Transactions: ACID with locking, deadlock detection, and row-versioned
  isolation (READ COMMITTED SNAPSHOT and SNAPSHOT) backed by a version store.
- ✅ Constraints with faithful error semantics (primary/unique keys, foreign
  keys, checks, defaults).
- ✅ Security: catalog-backed logins and users with hashed passwords,
  permissions.
- 🟡 A manual validation pass in SSMS (scripted checklist exists; needs a
  human at the GUI).
- 🟡 Long-tail SQL features, outlined but not planned in detail: cursors,
  savepoints, recursive CTEs, updatable views, cascading foreign keys,
  BULK INSERT, a cost-based optimizer with statistics.

### The search engine

- ✅ Full-text search with an Elasticsearch-style query DSL, running as a
  second engine over the shared log — the working proof of the
  parallel-engines architecture.

## Ahead

### Search convergence

- ❌ `CREATE FULLTEXT INDEX` and `CONTAINS()` over relational tables; retire
  the legacy standalone search surface.
- ❌ Vector similarity search for AI/RAG workloads, under the same catalog and
  log.

### Multiple databases (namespaces)

- ❌ `CREATE DATABASE` / `DROP DATABASE`, `USE`, `db.schema.object`
  resolution, per-database options and permissions, real rows in
  `sys.databases`.
- ❌ Container/engine tag on log records — required groundwork for filtered
  subscriptions and per-namespace retention (see the requirements in
  [VISION.md](VISION.md)).

### The streams engine

- ❌ Topics and durable subscriptions as filtered views over the shared log;
  change-data-capture from relational tables.
- ❌ Tiered log retention: the in-place log as hot tier, archived segments as
  warm tier (generalizing the recovery-model log archiver), per-namespace
  retention policies.
- ❌ Kafka wire-protocol compatibility, so existing producers and consumers
  connect unmodified.

### The ledger engine

- ❌ Double-entry accounts and transfers with TigerBeetle-style semantics:
  linked event chains committing atomically, domain-isolation rules enforced
  by the engine.
- ❌ Atomicity with the relational engine: a table update and a ledger
  transfer in one transaction.

### The temporal vertical

- ❌ `AS OF` queries: consistent point-in-time reads across tables and
  engines from one clock.
- ❌ Bitemporal views (valid time and transaction time) for pension
  administration and audit workloads.
- ❌ Deep-history retention built on the tiered log.
