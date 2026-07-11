# Benchmarking

## Status
Planned.

## Goal

Give TruthDB a repeatable, end-to-end benchmark tool that measures real performance over the wire. The tool should be easy to run, produce clear numbers, and expose bottlenecks.

## Problem

There is no way to measure TruthDB performance today. Without numbers there is no way to know whether a change makes things faster or slower, no way to find bottlenecks, and no way to make credible claims about the system.

Now that snapshots and WAL reclamation are implemented, the system can sustain indefinite write load. Benchmarking is the logical next step.

## Proposed Feature

Add a new crate `truthdb-bench` that acts as a load generator against a running TruthDB server.

```sh
truthdb-bench --host 127.0.0.1 --port 9623 --operations 100000 --connections 4
```

Add a Docker entrypoint that starts the server and runs the benchmark in one command:

```sh
docker build -f Dockerfile.bench -t truthdb-bench .
docker run --rm --security-opt seccomp=unconfined truthdb-bench
```

## What It Measures

### Write throughput
- Insert documents as fast as possible
- Report: total ops, elapsed time, ops/sec

### Write latency
- Per-request round-trip time (send CommandReq → receive CommandResp)
- Report: min, p50, p95, p99, p999, max

### Read throughput and latency
- Run search queries against indexed data
- Same metrics as write

### Checkpoint impact
- Track latency during and around checkpoint events
- Show whether checkpoints cause visible latency spikes

### Startup time
- Time from `Engine::new()` to ready
- Compare: cold start (no snapshot, full WAL replay) vs warm start (snapshot + short WAL replay)
- This is measured separately from the network benchmark

## Design

### Architecture

```
┌──────────────┐       TCP        ┌──────────────┐
│ truthdb-bench │ ──────────────► │   truthdb    │
│  (N conns)   │ ◄────────────── │   server     │
└──────────────┘                  └──────────────┘
```

The benchmark tool is a separate binary that connects to TruthDB over TCP using the existing protocol (`truthdb-proto` / `truthdb-net`). It does not link against `truthdb-core` and does not bypass the network path. This measures the real end-to-end performance a user would see.

### Crate structure

New workspace member: `crates/truthdb-bench`

Dependencies:
- `truthdb-proto` (message types, encoding)
- `truthdb-net` (frame read/write)
- `tokio` (async runtime, TCP)
- `clap` (CLI arguments)

No additional external dependencies for metrics. Latency tracking uses a simple sorted-vec approach.

### Benchmark phases

The benchmark runs in sequential phases:

**Phase 1: Setup**
- Connect all TCP connections to the server
- Handshake (HelloReq/HelloResp) on each connection
- Create a benchmark index with a fixed schema:
  ```
  create index bench {
    "mappings": {
      "properties": {
        "title": { "type": "text" },
        "category": { "type": "keyword" },
        "score": { "type": "float" }
      }
    }
  }
  ```

**Phase 2: Write benchmark**
- Each connection runs a tight loop sending insert commands
- Documents are generated with deterministic but varied content
- Each connection tracks its own latency samples
- Runs until the configured operation count is reached
- At the end, aggregate results across all connections

**Phase 3: Read benchmark**
- Each connection runs search queries against the data inserted in phase 2
- Mix of match queries (text search) and term queries (keyword exact match)
- Same latency tracking as writes

**Phase 4: Report**
- Print a summary table with all metrics
- Example output:
  ```
  === TruthDB Benchmark Results ===

  Write phase:
    operations:  100000
    duration:    12.34s
    throughput:  8103 ops/sec
    latency:
      min:   0.08ms
      p50:   0.12ms
      p95:   0.19ms
      p99:   0.31ms
      p999:  1.24ms
      max:   8.71ms

  Read phase:
    operations:  100000
    duration:    4.56s
    throughput:  21929 ops/sec
    latency:
      min:   0.03ms
      p50:   0.04ms
      p95:   0.07ms
      p99:   0.11ms
      p999:  0.34ms
      max:   2.15ms

  Connections: 4
  Errors: 0
  ```

### CLI arguments

| Argument | Default | Description |
|---|---|---|
| `--host` | `127.0.0.1` | Server host |
| `--port` | `9623` | Server port |
| `--operations` | `100000` | Total operations per phase |
| `--connections` | `1` | Number of concurrent TCP connections |
| `--write-only` | `false` | Skip the read phase |
| `--read-only` | `false` | Skip the write phase (assumes data exists) |
| `--payload-size` | `medium` | Document size: `small`, `medium`, `large` |

### Document generation

Documents are generated deterministically from an operation counter. No randomness needed — deterministic means reproducible.

Small (~100 bytes):
```json
{"title": "item 42", "category": "cat-2", "score": 4.2}
```

Medium (~500 bytes):
```json
{"title": "benchmark document number forty two with additional descriptive text for realistic sizing", "category": "category-02", "score": 42.0}
```

Large (~2 KB):
```json
{"title": "long text field with many words ...", "category": "category-042", "score": 420.0}
```

### Latency tracking

Each connection maintains a `Vec<f64>` of per-request latencies in milliseconds. After the phase completes, the vec is sorted and percentiles are extracted by index. No external histogram library needed.

For large operation counts, reservoir sampling can cap the vec at a fixed size (e.g., 100,000 samples). For v1, just collect all samples — 100K f64 values is under 1 MB.

### Concurrency model

Each connection is a Tokio task running its own send/receive loop. A shared atomic counter tracks total completed operations. When the counter reaches the target, all tasks stop.

Work distribution is simple: each connection pulls the next operation number from the atomic counter. No pre-partitioning needed.

## Docker Integration

### Dockerfile.bench

```dockerfile
FROM rust:1-bookworm AS builder
WORKDIR /src
COPY Cargo.toml Cargo.lock ./
COPY src ./src
COPY config ./config
COPY crates ./crates
RUN cargo build --release -p truthdb --bin truthdb \
  && cargo build --release -p truthdb-bench --bin truthdb-bench

FROM debian:bookworm-slim AS runtime
RUN apt-get update \
  && apt-get install -y --no-install-recommends netcat-openbsd \
  && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY --from=builder /src/target/release/truthdb /usr/local/bin/truthdb
COPY --from=builder /src/target/release/truthdb-bench /usr/local/bin/truthdb-bench
COPY docker-bench-entrypoint.sh /usr/local/bin/docker-bench-entrypoint.sh
RUN chmod 0755 /usr/local/bin/docker-bench-entrypoint.sh \
  && mkdir -p /data
ENV STATE_DIRECTORY=/data
ENTRYPOINT ["/usr/local/bin/docker-bench-entrypoint.sh"]
```

### docker-bench-entrypoint.sh

Same pattern as `docker-repl-entrypoint.sh`:
1. Start truthdb server in background
2. Wait for port 9623 to be ready
3. Run `truthdb-bench` with any arguments passed through
4. Print results
5. Shut down server

### Running

```sh
# Default benchmark
docker run --rm --security-opt seccomp=unconfined truthdb-bench

# Custom options
docker run --rm --security-opt seccomp=unconfined truthdb-bench \
  --operations 500000 --connections 8 --payload-size large
```

The `--security-opt seccomp=unconfined` is required because TruthDB uses io_uring.

## Known Bottlenecks To Expose

The benchmark should make these existing limitations visible in the numbers:

1. **Global mutex** — increasing `--connections` beyond 1 should show diminishing or zero throughput gains, proving the mutex is the bottleneck
2. **Per-write fsync** — every insert does a full fsync, capping write throughput to device IOPS
3. **No group commit** — each write is its own io_uring submission + completion cycle
4. **Checkpoint stalls** — at 75% WAL usage, a checkpoint serializes the entire engine state under the mutex

## Success Criteria

1. `truthdb-bench` builds and runs in Docker
2. Produces correct, reproducible throughput and latency numbers
3. Numbers are stable across runs (low variance on the same hardware)
4. Exposes the known bottlenecks clearly in the output
5. Easy to run — one Docker command

## Non-Goals For V1

- Comparison against other databases
- Automated regression detection
- Grafana/Prometheus integration
- Mixed read/write workloads (reads and writes run in separate phases)
- Cluster or multi-node benchmarks
- Custom query workloads
- Warmup phase tuning

## Open Questions

1. Should the benchmark also measure bulk insert (many documents in one command) if/when that's supported?
2. Should there be a `--duration` mode that runs for a fixed time instead of a fixed operation count?
3. Should the Docker image be published to a registry for easy access?
4. Should the report output a machine-readable format (JSON) in addition to the human-readable table?
