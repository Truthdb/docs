# INSTALL-DEBIAN.md

> Note: This lives in the org `.github` repo by request, but this repo’s README indicates product/architecture docs usually belong in the relevant product repository (e.g., `installer/` or `installer-iso/`). If we want to align with that, we can relocate this spec later.

## Goal
Build a TruthDB installer ISO that, when booted on a UEFI machine, installs a minimal Debian system onto an eligible disk and reboots into that Debian system.

Constraints:
- UEFI-only (no legacy BIOS support).
- No network is required during installation.
- After Debian boots, the installed system should obtain an IP via DHCP automatically.
- Installation is currently **semi-attended**: the installer asks for confirmation (press ENTER) before destructive steps.

## Current Implementation Status (Reality Check)

The current `truthdb-installer` implementation largely follows this spec’s install sequence, but there are a few important differences:

- **Not fully unattended yet**: the installer currently prompts for confirmation (press ENTER) before destructive steps.
- **Disk selection is stricter than “pick the first”**: it refuses to proceed if *more than one* eligible disk is found (safety gate).
- **Bootloader install uses file copy**: systemd-boot is installed by copying the EFI binary into the ESP (and writing loader entries), not by running `bootctl install` in the target.
- **Kernel cmdline differs**: the current loader entry uses `rw` and explicitly sets `init=/lib/systemd/systemd`.
- **Credentials are MVP defaults**: the installer currently sets both the `truthdb` user and `root` to a hardcoded password (`123456`).

Explicit networking policy:
- The installer environment must not attempt to bring up networking (no DHCP, no downloads).

## Definitions
- **Installer environment**: the initramfs userspace launched by the ISO (BusyBox + `truthdb-installer`).
- **Target disk**: the block device selected for installation (e.g., `/dev/vda`, `/dev/sda`, `/dev/nvme0n1`).
- **ESP**: EFI System Partition mounted at `/boot/efi` in the installed system.
- **Debian payload (rootfs payload)**: a prebuilt, compressed archive of a minimal Debian filesystem tree (e.g., a `tar.zst`) that the installer extracts onto the target root partition so it can install Debian **without** networking.

## Non-Goals (initially)
- Disk selection UI.
- Encryption (LUKS), LVM, RAID.
- Preserving existing data.
- Dual boot.

## End State (Acceptance Criteria)
After booting the ISO:
1. Installer prints the chosen target disk.
   - If no eligible disk exists, it fails with a clear error.
   - If more than one eligible disk exists, it refuses to choose automatically (fails with a clear error).
2. Installer partitions and formats the target disk:
   - GPT partition table.
   - Partition 1: ESP (FAT32), 512 MiB.
   - Partition 2: Debian root (EXT4), remainder.
3. Installer installs an offline minimal Debian root filesystem to the root partition.
4. Installer sets up UEFI boot with `systemd-boot` so the machine boots from the target disk without the ISO.
   - Current method: copy the systemd-boot EFI binary into the ESP and write loader entries.
5. On first boot of the installed Debian system:
   - The system reaches `multi-user.target`.
   - An Ethernet interface uses DHCP automatically.

## High-Level Approach
We will **embed a prebuilt minimal Debian root filesystem payload** inside the ISO (or initramfs) and have the installer write it to disk.

This avoids needing debootstrap, apt, mirrors, DNS, or DHCP during installation.

## Repository Responsibilities
This work spans multiple repos:

1. **installer** (Rust)
   - Add logic to detect disks, partition, format, mount, extract rootfs payload, configure boot.

2. **installer-iso**
   - Build the Debian rootfs payload (CI) and embed it into the initramfs at `/payload/debian-minbase-amd64-bookworm.tar.zst`.
   - Copy required external tools into initramfs (partitioning, formatting, mount, tar/zstd, chroot, efibootmgr, systemd-boot EFI bits).
   - Enforce version-matched inputs across repos for releases.

3. **installer-kernel**
   - Provide the kernel artifact used by the ISO build (`BOOTX64.EFI`).
   - (Tooling and payload assembly are handled by `installer-iso`.)

## Disk Detection and Selection
### Detection source
Use `/sys/block` to enumerate devices.

### Candidate filtering
Skip the following device classes:
- `loop*`, `ram*`, `sr*`, `fd*`, `dm-*`, `md*`.

Skip partitions; only accept whole disks (examples):
- Accept: `sda`, `vda`, `nvme0n1`.
- Reject: `sda1`, `vda2`, `nvme0n1p1`.

### Safety filters
Exclude disks that are:
- read-only (`/sys/block/<dev>/ro == 1`)
- removable (`/sys/block/<dev>/removable == 1`) (first iteration)
- smaller than a minimum size threshold (recommend: 8 GiB)

### Ordering
MVP safety policy (current):

- enumerate and log all eligible disks
- if exactly one eligible disk exists, select it
- if more than one eligible disk exists, abort (refuse to choose automatically)

### Logging
Current implementation logs the overall phase plus either:

- a single selected target disk line, or
- an error explaining why selection failed (no eligible disks, or multiple eligible disks).

## Partitioning Scheme (UEFI)
Target: GPT with 2 partitions.

- **Partition 1 (ESP)**
  - Size: 512 MiB
  - Type: EFI System Partition
  - Filesystem: FAT32
  - Mount point: `/boot/efi`

- **Partition 2 (root)**
  - Size: remaining space
  - Filesystem: EXT4
  - Mount point: `/`

Notes:
- NVMe partition naming uses `p1`, `p2`.
- SATA/virtio naming uses `1`, `2`.

## Formatting
- Format ESP: `mkfs.vfat -F 32 <esp>`
- Format root: `mkfs.ext4 -F <root>`

Optional (recommended): wipe old signatures before partitioning:
- `wipefs -a <disk>`

## Mount Layout During Install
Mount points in installer environment:
- `/mnt` → root partition
- `/mnt/boot/efi` → ESP

## Offline Debian Payload
This is the “offline Debian installer” approach: we ship a complete minimal Debian root filesystem as a single archive and extract it onto the target disk.

### Artifact
- Example name: `debian-minbase-amd64-bookworm.tar.zst`

### Contents
A bootable minimal Debian system containing:
- `systemd-sysv`
- `linux-image-amd64`
- `initramfs-tools`
- `e2fsprogs`
- `util-linux`
- `iproute2`
- `passwd` (for `chpasswd`)

Notes:
- Firmware packages are excluded initially.
- No network packages are required for first boot DHCP if using systemd-networkd (see below).

### TruthDB payload contents (MVP)
The Debian payload must also include the TruthDB runtime artifacts so the installer can simply enable the service.

- TruthDB binary at a stable path:
   - Recommend: `/usr/local/bin/truthdb`
- systemd unit shipped in the payload:
   - Recommend: `/lib/systemd/system/truthdb.service`
   - The unit should include `Wants=network-online.target` and `After=network-online.target`.
- Optional but recommended (can be deferred if TruthDB defaults are acceptable):
   - Config directory: `/etc/truthdb/`
   - Data directory: `/var/lib/truthdb/`

### Build-time generation (CI)
In the ISO build pipeline:
1. Run debootstrap to a staging directory.
2. Configure minimal system files.
3. Install required packages into the staging rootfs.
   - Ensure the Debian kernel and initramfs are present under `/boot` (e.g., by installing `linux-image-amd64` and running `update-initramfs` in the chroot during image build).
   - Set a known root password for the MVP (see “Root credentials”).
4. Clean apt caches.
5. Pack as a tarball with numeric ownership.

### Runtime installation
- Extract payload to `/mnt` with preservation of permissions, owners, symlinks:
  - `tar --zstd -xpf <payload> -C /mnt`

## Bootloader Setup (systemd-boot)
### Why systemd-boot
UEFI-only with minimal moving parts.

### Steps (installed system)
Current implementation:

- The installer copies the systemd-boot EFI binary into the ESP (both the fallback path and the systemd path), then writes the loader config and a single entry.
- It optionally attempts to create an NVRAM entry via `efibootmgr` (best-effort).

### Boot entry contents
Current implementation uses:

- `options root=UUID=<root-uuid> rw init=/lib/systemd/systemd`
- kernel/initrd copied into the ESP (e.g. `/EFI/debian/vmlinuz` and `/EFI/debian/initrd.img`)

## fstab
Write `/mnt/etc/fstab` entries using UUIDs:
- Root ext4 mounted at `/`.
- ESP vfat mounted at `/boot/efi`.

## First Boot DHCP
Configure systemd-networkd to bring up DHCP automatically.

### Files
Create `/mnt/etc/systemd/network/20-dhcp.network`:
- Match common wired interface patterns (e.g., `en*`, `eth*`).
- Enable DHCP.

Enable required services in the target:
- `systemd-networkd.service`
- `systemd-networkd-wait-online.service` (so `network-online.target` behaves as expected)
- `systemd-resolved.service`

Configure `/etc/resolv.conf` to use systemd-resolved stub.

## Root Credentials (MVP)
Set the root password of the installed Debian system to `123456`.

Current implementation also creates a `truthdb` user and sets its password to the same value.

Notes:
- This is insecure and intended only for early bring-up.
- Follow-up work should replace this with a safer approach (first-boot password change, SSH keys, or provisioning).

## Installer Runtime Dependencies
The initramfs environment must include:
- Partitioning tool: `sfdisk` or `parted`
- Formatting tools: `mkfs.vfat`, `mkfs.ext4`
- Mount tooling: `mount`, `umount`
- Extraction tooling: `tar` with zstd support (or separate `zstd` + tar)
- `wipefs` (optional)

Note: `installer-iso` release workflow is the authoritative place that assembles these tools into initramfs.

## Error Handling Requirements
Installer must:
- Abort if target disk appears mounted.
- Abort if any command fails; print the failing step.
- Leave clear logs on console for post-mortem.

## Installer UX / Output Requirements
The current installer is console-only and prints step-prefixed logs to stdout.

Notes:

- It does not currently print a full table of all candidate disks.
- It prints multiple lines for some phases (e.g., partition device paths).

## Verification Plan
- Boot ISO in UTM:
  - Confirm disk enumeration output.
  - Confirm partitions created and formatted.
  - Confirm reboot enters Debian without ISO.
  - Confirm DHCP assigns IP after boot (`ip a`).

- Smoke test on real UEFI hardware:
  - Confirm ESP creation.
  - Confirm boot entry works.

## Installer Execution Sequence (MVP)
The installer should run the following steps in order and abort on the first failure:

1. **Enumerate disks** and decide on a target disk.
   - If no eligible disk exists, abort.
   - If more than one eligible disk exists, abort (current safety gate).
2. **Print chosen target disk** (device path + size).
3. **Prompt for confirmation** before partitioning/formatting.
4. **Safety checks** on the chosen target:
    - Confirm it is not currently mounted (no mounted children).
    - Confirm it meets minimum size threshold.
    - Confirm it is writable and non-removable (per current safety filters).
5. **Wipe signatures** (recommended): `wipefs -a <disk>`.
6. **Partition** the disk as GPT with:
    - ESP 512 MiB
    - Root = remainder
7. **Re-read partition table** (e.g., `partprobe` or a short udev settle, depending on available tooling).
8. **Format**:
    - `mkfs.vfat -F 32 <esp>`
    - `mkfs.ext4 -F <root>`
9. **Mount**:
    - Root → `/mnt`
    - ESP → `/mnt/boot/efi`
10. **Install Debian payload** by extracting into `/mnt`.
11. **Set root password** in the target.
   - Example (inside installer environment):
     - `chroot /mnt /bin/sh -lc 'echo "root:123456" | chpasswd'`
12. **Write `/etc/fstab`** in the target using UUIDs.
13. **Configure bootloader**:
   - Copy systemd-boot EFI binary into the ESP (including the fallback path `EFI/BOOT/BOOTX64.EFI`).
   - Write loader config and a single entry referencing kernel/initrd on the ESP and `root=UUID=<root-uuid>`.
   - Best-effort: create an NVRAM boot entry via `efibootmgr` (non-fatal if unsupported).
14. **Configure first-boot DHCP** (systemd-networkd) in the target and enable services.
15. **Install + enable TruthDB systemd service** in the target.
   - The unit file must already exist in the target (shipped in the Debian payload).
    - In practice, the ISO build enables the unit in the payload at build time; the installer may also enable units by creating the appropriate symlinks under `/etc/systemd/system/*`.
16. **Sync + unmount** all mounts.
17. **Reboot**.

## TruthDB Service (systemd)
The installed Debian system must start TruthDB automatically on boot via systemd.

### Requirements (MVP)
- A systemd unit named `truthdb.service` exists on the target after install (shipped in the Debian payload).
   - Recommend location: `/lib/systemd/system/truthdb.service`
- The installer enables the service in the target via chroot:
   - `chroot /mnt /bin/sh -lc 'systemctl enable truthdb.service'`
- TruthDB requires network; the unit must order itself accordingly.
   - Required: `Wants=network-online.target` and `After=network-online.target`.
   - Ensure the installed system provides `network-online.target` for DHCP (e.g., via systemd-networkd + wait-online).

### Example unit file (MVP)
This is a minimal example of what should be shipped in the Debian payload as `/lib/systemd/system/truthdb.service`:

```ini
[Unit]
Description=TruthDB
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/truthdb
Restart=on-failure
RestartSec=2

[Install]
WantedBy=multi-user.target
```

### Acceptance Criteria
- After first boot, `systemctl is-enabled truthdb.service` returns `enabled`.
- After first boot, `systemctl is-active truthdb.service` returns `active` (or it must have clear logs explaining failure).

## Work Breakdown (per repo)
This is the concrete engineering work implied by the spec.

### `installer/` (Rust)
- Disk enumeration via `/sys/block` + safety filters (existing mounted checks included).
   - Abort if more than one eligible disk exists (MVP safety gate).
- Partitioning implementation (choose one):
   - `sfdisk` scripted GPT layout, or
   - `parted` non-interactive.
- Filesystem formatting wrappers and mount orchestration.
- Payload discovery + extraction (see “Payload placement” below).
- Root password set (MVP) via chroot (e.g., `chpasswd`).
- Target configuration writes:
   - `/etc/fstab` using UUIDs
   - systemd-boot loader files
   - systemd-networkd DHCP config + `systemctl enable ...` in chroot
   - TruthDB systemd service enablement: `systemctl enable truthdb.service` in chroot
- Error handling:
   - one clear “step name” per stage
   - propagate stderr to console
   - fail closed on any ambiguity (e.g., no eligible disks)

### `installer-kernel/`
This repo primarily provides the kernel artifact used in the ISO build (`BOOTX64.EFI`).

### `installer-iso/`
- Build the Debian rootfs payload and embed it into initramfs at `/payload/debian-minbase-amd64-bookworm.tar.zst`.
- Copy required external install tools into initramfs (partitioning/filesystems/mount/tar+zstd/chroot/efibootmgr/systemd-boot EFI bits).
- Keep strict release coupling between kernel + installer + payload + truthdb artifacts (same version).

## Payload Placement
For MVP, the payload lives **inside initramfs**.

- Payload is available at a fixed path: `/payload/debian-minbase-amd64-bookworm.tar.zst`.
- The installer reads from that path and extracts to `/mnt`.
- No need to mount the ISO filesystem at runtime.

Follow-up option (not MVP): store the payload as a file on the ISO and mount `/dev/sr0` read-only to access it.

## Safety Gates (recommended)
Because this is destructive, the MVP safety gate is:
- Abort if more than one eligible disk is present.

Follow-up options (not required for MVP):
- Require a kernel cmdline flag (e.g., `truthdb.install=1`) before doing any writes.
- Require explicit target disk override (e.g., `truthdb.disk=/dev/sda`) and refuse auto-pick.

## Open Questions
- Target Debian suite and kernel strategy:
  - `bookworm` pinned, or track `stable`?
- Disk selection safety:
  - require an explicit kernel cmdline flag to allow destructive install?

