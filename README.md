# TruthDB Docs

This repository is the canonical home for TruthDB documentation and planning.

## Start here

- Development overview: `development/`
	- Architecture: `development/architecture/OVERVIEW.md`
	- Key specs:
		- `development/specs/TRUTHDB-START-HERE.md`
		- `development/specs/INSTALL-DEBIAN.md`
		- `development/specs/INSTALLER-BOOT-AND-UX.md`
		- `development/specs/WAL.md`
	- Decisions:
		- `development/decisions/ADR-0001-wal-is-source-of-truth.md`
		- `development/decisions/ADR-0002-io-strategy.md`
- Product documentation: `product/`
	- Use cases: `product/use-cases.md`
	- Concepts:
		- `product/concepts/how-truthdb-differs-from-postgres.md`

## Repo structure

- `development/`: engineering-facing docs for building, changing, and releasing TruthDB.
	- Use this for: architecture notes, interfaces/invariants, decisions (ADRs), runbooks, specs.
- `product/`: user-facing documentation.
	- Use this for: installation guides, user manuals, examples.
- `gptinput/`: raw input from external chats/LLMs.
	- This is not authoritative until it is summarized and promoted into `development/` or `product/`.

## Conventions

- Prefer small docs with a clear scope over one giant document.
- Make “reality vs plan” explicit:
	- **Current behavior** (descriptive): what the code/workflows do today.
	- **Target behavior** (normative): what we want to build.
- When you change a cross-repo invariant (e.g., release version-locking), update the relevant spec in `development/specs/`.