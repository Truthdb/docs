# Installer Boot + UX Strategy

Status: draft (agreed direction + current reality)
Last updated: 2026-01-23

This document captures the agreed installer/boot philosophy and maps it to what exists today.

It is intentionally split into:

- **Agreed direction** (what we’re building toward)
- **Current implementation** (what the repos do now)

For the concrete Debian-on-disk install steps, see `development/specs/INSTALL-DEBIAN.md`.

## Agreed direction (from prior agreements)

### 1) Minimal UEFI stage

- Keep the UEFI stage “dumb” and minimal.
- No installer wizard/business logic in UEFI.
- No networking in UEFI.

If the platform can boot the installer kernel directly via EFI stub / UKI, the UEFI loader can be omitted.

### 2) Installer kernel owns the installer experience

The installer environment (Linux kernel + initramfs/userspace) is responsible for:

- graphics initialization (early)
- branded splash + installer UI
- all installation logic (disk selection, partitioning, formatting, payload extraction, boot config)

### 3) Conservative disk safety posture

- Present disks clearly and unambiguously.
- Default posture is conservative (“make it hard to wipe the wrong disk”).
- Require explicit confirmation before destructive actions.

### 4) Quiet boot goal

- Avoid scrolling kernel messages / console spam.
- Switch to graphics and show a splash early.
- Applies to both installer boot and runtime boot.

Status: aspirational (not fully implemented today).

### 5) Two-kernel model

Keep installer and runtime kernels separate:

- **Installer kernel**: broad hardware support + installer UI/tooling.
- **Runtime kernel**: optimized, locked down for the appliance.

Status: partly aspirational. The current pipeline is focused on delivering an installer that installs Debian + TruthDB.

## Current implementation (what exists in this workspace)

### Repos and responsibilities (current)

- `installer/`: Rust installer app (console-only today).
- `installer-kernel/`: kernel config and release artifact used to boot the installer environment.
- `installer-kernel-builder-image/`: CI builder image for the kernel.
- `installer-iso/`: assembles initramfs tooling + Debian payload + UKI/ISO; enforces version-locking.

### Boot chain (current)

- ISO contains an ESP image with `EFI/BOOT/BOOTX64.EFI`.
- The ISO boot uses a UKI-style artifact assembled in CI (kernel + initramfs + cmdline).
- `truthdb-installer` runs in initramfs and performs the installation.

### UX (current)

- Console-only logs.
- Semi-attended: requires an ENTER confirmation before destructive operations.
- Disk selection is conservative: refuses to proceed if more than one eligible disk is found.

## Design invariants to keep stable

- Version-locking: the ISO release consumes matching tags across kernel/installer/truthdb.
- Offline install: no network required during install; Debian payload is embedded.
- Disk safety: no “pick first disk” behavior without explicit design decision.

## Open questions / future specs

- Define the desired “graphical installer UI” scope (framebuffer vs DRM/KMS, fonts/assets, input model).
- Decide whether a minimal UEFI loader is actually needed once UKI is stable.
- Define “runtime kernel” scope vs “installer kernel” scope.
- Define log export/diagnostics UX for failures.
