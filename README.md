# CachyOS KVM Configuration — Isolated Linux Guest on a GTR9 Pro Host

**Version:** 6.3.0 · **Date:** 2026-08-30

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
`ryanmusante/ry-install` and its verifier `ryanmusante/ry-verify` with Claude
Code — is retained only as the workload the environment is **sized and
contained for**, and is confined to a single reference section. The artifact
deliberately excludes host firmware remediation, GPU/kernel-parameter tuning,
and hardware-revision handling.

## Host and guest

- **Host:** Beelink GTR9 Pro — AMD Ryzen AI Max+ 395 (16 C / 32 T Zen 5,
  x86-64-v4), 128 GB LPDDR5X, Radeon 8060S (`gfx1151`). OS base: CachyOS
  (rolling).
- **Guest:** CachyOS, virtio devices only, systemd-boot layout.
- **Workload (context only):** two repositories at v7.195.0 — `ry-install.fish`
  (3,413 lines, 207 functions) and `ry-verify.fish` (2,591 lines, 170
  functions), rolling, `main` HEAD as of 2026-08-30. Engine: Claude Code
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
   (P5) — the clones need GitHub reachable.
4. Keep host and guest on a current rolling kernel; do not pin LTS for the
   host role.

## Files

- `cachyos-kvm-configuration.md` — the configuration document (source of truth)
- `CHANGELOG.md` — version history, newest first
- `README.md` — this file
