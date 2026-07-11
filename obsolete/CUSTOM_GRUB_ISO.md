# Custom GRUB ISO Plan

## Purpose

This plan keeps TruthDB on a custom purpose-built installer ISO, but replaces the weakest parts of the current boot path:

- no more direct UKI-only boot as the main installer path
- no more BusyBox-only runtime as the whole installer environment
- no more installer UI fighting kernel messages on `/dev/console`

The goal is a more portable and cleaner installer that still remains small and controlled by TruthDB.

## Why Change

The current ISO is lean, but it has three structural problems:

1. The boot path is too custom for broad VM compatibility.
2. The runtime environment is too primitive for a polished installer UX.
3. Kernel logs and installer output share the same console, which makes the experience look broken even when the installer is working.

If the target is "works well on VMware, VirtualBox, Hyper-V, UTM, QEMU, and normal UEFI PCs", the current design is not the best base.

## Target Outcome

Build a conventional installer ISO with:

- `GRUB` as the ISO bootloader
- BIOS and UEFI boot support
- normal `vmlinuz + initrd` boot entries instead of relying only on direct UKI boot
- a small `systemd`-based or `dracut`-style initramfs runtime
- the installer running on its own VT with quiet boot defaults
- a broader installer kernel configuration for common VM and PC hardware

The installed system may still use `systemd-boot` if desired. This plan only changes the installer ISO architecture.

## Proposed Architecture

### ISO boot layer

- Use `GRUB` for the installer ISO.
- Support both:
  - UEFI boot
  - legacy BIOS boot
- Provide multiple menu entries:
  - `Install TruthDB`
  - `Install TruthDB (safe graphics)`
  - `Install TruthDB (verbose debug)`
  - `Install TruthDB (serial console)`
  - `Rescue shell`

### Kernel and initramfs

- Boot a normal installer kernel plus installer initramfs from GRUB.
- Stop using the current "installer on `/dev/console` via BusyBox init" design.
- Move to one of:
  - `systemd` in initramfs
  - `dracut` generated initramfs with custom units/hooks

Recommended direction: `systemd`-based initramfs, because it gives better service control, VT handling, and logging separation.

### Installer runtime

- Run the installer as a dedicated service.
- Present the UI on `tty1`.
- Keep kernel and service logs off the main installer screen by default.
- Use a quiet kernel command line for normal boot.
- Keep a debug boot entry that enables verbose logs and serial output.

### Installer UI

- Keep the installer text-based at first.
- Replace plain console printing with a proper fullscreen TUI.
- Use a single controlled screen with panels, prompts, and progress states.

This avoids the fragility of early graphics work while fixing the current visual mess.

### Installed system boot

- Keep the current installed-system boot approach for now:
  - GPT with ESP + root
  - `systemd-boot` on the target ESP
  - installed Debian kernel/initrd copied into the ESP

That keeps the migration smaller. The ISO boot path and installed-system boot path do not need to be identical.

## Major Work Items

### Phase 1: Rework ISO boot

- Replace the current direct EFI image layout with a GRUB-based ISO layout.
- Add UEFI and BIOS boot support.
- Define boot menu entries and per-entry kernel command lines.
- Keep one debug-oriented entry that resembles current verbose behavior.

Deliverable:
- an ISO that boots through GRUB in the main supported VM platforms

### Phase 2: Replace BusyBox-only init flow

- Remove BusyBox as the primary init model.
- Build a small initramfs that includes:
  - `systemd`
  - udev/device handling
  - required storage/network/install tools
  - the `truthdb-installer` binary
- Start the installer through a proper unit/service instead of `respawn` on `/dev/console`.

Deliverable:
- installer launches reliably on a dedicated VT with controlled startup ordering

### Phase 3: Quiet and separate output

- Remove noisy defaults such as aggressive `earlycon` and maximum kernel log verbosity.
- Reserve verbose output for dedicated debug entries.
- Send the installer UI to `tty1`.
- Keep kernel/service logs accessible through another VT or journal access.

Deliverable:
- no mixed kernel/UI output during the normal install path

### Phase 4: Broaden installer kernel support

- Review and widen the installer kernel config for:
  - EFI framebuffer and common display paths
  - virtio block/net
  - AHCI/SATA
  - NVMe
  - Hyper-V storage/network/input
  - VMware virtual hardware
  - VirtualBox virtual hardware
  - common USB keyboard/storage support

Deliverable:
- consistent boot and disk/network detection across the target VM matrix

### Phase 5: Upgrade installer UI

- Replace line-by-line stdout output with a structured TUI.
- Add explicit screens for:
  - disk detection
  - destructive confirmation
  - install progress
  - error handling
  - completion/reboot

Deliverable:
- usable installer UX without requiring a graphics stack

### Phase 6: Validation matrix

Test every ISO build on:

- VMware
- VirtualBox
- Hyper-V Gen2
- UTM
- QEMU
- at least one normal UEFI physical machine

For each platform, verify:

- ISO boot works
- keyboard works
- disk detection works
- install completes
- installed system boots after ISO removal
- DHCP comes up on first boot
- `truthdb.service` starts

## Advantages

- Much lower risk than moving straight to a full live desktop installer.
- More conventional ISO boot behavior.
- Better VM portability.
- Cleaner UI without depending on early graphics support.
- Easier debug path through separate GRUB entries.

## Disadvantages

- Still a custom installer stack.
- More moving parts than the current UKI + BusyBox approach.
- Less polished than a live desktop installer.
- BIOS support adds build and test complexity.

## Acceptance Criteria

This plan is successful when:

- the normal installer screen is clean and not mixed with kernel boot spam
- the ISO boots consistently across the agreed VM matrix
- safe/debug/rescue boot modes exist and are useful
- installer behavior is deterministic and recoverable
- installed systems still boot cleanly after installation

## Recommendation

This is the best next step if the goal is:

- broad compatibility
- cleaner UX
- low-to-medium implementation risk
- keeping full control of the installer stack

It is the most pragmatic migration path from the current design.
