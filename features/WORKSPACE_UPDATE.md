# Workspace Update

## Status
Implemented.

## Goal
Describe a new `orchestrator workspace-update` feature that keeps a TruthDB developer workspace present, consistent, and easy to enter.

## Problem
Today the workspace is too manual.

A developer has to:

- clone each repo by hand
- keep workspace-level files in sync manually
- remember where `orchestrator` lives in order to run it

That creates friction and pushes workspace conventions into tribal knowledge.

## Proposed Feature
Add an `orchestrator` command named:

```sh
orchestrator workspace-update
```

The command should manage the local TruthDB workspace in three areas:

1. repo presence
2. workspace-level files
3. `orchestrator` availability in the workspace root

The intended outcome is that a developer can run one command regularly and get a correct workspace layout without manually chasing missing repos or copied helper files.

## User Experience
Desired usage:

```sh
orchestrator workspace-update
```

Expected behavior:

- if known TruthDB repos are missing from the workspace, clone them
- if workspace-level files are outdated, refresh them from a canonical source
- if the workspace-local `orchestrator` command is missing or stale, update it
- leave the user with a ready-to-use workspace root

The long-term convenience goal is that the workspace itself contains the up-to-date `orchestrator` entry point, so the user does not need to remember where some original clone lives.

## Technical Design

### Workspace model

Assume one workspace root directory containing multiple sibling repos, for example:

- `truthdb/`
- `docs/`
- `orchestrator/`
- `installer/`
- `installer-iso/`
- `installer-kernel/`
- `installer-kernel-builder-image/`
- `website/`

`workspace-update` should treat that directory as the canonical local workspace.

### Repo bootstrap

`orchestrator` should have an explicit workspace manifest describing which repos belong in a normal TruthDB workspace.

That manifest should be the source of truth for workspace composition.

GitHub should be used to validate and clone those repos, not to decide that every repo in the organization belongs in every workspace.

For each expected repo:

- if the directory exists, leave it alone
- if the directory does not exist, clone it into the workspace root

First-version behavior should be conservative:

- clone missing repos
- do not auto-pull existing repos
- do not change branches
- do not overwrite local work

That keeps the command safe enough to run regularly.

### Repo manifest

The workspace repo set should come from a manifest file or equivalent internal definition, not from enumerating the whole GitHub organization.

Reasoning:

- the GitHub org may contain repos that do not belong in every developer workspace
- future internal, archived, experimental, or infrastructure repos should not automatically appear in all workspaces
- workspace composition is a product decision, not just an org listing

GitHub can still be used for:

- validating that a listed repo exists
- determining clone URLs
- checking metadata such as default branch if needed later

But the repo list itself should remain explicit.

### Workspace files

The `orchestrator` repo should contain a directory that acts as the canonical source for workspace-level files, for example:

```text
orchestrator/workspace/
```

That directory would contain files that belong to the workspace root rather than any individual repo, such as:

- `.vscode/` settings
- `oc.sh`
- workspace `.code-workspace` files
- helper scripts
- shared workspace README or notes

`workspace-update` should sync those files into the actual workspace root.

The sync behavior should be explicit and predictable:

- files in `orchestrator/workspace/` are the source of truth
- matching files in the workspace root are copied or updated from that source
- the command should report what it changed

### Orchestrator self-install/update

The workspace should contain an up-to-date `orchestrator` entry point in a predictable place so the user does not need to remember where the original repo checkout lives.

The command should therefore also install or refresh a workspace-local launcher.

In practice, that launcher should live under:

```text
workspace/.bin/orchestrator
```

with a copied executable alongside it, for example:

```text
workspace/.bin/.orchestrator-bin
```

This location avoids a naming conflict with the `workspace/orchestrator/` repo directory.

Conceptually:

- `orchestrator` checks whether the workspace-local launcher is present
- if missing or outdated, it copies or regenerates it
- the workspace then has a stable local way to run `orchestrator`

The first implementation uses:

- a copied executable
- a thin wrapper script

That keeps the command easy to invoke while still letting `workspace-update` refresh the installed copy from the currently running executable.

### Update detection

The command should determine whether the workspace-local launcher is current relative to the canonical version in the `orchestrator` repo.

Possible approaches:

- compare file contents
- compare embedded version/hash metadata
- always overwrite

The implemented version uses:

- manifest file under `orchestrator/workspace/repos.toml`
- content-based update for synced workspace files
- launcher install under `.bin/`
- conservative repo handling for existing repos

### Reporting

The command should print a concise summary such as:

- cloned repos
- updated workspace files
- updated workspace-local `orchestrator`
- no-op items that were already current

## Constraints

- Must be safe to run regularly.
- Must not overwrite user changes inside existing repos.
- Must make missing-repo bootstrap easy.
- Must make workspace-level files easy to standardize.
- Must make `orchestrator` easy to run from the workspace root.

## Non-Goals

The first version should not:

- auto-pull or auto-merge existing repos
- switch branches automatically
- reset dirty working trees
- manage per-repo release state
- solve every local environment issue

## Open Questions
1. Should `workspace-update` also verify that each existing repo remote points at the expected GitHub origin?
2. Should the command eventually support optional pull/update behavior for clean repos?
3. Should the workspace launcher later also support a shorter alias such as `oc`?
