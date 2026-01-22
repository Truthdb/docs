# TruthDB — Start Here (Org Overview)

This document is the high-level map of the TruthDB organization: what each repository does, how releases are produced, and how the installer ISO is assembled.

If you only read one doc to get productive quickly, read this first.

## Repos at a glance

| Repo                              | Purpose                                                                                 | Primary outputs                                                       |
| --------------------------------- | --------------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| `truthdb/`                        | Main TruthDB service (Tokio app) + systemd unit                                         | `truthdb` binary + `truthdb.service` release asset                    |
| `installer/`                      | Initramfs installer app (Rust, console-only installer that partitions/formats/installs) | `truthdb-installer` (x86_64 musl) release asset                       |
| `installer-kernel/`               | Linux kernel config used for the installer boot environment                             | `BOOTX64.EFI` (kernel bzImage built with EFI stub) release asset      |
| `installer-kernel-builder-image/` | Container image used to build the installer kernel reproducibly in CI                   | GHCR image `ghcr.io/truthdb/truthdb-installer-kernel-builder-image:*` |
| `installer-iso/`                  | Produces the bootable installer ISO and embeds an offline Debian payload                | `truthdb-installer-vX.Y.Z.iso` release asset                          |
| `orchestrator/`                   | Admin/developer CLI for TruthDB org (automates tagging + waits for release assets)      | `orchestrator` release asset                                          |
| `website/`                        | Public website (Vue + Vite)                                                             | `dist/` tarball release asset                                         |
| `.github/`                        | Org-level GitHub configuration and shared docs/specs                                    | Community health docs + specs                                         |

## How the installer ISO actually boots and installs

### Boot chain (current design)

1. **UEFI firmware** boots the ISO.
2. The ISO contains a **UKI (Unified Kernel Image)** written as `EFI/BOOT/BOOTX64.EFI` inside an ESP FAT image (`efi.img`).
3. That UKI contains:
   - the Linux kernel built from `installer-kernel/` config (`BOOTX64.EFI` in kernel releases)
   - an initramfs built by `installer-iso` release workflow
4. `busybox init` starts `truthdb-installer` (the installer app).
5. The installer app:
   - enumerates disks
   - partitions GPT (ESP + root)
   - formats vfat/ext4
   - mounts `/mnt` and `/mnt/boot/efi`
   - extracts an offline Debian payload (a prebuilt tar.zst)
   - configures hostname/users/networking
   - installs and configures `systemd-boot`
   - reboots

### Installer UX note (current)

The current `truthdb-installer` implementation is **console-only** and includes a confirmation prompt before destructive steps. The long-term goal is an unattended install, but today you should expect at least one `Press ENTER to continue` prompt.

### Offline Debian payload

`installer-iso` builds an offline Debian (bookworm) root filesystem payload during its **release workflow** using `debootstrap`. It then embeds that payload into the initramfs at `/payload/debian-minbase-amd64-bookworm.tar.zst`.

The payload includes:
- `linux-image-amd64` + initramfs tooling
- core userspace (`systemd-sysv`, `iproute2`, etc.)
- the TruthDB binary + `truthdb.service`, enabled for `multi-user.target`

See also: `specs/INSTALL-DEBIAN.md`.

## Versioning and releases (important)

Most repos publish releases when you push a Git tag like `vX.Y.Z`.

### Version locking

The `installer-iso` release workflow enforces *same-version* inputs:
- It downloads `truthdb` release `vX.Y.Z` and embeds it into the Debian payload.
- It downloads `installer` release `vX.Y.Z` as the initramfs installer binary.
- It downloads `installer-kernel` release `vX.Y.Z` as the kernel input for the UKI.

If any matching release is missing, the ISO build fails rather than mixing versions.

## Pipelines (what runs in CI vs what’s just for local use)

### `installer-iso`
- CI workflow builds a smoke-test ISO using the **latest** released kernel + installer (and uses placeholders if releases are missing).
- Release workflow (tagged) does the “real” work:
  - builds the Debian payload via `debootstrap` (bookworm/amd64)
  - embeds TruthDB runtime artifacts from the matching `truthdb` release
  - builds initramfs including required host-install tools (`wipefs`, `sfdisk`, `mkfs.*`, `tar`, `zstd`, `efibootmgr`, `bootctl` and systemd-boot EFI binaries)
  - builds a UKI using `ukify`
  - generates an ISO via `xorriso`

Key file: `.github/workflows/release.yml`

Note: `build_and_run.sh` is primarily a local reproduction helper; the authoritative build steps for releases are in the workflow.

### `installer`
- CI: `fmt`, `clippy`, `cargo test` (musl), builds the musl release binary.
- Release: packages `truthdb-installer` into `truthdb-installer-vX.Y.Z-x86_64-linux-musl.tar.gz` with a checksum.

Key files: `.github/workflows/ci.yml`, `.github/workflows/release.yml`

### `installer-kernel`
- CI: validates the config and builds kernel artifacts.
- Release: builds the kernel inside the kernel builder image and uploads `BOOTX64.EFI`.

Key files: `.github/workflows/release.yml`, `truthdb-installer-kernel.config`

### `installer-kernel-builder-image`
- CI: hadolint + build test.
- Release: builds and pushes a multi-arch image to GHCR.

Key file: `docker/Dockerfile`

### `truthdb`
- CI: fmt/clippy/test/build.
- Release: packages `truthdb` + `truthdb.service` into `truthdb-vX.Y.Z-x86_64-linux-gnu.tar.gz` and uploads checksum.

Key file: `.github/workflows/release.yml`

### `orchestrator`
- CI/release exists; binary currently prints help/usage and has minimal structure.

### `website`
- CI: `npm ci`, lint, build.
- Release: packages `dist/` as `website-vX.Y.Z.tar.gz` and uploads checksum.

## Installer internals (what runs in initramfs)

The installer app is designed to run in a minimal initramfs environment.

- UI/input: currently console-only with blocking stdin prompts.
- Disk safety policy: refuses to choose automatically if more than one eligible disk exists.

The installer executes external tools directly (no shell), so the initramfs must include all required programs and (if dynamically linked) their shared libraries. The `installer-iso` **release workflow** assembles the initramfs and is the authoritative place to verify tooling.

Key code:
- `installer/src/main.rs`
- `installer/src/platform/disks.rs`
- `installer/src/platform/partition.rs`
- `installer/src/platform/install.rs`

## Developing locally

### Build TruthDB service

```bash
cd truthdb
cargo build --release
```

### Build installer (musl)

```bash
cd installer
rustup target add x86_64-unknown-linux-musl
cargo build --release --target x86_64-unknown-linux-musl
```

### Build ISO locally (developer workflow)

The most accurate “build like releases” reference is `installer-iso/.github/workflows/release.yml`.

For local experimentation, `installer-iso/build_and_run.sh` and `installer-iso/run_container.sh` can be used, but they may not include all steps from the release workflow.

## Release checklist (recommended)

TruthDB releases are tag-driven: pushing a tag like `vX.Y.Z` triggers release workflows in most repos.

### What must match versions

`installer-iso` release builds are intentionally strict and require matching tags to exist:
- `Truthdb/truthdb` tag `vX.Y.Z` (provides `truthdb-vX.Y.Z-x86_64-linux-gnu.tar.gz`)
- `Truthdb/installer` tag `vX.Y.Z` (provides `truthdb-installer-vX.Y.Z-x86_64-linux-musl.tar.gz`)
- `Truthdb/installer-kernel` tag `vX.Y.Z` (provides `BOOTX64.EFI`)

If any of those are missing, `installer-iso` refuses to build a mixed-version ISO.

### Recommended tag order (to ship a bootable ISO)

1. **Build/publish the kernel builder image (optional but recommended when changed)**
   - Repo: `installer-kernel-builder-image`
   - Tag: `vX.Y.Z` (or any tag you use for changes)
   - Outcome: updates `ghcr.io/truthdb/truthdb-installer-kernel-builder-image:latest`
   - Reason: `installer-kernel` release uses the GHCR image `:latest` as its build container.

2. **Release the installer kernel**
   - Repo: `installer-kernel`
   - Tag: `vX.Y.Z`
   - Verify release assets include: `BOOTX64.EFI`

3. **Release the TruthDB service**
   - Repo: `truthdb`
   - Tag: `vX.Y.Z`
   - Verify release assets include:
     - `truthdb-vX.Y.Z-x86_64-linux-gnu.tar.gz` (contains `truthdb` + `truthdb.service`)
     - `truthdb-vX.Y.Z-x86_64-linux-gnu.sha256`

4. **Release the installer app**
   - Repo: `installer`
   - Tag: `vX.Y.Z`
   - Verify release assets include:
     - `truthdb-installer-vX.Y.Z-x86_64-linux-musl.tar.gz`
     - `truthdb-installer-vX.Y.Z-x86_64-linux-musl.sha256`

5. **Release the ISO**
   - Repo: `installer-iso`
   - Tag: `vX.Y.Z`
   - Verify release assets include:
     - `truthdb-installer-vX.Y.Z.iso`
     - `truthdb-installer-vX.Y.Z.iso.sha256`

6. **Optional, independent releases**
   - Repo: `orchestrator` tag `vX.Y.Z` (CLI binary)
   - Repo: `website` tag `vX.Y.Z` (static dist tarball)

### Post-release sanity checks

- **ISO integrity**: `sha256sum -c truthdb-installer-vX.Y.Z.iso.sha256`
- **Boot smoke test** (pick one):
  - UTM (macOS)
  - QEMU + OVMF (Linux)
  - Hyper-V Gen2 (Windows, UEFI)
- **Install outcome**:
  - Debian boots after reboot without the ISO.
  - `truthdb` service is enabled and starts under systemd.
  - DHCP brings up networking on first boot.


## Where to add new docs/specs

- Org-wide specs and cross-repo design notes: `.github/specs/`
- Repo-specific docs: keep them in the repo that owns the behavior.

## Pointers / known constraints

- The installer ISO is currently **UEFI-first** by design.
- Release workflows assume **amd64** for the Debian payload.
- `truthdb` and `orchestrator` are currently early-stage skeletons.
