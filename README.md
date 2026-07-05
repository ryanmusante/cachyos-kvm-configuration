# CachyOS KVM Configuration — Isolated Linux Guest on a GTR9 Pro Host

**Version:** 6.0.3 · **Date:** 2026-07-05

A comprehensive, risk-ordered procedure for building, containing, and operating
an isolated CachyOS KVM guest on a Beelink GTR9 Pro, without exposing the
physical host.

## Scope

This artifact covers the **KVM configuration** end to end: host prerequisites
and firmware toggles, the host virtualization stack, the full VM definition
(chipset, firmware, CPU model, vCPU topology, memory, virtio disk/NIC/GPU/RNG,
guest agent), guest-side virtio integration, network containment,
persistence/autostart, and snapshot/recovery.

The guest's downstream use — reviewing the Fish installer
`ryanmusante/ry-install` with Claude Code — is retained only as the workload
the environment is **sized and contained for**, and is confined to a single
reference section. It deliberately excludes host firmware remediation,
GPU/kernel-parameter tuning, and hardware-revision handling.

## Host and guest

- **Host:** Beelink GTR9 Pro — AMD Ryzen AI Max+ 395 (16 C / 32 T Zen 5,
  x86-64-v4), 128 GB LPDDR5X, Radeon 8060S (`gfx1151`). OS base: CachyOS
  (rolling).
- **Guest:** CachyOS, virtio devices only, systemd-boot layout.
- **Workload (context only):** `ry-install.fish` — v7.91.0, 4,952 lines,
  288 functions (rolling; `main` HEAD as of 2026-07-05). Engine: Claude Code
  (native installer).

## Phase order (safest first)

Execution order is dependency order. Every mutating phase is preceded by a
read-only or protective phase and states its rollback before execution.
P0–P7 are the KVM configuration proper; P8 gates any live workload; P9 is
recovery reference.

| Phase | Action | Risk |
| --- | --- | --- |
| P0 | KVM prerequisites (read-only) | none |
| P1 | Host update hygiene | low |
| P2 | Host virtualization stack | low |
| P3 | VM definition (virt-manager) | none to host |
| P4 | Guest OS + virtio tooling | guest-confined |
| P5 | Guest workspace preparation | none |
| P6 | Network containment (keep Anthropic, block GitHub) | low |
| P7 | Persistence + autostart | low |
| P8 | Snapshot baseline | none (protective) |
| P9 | Recovery procedures | reference |

## Usage

1. Read the document top to bottom once before executing anything.
2. Execute phases in order. Do not skip the snapshot (P8) before the first
   live run of any workload inside the guest.
3. Apply network containment (P6) only *after* provisioning the workspace
   (P5) — the clone needs GitHub reachable.
4. Keep host and guest on a current rolling kernel; do not pin LTS for the
   host role.

## Files

- `cachyos-kvm-configuration.md` — the configuration document (source of truth)
- `CHANGELOG.md` — version history, newest first
- `README.md` — this file

## Verification

All shell blocks pass `bash -n` and `shellcheck`; the containment network XML
passes `xmllint`. Every factual claim was re-verified line-by-line against live
sources on 2026-07-05:

- **Package versions** against the Arch package database (`qemu-full` 11.0.2,
  `libvirt` 12.5.0, `virt-manager` 5.1.0, `edk2-ovmf` 202605, `swtpm` 0.10.1,
  `dnsmasq` 2.93, `iptables` 1.8.13, `diffutils` 3.12).
- **Dependency chains and optdepends** against the same DB (the
  `qemu-full → qemu-desktop → qemu-base → qemu-system-x86 → edk2-ovmf` firmware
  chain; `virt-manager` pulling `libvirt` via `virt-install`/`libvirt-glib`;
  libvirt's `dnsmasq` and `iptables-nft` NAT optdepends; core `iptables`
  providing and replacing `iptables-nft` since the 1.8.11-3 merge).
- **The OVMF firmware path** `/usr/share/edk2/x64/OVMF_CODE.4m.fd` against the
  `edk2-ovmf` file list and the ArchWiki.
- **The dnsmasq XML namespace** (available since libvirt 5.6.0, no support
  guarantees) against the libvirt network-XML reference.
- **Claude Code** native-installer command, install path, authentication, and
  the `api.anthropic.com` startup dependency against the official setup docs.
- **Target repository stats** against `main` HEAD: `ry-install.fish` measures
  4,952 lines / 288 functions / v7.91.0.
