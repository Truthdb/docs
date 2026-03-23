# Ubuntu-Style ISO Plan

## Purpose

This plan adopts the same high-level model used by Ubuntu Desktop:

- boot into a real live operating environment first
- run the installer as an application inside that environment
- perform installation from that fuller userspace instead of from a tiny initramfs-only runtime

This is the most conventional path if the priority is compatibility and polish across varied hardware and VM platforms.

## What "Ubuntu-style" Means Here

For TruthDB, "Ubuntu-style" does not have to mean "copy Ubuntu exactly" or "reuse Canonical's stack immediately".

It means adopting the same architecture:

1. bootloader loads a normal live system
2. live system starts a regular session
3. installer frontend runs inside that session
4. installer backend performs the real disk and system changes

That separation is the key idea.

## Why Change

Compared with the current design, a live environment gives:

- a much more stable hardware bring-up path
- full userland services, not just emergency initramfs tooling
- cleaner graphics, input, networking, and logging behavior
- easier debugging when something goes wrong
- a more normal place to run a graphical installer

If the target is "works equally well on VMware, VirtualBox, Hyper-V, UTM, QEMU, and normal machines", this is the strongest architectural direction.

## Target Outcome

Build a live installer ISO with:

- a standard bootloader path
- a compressed live root filesystem
- `systemd`, udev, networking, graphics/input stack, and normal userspace services
- a dedicated installer frontend
- a backend service or tool that performs disk probing, partitioning, payload install, and bootloader configuration

The user lands in a stable live session first, then installs from there.

## Proposed Architecture

### ISO boot layer

- Use a conventional ISO bootloader.
- Recommended: `GRUB` for the installer ISO.
- Support:
  - UEFI boot
  - BIOS boot
- Provide boot entries such as:
  - `Start Installer`
  - `Try TruthDB`
  - `Safe graphics`
  - `Verbose debug`
  - `Rescue shell`

### Live environment

- Boot into a live root filesystem, not a tiny installer initramfs only.
- Base it on a Debian or Ubuntu live userspace.

Recommended practical choice:
- Debian live-style base for control and easier alignment with the existing Debian payload

Alternative:
- Ubuntu-based live environment if reusing more Ubuntu packaging or installer stack components becomes attractive

### Frontend / backend split

- The installer frontend should be a real application, not PID 1 console output.
- The installer backend should be a service or dedicated backend tool.
- Frontend and backend communicate over a stable local API, for example:
  - D-Bus
  - local HTTP socket
  - Unix socket RPC

This makes the UI replaceable and keeps destructive logic out of the presentation layer.

### Installer frontend

Recommended end state:
- graphical installer UI

Possible implementations:
- Flutter
- GTK
- Qt
- web UI in kiosk mode

The exact toolkit is less important than the architecture. The frontend should manage:

- hardware readiness checks
- disk selection
- confirmation and warnings
- networking choices
- hostname/user/password setup
- install progress
- error screens and logs

### Installer backend

The backend should own:

- disk discovery
- partition plan creation
- filesystem creation
- target root mount handling
- payload extraction or system deployment
- bootloader installation
- installed-system configuration

This backend may reuse parts of the current `installer/` logic, but should be refactored out of the current initramfs-only assumptions.

## Live Session Modes

### Mode 1: Kiosk installer

- Boot straight into the installer UI.
- Keep a minimal desktop underneath, but hide it from normal users.

Advantages:
- cleaner branding
- simpler UX

### Mode 2: Try then install

- Boot into a small live desktop/session.
- Allow launching the installer from a desktop icon or shell.

Advantages:
- closer to Ubuntu Desktop
- user can inspect hardware/network behavior before committing

Recommendation:
- start with kiosk mode
- add "Try TruthDB" later if there is real product value in it

## Installation Model Options

### Option A: Keep current deployment model

- Continue embedding a prepared Debian payload.
- Installer backend writes partitions, extracts payload, configures system, installs bootloader.

Pros:
- closest to current TruthDB flow
- reuses existing install logic

Cons:
- still maintains a custom deployment pipeline

### Option B: Build from packages inside the live environment

- Install the target system from packages during install, more like a traditional distro installer.

Pros:
- more flexible

Cons:
- much more complexity
- larger network/package-management surface

Recommendation:
- use Option A first

That means the architecture changes to "Ubuntu-style", while the payload strategy stays familiar.

## Major Work Items

### Phase 1: Choose live ISO base

- Decide whether to base the live environment on Debian live tooling or Ubuntu live tooling.
- Build a prototype ISO that reaches a stable live session on the target VM matrix.

Deliverable:
- bootable live environment with keyboard, storage, and networking working

### Phase 2: Define frontend/backend boundary

- Split the current installer into:
  - backend install engine
  - frontend UI
- Remove assumptions that the installer owns the whole initramfs or console directly.

Deliverable:
- backend can be invoked from a running live session

### Phase 3: Build minimal live session

- Start a live session automatically.
- Launch the installer frontend on login or in kiosk mode.
- Provide a shell/debug escape path.

Deliverable:
- live TruthDB installer session that feels like a normal live OS, not an initramfs

### Phase 4: Implement installer UI

- Create the graphical frontend.
- Add screens for:
  - welcome
  - hardware checks
  - disk selection
  - destructive confirmation
  - hostname/user setup
  - progress
  - completion

Deliverable:
- installer can drive the backend end to end

### Phase 5: Integrate backend install actions

- Reuse or refactor current disk, partition, payload, and boot setup logic.
- Add progress reporting and structured errors for the frontend.

Deliverable:
- complete install path from live session to installed system

### Phase 6: Polish and recovery paths

- Add:
  - safe graphics mode
  - debug mode
  - rescue shell
  - install logs
  - graceful failure screens

Deliverable:
- recoverable and supportable installer behavior

### Phase 7: Validation matrix

Test on:

- VMware
- VirtualBox
- Hyper-V Gen2
- UTM
- QEMU
- at least one normal UEFI physical machine

Verify:

- live session boots reliably
- graphics/input work acceptably
- disk detection works
- installer UI remains responsive
- install completes
- installed system boots after ISO removal
- networking and `truthdb.service` work on first boot

## Advantages

- Strongest hardware and VM compatibility story.
- Best path to a polished graphical installer.
- Cleaner separation of UI and installation logic.
- Easier to debug because the environment is a normal live system.
- Most familiar user experience.

## Disadvantages

- Largest implementation cost.
- Significantly bigger ISO and runtime surface area.
- More packaging and session-management work.
- Higher maintenance burden than a custom text installer.

## Acceptance Criteria

This plan is successful when:

- the ISO boots to a stable live session on the target VM matrix
- the installer UI behaves predictably across those platforms
- the installation backend is decoupled from boot-console assumptions
- the installed system still boots and runs TruthDB correctly

## Recommendation

This is the best direction if the priority is:

- maximum cross-platform compatibility
- a genuinely polished installer UX
- moving toward a standard live-installer architecture

It is the more expensive path, but also the one that best matches the long-term goal of "works like a proper installer everywhere".
