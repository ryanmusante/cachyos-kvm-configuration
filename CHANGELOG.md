CachyOS KVM Configuration — Changelog
=====================================

Newest first. Versioning is MAJOR.MINOR.PATCH.
Format: - area: imperative summary.

6.0.2 (2026-07-05)
- verify: re-check every factual claim line-by-line against live sources. All
  package versions confirmed unchanged against the Arch DB (qemu-full 11.0.2,
  libvirt 12.5.0, virt-manager 5.1.0, edk2-ovmf 202605, swtpm 0.10.1, dnsmasq
  2.93, iptables 1.8.13, diffutils 3.12); the full edk2-ovmf dependency chain,
  libvirt NAT optdepends, and iptables-nft provision reconfirmed; the OVMF
  path, dnsmasq XML namespace (since 5.6.0), Claude Code native installer, and
  guest-agent/spice-vdagentd service names all reconfirmed. No corrections
  required
- target: pin exact ry-install figures — 4,952 lines / 288 functions at main
  HEAD (was expressed as ~4,950 / ~288); keep the rolling-pin caveat
- header/README: rewrite verification stamp to record the line-by-line
  re-verification and its full source scope

6.0.1 (2026-07-05)
- target: resync to ry-install v7.91.0; script now ~4,950 lines / ~288
  functions (was 5,067 / 293 at v7.88.3). Express as approximate rolling
  figures against main HEAD so a single upstream push does not stale the pin
- P6: fix header cross-reference — the workload note in the intro pointed the
  containment posture at (P7); containment is P6 (P7 is persistence/autostart)
- header/README: bump verification stamp to 2026-07-05

6.0.0 (2026-07-04)
- scope: reorient the document around KVM configuration as the subject;
  demote auditing from the document's purpose to a single "intended
  workload" reference section; rename artifact to
  cachyos-kvm-configuration
- title/purpose: rewrite header and README to describe a general-purpose
  isolated Linux guest, sized and contained for the audit workload rather
  than built around it
- P0: add /dev/kvm node check, CONFIG_KVM_AMD check, and a UEFI toggle
  table (SVM, IOMMU); note nested-virt is a host module param, out of scope
- P2: add explicit post-install verification block (socket state, virsh
  version, net-list, /dev/kvm perms); resync package versions to the Arch
  DB (qemu-full 11.0.2, libvirt 12.5.0, virt-manager 5.1.0, swtpm 0.10.1,
  dnsmasq 2.93, iptables 1.8.13, diffutils 3.12)
- P2: correct the edk2-ovmf provenance chain to include the qemu-base
  intermediate (qemu-full → qemu-desktop → qemu-base → qemu-system-x86 →
  edk2-ovmf), verified against the Arch package DB 2026-07-04
- P2: sharpen the iptables optdepend claim — libvirt's NAT optdepend is on
  iptables-nft specifically, which core iptables provides
- P3: expand the definition table with vCPU topology, disk bus rationale
  (virtio-scsi vs virtio-blk), cache/IO modes, virtio-rng, and an optional
  hardening subsection (TPM/swtpm, SEV); explain host-passthrough migration
  tradeoff
- P4: refocus on virtio integration (guest-agent + spice-vdagentd services,
  virtio verification on guest and host); move the review toolchain out to
  P5 so this phase is purely KVM/guest integration
- P5: rename to "guest workspace preparation"; gather workload tooling here;
  keep Claude Code install
- P6: former P7 — network containment retained verbatim in intent,
  reworded around "reach Anthropic, not the source host"
- P7: new phase — persistence + autostart (domain/network autostart, static
  vcpu pinning as optional tuning)
- P8: former P6 — snapshot baseline promoted after persistence; add
  --description and snapshot-list; rename snapshot to pre-run-baseline
- P9: add "autostart wedged" and "whole environment suspect / rebuild from
  P3" recovery rows
- target: resync to ry-install v7.88.3 (line/function counts unchanged at
  5,067 / 293)

5.2.1 (2026-07-03)
- target: resync to ry-install v7.88.2 (line/function counts unchanged at
  5,067 / 293)
- P4: confirm `cachyos-rate-mirrors` against the CachyOS FAQ; annotate the
  command with its documented behaviour (ranks CachyOS mirrors into
  /etc/pacman.d/cachyos-mirrorlist)

5.2.0 (2026-07-03)
- P2: correct dependency provenance against the Arch package DB — `libvirt`
  is pulled by `virt-manager` (via `virt-install`/`libvirt-glib`), not by
  `qemu-full`; `qemu-full` pulls `edk2-ovmf` via `qemu-system-x86`
- P2: add a `ufw route allow` note for hosts running ufw
- P3: add OS-autodetect fallback (select Arch Linux for the CachyOS ISO)

5.1.0 (2026-07-03)
- P2: reconcile host stack with the official CachyOS virtualization wiki —
  enable `libvirtd.socket` (not `.service`; service is the optional LXC
  backend), write `firewall_backend = "iptables"` to
  /etc/libvirt/network.conf, add `virsh net-autostart default`, and set the
  package set to `qemu-full virt-manager swtpm dnsmasq`
- P3: add the Q35 chipset row and the "Customize configuration before
  install" wizard step

5.0.1 (2026-07-03)
- P4: fix stale cross-reference (quiesced snapshots now point to P6)
- P4: drop `nodejs`/`npm` — the native Claude Code installer needs neither
- P4: justify retained `nvme-cli`/`lm_sensors`/`iw` against the script's own
  install set and runtime probes
- P3: remove a dangling version-lineage note; normalize P0/P9 heading tags

5.0.0 (2026-07-03)
- scope: remove the firmware-remediation phase and hardware-revision
  detection; reduce scope to the KVM audit environment only
- P0: add read-only KVM prerequisites (AMD-V check, kernel currency,
  disk/RAM headroom)
- structure: renumber phases to P0–P9; rebuild the phase map and recovery
  matrix; fix cross-references
- header: retain a one-line GTR9 Pro hardware base as the sizing basis

4.2.0 (2026-07-03)
- doc: remove the embedded changelog and sources appendix; retain inline
  source notes
- target: resync to v7.88.1 (5,067 lines / 293 functions)
- P2: switch to core `iptables` (1.8.13 provides `iptables-nft`); clarify
  libvirt-group vs kvm-group rationale
- P4: reframe the Claude Code native installer as primary (npm deprecated as
  default in v2.1.15); document `--dangerously-skip-permissions` as
  equivalent to `--permission-mode bypassPermissions` (refuses root)
- firmware: update the Intel envelope to release 31.2.1 / ixgbe 6.4.4; add
  the Secure Boot 2023 UEFI CA note

4.1.0 (2026-07-03)
- remove Appendix A (amdgpu kernel-parameter verdicts, Strix Halo tuning
  levers, sdboot-manage application)
- rewrite Pass 4 self-contained (no appendix dependency)
- relabel the sources appendix and prune appendix-only references

4.0.0 (2026-07-03)
- restructure into risk-rated phases, safest-first ordering
- hardware: add GTR9 Pro v1.0 (Intel E610) vs v2.2 (Realtek) revision
  detection; scope firmware remediation to v1.0 units
- firmware: E610 NVM 1.60 + driver pack 31.2; retain 1.30 as validated-fix
  floor; add EFI flash method and `ethtool` verification
- containment: add the net XML backup step; document the `net-update`
  limitation and the tap-detach / libvirtd-restart requirement
- shell: quote `"$USER"`; all blocks pass `bash -n`, `shellcheck`, `xmllint`

3.0.0 (2026-07-03)
- merge and deduplicate the two source documents (claude-code-text.txt
  v2.1.0 and exhaustive-claude.txt); retarget to GTR9 Pro + CachyOS
- fix the clone URL, containment ordering, corrupted hostname XML, OVMF
  path, `diffutils` package name, dnsmasq service conflict, empty baseline
  commit, reachability check, and git identity scope
- replace the npm Claude Code install with the native installer
- correct target repo stats to v7.87.7 / 5,066 lines / 292 functions
