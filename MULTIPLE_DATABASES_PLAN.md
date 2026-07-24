# Multiple Databases (Namespaces) — Plan

Status: in progress
Last updated: 2026-07-24

Databases as naming, security, and organizational containers over the one
shared log and data file — level 1 of the vision, deliberately nothing more.
One WAL, one data file, one allocator, one LSN space; restore, replication,
and failover stay instance-granular.

## Design

**The database entity.** A database is a catalog row: the existing catalog
B+ tree gains rows whose `database: Option<DatabaseDef>` payload is `Some`,
alongside tables, views, procedures, and principals (the established
kind-discriminator pattern). `DatabaseDef` carries `db_id: u32` and creation
metadata. Database ids are allocated max+1 over stored database rows,
starting at 2.

**The default database is synthesized, not stored.** Database id 1 is always
present, cannot be dropped, and takes its name from the existing
`[tds] database` config value (default `truthdb`) — exactly where the
session's database name comes from today. No bootstrap write on open, so
old data files and read-only standbys need nothing.

**Objects belong to a database.** `TableDef` gains
`database_id: u32` (serde default = 1, so every pre-existing catalog row
lands in the default database). The in-memory table map is re-keyed from
`name` to `(database_id, name)`; storage APIs gain the database dimension.
Uniqueness of object names is per database.

**Name resolution.** `db.dbo.t` resolves (three parts, middle must be `dbo`);
`dbo.t` and `t` resolve in the session's current database; four-part names
are rejected. Cross-database queries and DML work — it is all one catalog and
one log. Column-qualifier matching accepts the qualified spellings of the
same table.

**Sessions.** `TxnContext` gains `database_id`, resolved from the name at
session open. `USE <db>` becomes real: validated against the catalog,
mutates the session's current database, emits the ENVCHANGE with the
catalog's canonical casing. TDS login validates the requested database
(refused if it does not exist, SQL Server's 4060 behavior); an empty request
falls back to the default database. The native protocol's transient context
runs in the default database (today it runs with an empty string, which is
why `USE` always failed over the CLI).

**Lock-analysis seam.** Lock analysis re-resolves names before execution; a
mutating `USE` mid-batch could diverge the two derivations. Rule: lock
analysis tracks `USE` statements while walking the batch and resolves each
table under every database context that appears in the batch, taking the
union of lock ids. Over-locking is safe; under-locking is not.

**WAL container tag (vision groundwork).** The REL record envelope's
`flags: u16` field — written as zero and read by nothing since the format's
birth — becomes the database id of page-scoped records (PAGE_OP,
PAGE_IMAGE); 0 means global/legacy (transaction control, allocator, catalog
root, and all pre-existing logs). This changes no byte layout and needs no
version bump: old builds decode new logs unchanged, backups and standbys
ship the same raw bytes. Database ids are capped at u16::MAX at creation.

**Statements and errors** (SQL Server codes): `CREATE DATABASE` (1801 on
duplicate, 226 inside a transaction, privileged DDL, Database X lock);
`DROP DATABASE` (3701 nonexistent, 3702 for the session's current database,
3708 for the default database; drops every contained object in one
statement transaction); `USE` (911); login 4060; cross-database foreign
keys refused (1763). `sys.databases` returns real rows;
`DB_ID()`/`DB_ID(name)` implemented, `DB_NAME(id)` becomes
argument-sensitive. Permission-denied (229) and constraint messages stop
hardcoding `truthdb`.

## Deliberate deviations (naming-level scope)

- **Users, roles, and logins stay server-wide.** Object permissions become
  per-database automatically (they ride object rows), but principals are not
  scoped per database yet. dbo/db_owner bypass remains global.
- **Database options are instance-wide.** RCSI, snapshot isolation, and the
  recovery model are physical properties of the shared log and version
  machinery; `ALTER DATABASE <any> SET ...` affects the instance, and every
  `sys.databases` row reports the shared values.
- **`BACKUP DATABASE <name>` validates the name but backs up the instance**
  (and restore restores the instance) — level-1 granularity, per the vision;
  the backup contains all databases.
- **Dropping a database out from under another session's feet degrades
  gracefully** (their next statement errors) rather than killing their
  connection as SQL Server does.

## Slices

1. **A1 — catalog layer**: `DatabaseDef`, database rows, id allocation,
   synthesized default database, `(db_id, name)` re-keying of the table map
   and storage APIs, `TxnContext.database_id`. Behaviorally still
   single-database; everything existing stays green.
2. **A2 — SQL surface**: `CREATE/DROP DATABASE`, real `USE`, three-part
   resolution, `DB_ID`/`DB_NAME(id)`, real `sys.databases` rows, TDS login
   validation, native-path default database, error-message threading, lock
   analysis union rule.
3. **B — WAL tag**: database id into REL `flags` for page-scoped records.

Each slice: full `cargo test --workspace`, fmt/clippy clean, PR, adversarial
review, squash-merge.
