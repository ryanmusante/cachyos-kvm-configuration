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

## Sources

All shell blocks pass `bash -n` and `shellcheck`; the containment network XML
passes `xmllint`. Factual claims are checked against upstream sources, last
on 2026-08-30:

- **Package versions** against the Arch package database (`qemu-full` 11.1.1,
  `libvirt` 1:12.6.0, `virt-manager` 5.1.0, `edk2-ovmf` 202608, `swtpm`
  0.10.1, `dnsmasq` 2.93, `iptables` 1:1.8.13, `diffutils` 3.12).
- **Dependency chains and optdepends** against the same database (the
  `qemu-full → qemu-desktop → qemu-base → qemu-system-x86 → edk2-ovmf` firmware
  chain; `virt-manager` pulling `libvirt` via `virt-install`/`libvirt-glib`;
  libvirt's `dnsmasq` and `iptables-nft` NAT optdepends; core `iptables`
  providing and replacing `iptables-nft` since the 1:1.8.11-3 overhaul).
- **The OVMF firmware path** `/usr/share/edk2/x64/OVMF_CODE.4m.fd` against the
  `edk2-ovmf` file list and the ArchWiki.
- **Disk and snapshot validation rules** against libvirt 12.6.0 — the
  `io='native'` cache-mode requirement, the `--quiesce`/`--live` flag
  constraints on `snapshot-create-as`, and the release history for external
  snapshots (deletion from 9.0.0, reverting from 9.9.0).
- **The dnsmasq XML namespace** (available since libvirt 5.6.0, no support
  guarantees) against the libvirt network-XML reference.
- **The `/dev/kvm` device mode** against the systemd udev defaults.
- **The libvirt default URI** against libvirt 12.6.0 — an unqualified `virsh`
  probes to `qemu:///system` only for a privileged caller; everyone else gets
  `qemu:///session`, hence the `uri_default` step in P2.
- **dnsmasq `address=` semantics** against dnsmasq(8) — matching A and AAAA
  queries are answered locally and never forwarded.
- **How `lsblk` reports transport** against util-linux — `TRAN` is set to
  `virtio` for `vd*` names only, so a virtio-scsi disk shows none.
- **CachyOS optimized repositories** against the CachyOS wiki — repo
  selection and the v4/znver4 mirrorlist are a manual `pacman.conf` matter.
- **Claude Code** native-installer command, launcher path, authentication, and
  permission-flag semantics against the official setup docs, and the required
  endpoints against the network-configuration docs.
- **Target repository stats** against each `main` HEAD: `ry-install.fish`
  measures 3,413 lines / 207 functions and `ry-verify.fish` 2,591 lines /
  170 functions, both at v7.195.0.
