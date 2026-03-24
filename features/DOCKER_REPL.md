# Docker REPL

## Status
Draft.

## Goal
Describe the Docker REPL feature we want, including the user experience, technical design, constraints, and open questions.

## Problem
We want a simple developer-facing way to start TruthDB in Docker and immediately interact with it through `truthdb-cli` without manually:

- building binaries
- wiring up a container
- starting the server
- waiting for it to accept connections
- attaching a client

The intended outcome is a single shell script that drops the user into an interactive TruthDB REPL backed by a running TruthDB server inside Docker.

## Proposed Feature
Add a shell script that launches a Docker container containing both:

- the `truthdb` server binary
- the `truthdb-cli` binary

The container should:

1. start `truthdb` in the background
2. wait until the server is accepting TCP connections
3. start `truthdb-cli` in REPL mode in the foreground
4. when the CLI exits, stop the background server cleanly

This is a developer convenience feature. It is not intended to model production deployment and should not attempt to run the existing `systemd` service definition inside Docker.

## User Experience
Desired UX:

```sh
./scripts/docker_repl.sh
```

Expected behavior:

- the script ensures the Docker image exists, or builds it
- the script starts a container interactively
- the container starts TruthDB
- once TruthDB is ready, the user lands directly in `truthdb-cli`
- the user can type commands and receive responses immediately
- when the user exits the REPL, the container exits too

Optional UX extensions we may add later:

- expose the TruthDB TCP port to the host
- accept script flags for rebuild, reset-data, port, or image tag
- support a one-shot command mode instead of interactive REPL only

## Technical Design
### High-level approach

Use a single container, not multiple containers.

Reasoning:

- `truthdb-cli` already speaks TCP to the server
- both binaries belong to the same codebase
- the simplest REPL workflow is to run the server and client in one container
- `docker compose` is unnecessary for the first version

### Container runtime model

The container should not try to run `systemd`.

Instead:

- run `truthdb` directly as a normal process
- keep `truthdb-cli repl` as the foreground interactive process
- use a small shell entrypoint to coordinate startup and shutdown

Conceptual container flow:

```sh
truthdb &
server_pid=$!

cleanup() {
  kill -TERM "$server_pid" 2>/dev/null || true
  wait "$server_pid" 2>/dev/null || true
}
trap cleanup EXIT INT TERM

until nc -z 127.0.0.1 9623; do
  sleep 0.1
done

truthdb-cli repl
```

### Image contents

The Docker image should contain:

- `truthdb`
- `truthdb-cli`
- a minimal shell environment for the entrypoint script
- a small readiness-check tool such as `nc`, unless we choose a shell-native or Rust-native readiness loop

### Storage

The container should mount a writable Docker volume for database state.

TruthDB already respects `STATE_DIRECTORY` for storage path resolution, so the container can set:

```sh
STATE_DIRECTORY=/data
```

and mount:

```sh
-v truthdb-repl-data:/data
```

This gives persistent state without requiring a config file for the first version.

### Persistence flags

The launcher script should support both persisted and ephemeral workflows.

Recommended interface:

```sh
./scripts/docker_repl.sh --persist
./scripts/docker_repl.sh --ephemeral
./scripts/docker_repl.sh --reset-data
```

Behavior:

- persisted mode uses a named Docker volume
- ephemeral mode uses disposable container-local storage
- `--reset-data` deletes the persistent named volume before launch

Default behavior:

- persisted mode should be the default

Reasoning:

- the REPL is more useful if prior state survives across sessions
- clean-slate experimentation is still available through `--ephemeral`
- destructive state reset should be explicit

### Networking

Inside the container:

- `truthdb` listens on `0.0.0.0:9623` by default
- `truthdb-cli` connects to `127.0.0.1:9623` by default

This works for the single-container REPL design without extra configuration.

Exposing `9623` to the host should be optional, not required for the basic REPL workflow.

### Host-side launcher

The feature should include a host-side shell script, likely:

```sh
./scripts/docker_repl.sh
```

Responsibilities:

- build or locate the Docker image
- launch the container with `--rm -it --init`
- default to `linux/amd64` so the Docker REPL matches the existing CI/release architecture
- mount persistent state volume
- pass `STATE_DIRECTORY=/data`
- optionally pass through extra flags later

Location:

- the launcher should live in the `orchestrator` repo so it is stored in git
- it should assume `truthdb/` is a sibling repo by default, with an override such as `TRUTHDB_DIR=/path/to/truthdb`

## Non-Goals

The first version should not:

- run the existing `truthdb.service` under `systemd` inside the container
- model a production deployment topology
- require Docker Compose
- add orchestration for multiple services
- implement remote access, auth, or multi-user session management

## Constraints
- Must be simple to run for developers.
- Must not require `systemd` inside Docker.
- Should work with the existing TruthDB server and CLI behavior instead of introducing a separate dev-only protocol path.
- Should shut down the background server cleanly when the REPL exits.
- Should support persisted state across runs.
- Should keep the initial implementation small and understandable.

## Open Questions
1. Should the script always rebuild the image, or only when explicitly asked?
2. Should the host-side script publish port `9623` to the host in the first version?
3. Should the script live at repo root, under `scripts/`, or inside the `truthdb/` workspace?
4. Should the Docker image be built from source every time, or should we also support using prebuilt release binaries later?
