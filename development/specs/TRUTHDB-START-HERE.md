# TruthDB — Start Here (Org Overview)

This document is the high-level map of the TruthDB organization: what each repository does, how releases are produced, and how the installer ISO is assembled.

If you only read one doc to get productive quickly, read this first.

## Repos at a glance

| Repo                              | Purpose                                                                                 | Primary outputs                                                       |
| --------------------------------- | --------------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| `truthdb/`                        | Main TruthDB service (Tokio app) + systemd unit                                         | `truthdb` binary + `truthdb.service` release asset                    |
| `installer/`                      | Initramfs installer app (Rust, console-only installer that partitions/formats/installs) | `truthdb-installer` (x86_64 musl) release asset                       |
| `installer-kernel/`               | Linux kernel config used for the installer boot environment                             | `BOOTX64.EFI` (kernel bzImage built with EFI stub) release asset      |
| `installer-kernel-builder-image/` | Container image used to build the installer kernel in CI                                | GHCR image `ghcr.io/truthdb/truthdb-installer-kernel-builder-image:*` |
| `installer-iso/`                  | Produces the bootable installer ISO and embeds an offline Debian payload                | `truthdb-installer-vX.Y.Z.iso` release asset                          |
| `orchestrator/`                   | Admin/developer CLI for TruthDB org (automates tagging + waits for release assets)      | `orchestrator` release asset                                          |
| `website/`                        | Public website (Vue + Vite)                                                             | `dist/` tarball release asset                                         |
| `docs/`                           | Historical and supporting documentation                                                 | Markdown documentation                                                |
| `.github/`                        | Org-level GitHub configuration, templates, and community health docs                    | Templates + community health docs                                     |

## How the installer ISO actually boots and installs

### Boot chain (current design)

1. **Firmware** boots the ISO through `GRUB`.
2. `GRUB` loads:
   - the Linux installer kernel built from `installer-kernel/` config (currently fetched from the `BOOTX64.EFI` kernel release asset)
   - an initramfs built by `installer-iso` release workflow
3. The initramfs starts a small custom `/init` flow and launches `truthdb-installer`.
   - Current default path: installer runs on the active console with kernel log noise suppressed as much as possible.
   - Optional GRUB entry: dedicated-VT mode can still be selected explicitly for testing.
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

See also: `INSTALL-DEBIAN.md`.

## Versioning and releases (important)

Most repos publish releases when you push a Git tag like `vX.Y.Z`.

### Installer ISO dependency selection

The `installer-iso` release workflow uses the **latest published** releases from:
- `truthdb`
- `installer`
- `installer-kernel`

It does **not** version-lock those dependencies to the `installer-iso` tag. In practice, this means you should publish the desired dependency releases first and only tag `installer-iso` once those releases are available as the latest ones on GitHub.

## Pipelines (what runs in CI vs what’s just for local use)

### `installer-iso`
- CI workflow builds a smoke-test ISO using the **latest** released kernel + installer (and uses placeholders if releases are missing).
- Release workflow (tagged) does the “real” work:
  - builds the Debian payload via `debootstrap` (bookworm/amd64)
  - uses the shared `build_rootfs_payload.sh` helper in release mode to embed TruthDB runtime artifacts from the latest published `truthdb` release available at build time
  - downloads the latest published `installer` and `installer-kernel` artifacts available at build time
  - verifies the published checksums for `truthdb`, `installer`, and `installer-kernel` before embedding them
  - builds initramfs including required host-install tools (`wipefs`, `sfdisk`, `mkfs.*`, `tar`, `zstd`, `efibootmgr`, `bootctl` and systemd-boot EFI binaries)
  - assembles a `GRUB`-based installer ISO with normal, safe-graphics, debug, serial-console, and rescue entries
  - generates a BIOS+UEFI-capable ISO via `grub-mkrescue`

Key file: `.github/workflows/release.yml`

Note: local builds now have two explicit modes:
- `INPUT_MODE=dev`: build local `truthdb` + `installer`, build payload locally, and optionally override the kernel with `KERNEL_SRC`
- `INPUT_MODE=release`: use published `truthdb` + `installer` + `installer-kernel` artifacts while still running locally

The release workflow remains authoritative, but local `INPUT_MODE=release` now uses the same payload-builder (`build_rootfs_payload.sh`) and ISO assembler (`build_iso.sh`) as the tagged workflow.

### `installer`
- CI: `fmt`, `clippy`, `cargo test` (musl), builds the musl release binary.
- Release: packages `truthdb-installer` into `truthdb-installer-vX.Y.Z-x86_64-linux-musl.tar.gz` with a checksum.

Key files: `.github/workflows/ci.yml`, `.github/workflows/release.yml`

### `installer-kernel`
- CI: validates the config and builds kernel artifacts.
- Release: builds the kernel inside the kernel builder image and uploads `BOOTX64.EFI` plus `BOOTX64.EFI.sha256`.

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

### Runtime verification rule

For TruthDB runtime verification, smoke tests, and feature validation:

- prefer `linux/amd64` execution
- prefer Docker/containerized runs over host-native macOS runs
- do not treat host-native macOS execution as authoritative for TruthDB behavior

Reasoning:

- current CI and release artifacts target `x86_64` / `amd64`
- Docker gives a closer match to the current supported runtime than host-native macOS
- Linux-specific behavior must be validated in a Linux environment, not inferred from macOS

Practical rule for engineers and agents:

- if a task requires actually running `truthdb` or `truthdb-cli`, run them in `linux/amd64`
- use the Docker REPL or another Docker-based path when possible
- only use host-native execution for limited compile/test convenience, not for authoritative runtime verification

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

Recommended entry points:

- `installer-iso/build_in_container.sh`
  - default: `INPUT_MODE=dev`
  - release-like local build: `INPUT_MODE=release ./build_in_container.sh`
- `installer-kernel/build_iso_with_local_kernel.sh`
  - builds a local kernel first, then forwards into `installer-iso/build_in_container.sh`
  - supports `INPUT_MODE=dev` and `INPUT_MODE=release`

The most accurate local “build like releases” path is now `INPUT_MODE=release ./build_in_container.sh`, because it uses the same shared payload-builder and ISO-assembly scripts as the release workflow.

## Release checklist (recommended)

TruthDB releases are tag-driven: pushing a tag like `vX.Y.Z` triggers release workflows in most repos.

### What `installer-iso` consumes

`installer-iso` release builds consume whatever GitHub currently reports as the latest published releases for:
- `Truthdb/truthdb`
- `Truthdb/installer`
- `Truthdb/installer-kernel`

If you need a particular set of artifacts in the ISO, release those repos first and verify they are the latest published releases before tagging `installer-iso`.

### Recommended tag order (to ship a bootable ISO)

1. **Build/publish the kernel builder image (optional but recommended when changed)**
   - Repo: `installer-kernel-builder-image`
   - Tag: `vX.Y.Z` (or any tag you use for changes)
   - Outcome: updates `ghcr.io/truthdb/truthdb-installer-kernel-builder-image:latest`
   - Reason: `installer-kernel` release uses the GHCR image `:latest` as its build container.

2. **Release the installer kernel**
   - Repo: `installer-kernel`
   - Tag: `vX.Y.Z`
   - Verify release assets include:
     - `BOOTX64.EFI`
     - `BOOTX64.EFI.sha256`

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
   - Note: the build pulls the latest published `truthdb`, `installer`, and `installer-kernel` releases at that moment
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


## Where to add new docs

- Repo-specific docs: keep them in the repo that owns the behavior.
- Shared or historical documentation: the `docs/` repo.
- Org-level GitHub configuration and community health docs: the `.github/` repo.

## Pointers / known constraints

- The installer ISO is currently **UEFI-first** by design.
- Release workflows assume **amd64** for the Debian payload.
- Some components are still early-stage, but the release workflows are real and operational.
