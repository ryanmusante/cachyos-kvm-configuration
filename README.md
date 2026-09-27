# CachyOS KVM Configuration — Isolated Linux Guest on a GTR9 Pro Host

**Version:** 6.4.0 · **Date:** 2026-09-27

A risk-ordered procedure for building, containing and operating an isolated CachyOS KVM guest on a Beelink GTR9 Pro without exposing the physical host. The procedure lives in [`cachyos-kvm-configuration.md`](cachyos-kvm-configuration.md).

## Scope

The artifact covers **KVM configuration** end to end: host prerequisites and firmware toggles, the host virtualization stack, the full VM definition (chipset, firmware, CPU model, vCPU topology, memory, virtio disk/NIC/GPU/RNG, guest agent), guest-side virtio integration, network containment, persistence and autostart, and snapshot/recovery.

The guest's downstream use — reviewing the Fish installer `ryanmusante/ry-install` and its verifier `ryanmusante/ry-verify` with Claude Code — is recorded only as the workload the environment is **sized and contained for**, in a single reference section. The artifact deliberately excludes host firmware remediation, GPU/kernel-parameter tuning, and hardware-revision handling.

## Host and guest

- **Host:** Beelink GTR9 Pro — AMD Ryzen AI Max+ 395 (16 C / 32 T Zen 5, x86-64-v4), 128 GiB LPDDR5X unified memory, Radeon 8060S (`gfx1151`); CachyOS, rolling.
- **Guest:** CachyOS, virtio devices only, systemd-boot layout.
- **Workload (context only):** `ry-install.fish` (3,479 lines, 209 functions) and `ry-verify.fish` (3,099 lines, 232 functions), both v7.219.0 at `main` on 2026-09-27; engine Claude Code (native installer).

## Requirements

- A CachyOS host with AMD-V (SVM) enabled in firmware.
- At least 70 GiB free under `/var/lib` and 32 GiB of RAM to give the guest.
- The CachyOS ISO for the guest install.
- For the workload only: a Claude account with Claude Code access (Pro, Max, Team, Enterprise or Console).

## Table of contents

Execution order is dependency order; every mutating phase states its rollback in the document before it runs.

| Phase | Section | Risk |
| --- | --- | --- |
| P0 | [KVM prerequisites](cachyos-kvm-configuration.md#p0--kvm-prerequisites) | none (read-only) |
| P1 | [Host update hygiene](cachyos-kvm-configuration.md#p1--host-update-hygiene) | low |
| P2 | [Host virtualization stack](cachyos-kvm-configuration.md#p2--host-virtualization-stack) | low |
| P3 | [VM definition](cachyos-kvm-configuration.md#p3--vm-definition) | none |
| P4 | [Guest OS and virtio integration](cachyos-kvm-configuration.md#p4--guest-os-and-virtio-integration) | guest-only |
| P5 | [Guest workspace preparation](cachyos-kvm-configuration.md#p5--guest-workspace-preparation) | guest-only |
| P6 | [Network containment](cachyos-kvm-configuration.md#p6--network-containment) | low |
| P7 | [Persistence and autostart](cachyos-kvm-configuration.md#p7--persistence-and-autostart) | low |
| P8 | [Snapshot baseline](cachyos-kvm-configuration.md#p8--snapshot-baseline) | none (protective) |
| P9 | [Recovery procedures](cachyos-kvm-configuration.md#p9--recovery-procedures) | reference |
| — | [Intended workload](cachyos-kvm-configuration.md#intended-workload) | reference |

## Usage

1. Read the document top to bottom once before executing anything.
2. Execute phases in order and run each verification block before moving on. Do not skip the snapshot (P8) before the first live run of any workload inside the guest.
3. Apply network containment (P6) only after provisioning the workspace (P5) — the clones need GitHub reachable.
4. Keep host and guest on a current rolling kernel; do not pin LTS for the host role.

Shell blocks paste unchanged into bash or fish.

## Files

- `cachyos-kvm-configuration.md` — the configuration document (source of truth)
- `CHANGELOG.md` — version history, newest first
- `README.md` — this file
