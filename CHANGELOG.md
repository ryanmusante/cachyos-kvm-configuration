CachyOS KVM Configuration — Changelog
=====================================

Newest first. Versioning is MAJOR.MINOR.PATCH.
Format: - area: imperative summary.

6.1.0 (2026-07-26)
- P3: fix an invalid disk configuration. libvirt refuses io='native'
  unless the cache mode is none or directsync, so cache=writeback with
  io=native never starts; ship io=threads and list the valid pairings
- P3: correct the disk-bus rationale. virtio-blk has carried
  discard/unmap since QEMU 4.0; virtio-scsi is kept for its multi-queue
  path and multi-disk topology, not for trim support
- P8: fix an invalid snapshot command. --quiesce requires --disk-only,
  and --live is accepted only for a full-system external snapshot
  (--memspec). Split into a powered-off internal snapshot (preferred)
  and a quiesced disk-only alternative for a running guest
- P2: correct the iptables-nft merge provenance. The packaging overhaul
  landed in 1:1.8.11-3 (released 2025-10-03) and was refined in
  1:1.8.11-4; record the epoch on the current 1:1.8.13-1
- P2: fix the /dev/kvm mode in the verification block. systemd ships
  dev-kvm-mode 0666, so the node is crw-rw-rw- root kvm
- P2: fix rollback. /etc/libvirt/network.conf ships with the libvirt
  package; remove the appended line instead of the file
- P3: fix a one-character misalignment in the definition table
- P5: configure the clone with git -C instead of cd, matching the P6
  verification block and keeping every shell block shellcheck-clean
- target: resync to ry-install v7.139.0 (4,974 lines / 293 functions)
- header/README: restate the source stamp for 2026-07-26 and widen the
  cited scope to libvirt validation rules and systemd

6.0.3 (2026-07-05)
- P2: fix a dependency-provenance error carried since 4.2.0 — the
  iptables-nft merge was attributed to the current version 1.8.13
  rather than to its own release
- P2: reconcile a stale internal verification date
- header/README: bump the verification stamp

6.0.2 (2026-07-05)
- verify: re-check every factual claim against live sources. Package
  versions, dependency chains, optdepends, OVMF path, dnsmasq XML
  namespace, and service names all confirmed; no corrections required
- target: pin exact ry-install figures — 4,952 lines / 288 functions at
  main HEAD (was ~4,950 / ~288); keep the rolling-pin caveat
- header/README: rewrite the verification stamp

6.0.1 (2026-07-05)
- target: resync to ry-install v7.91.0; ~4,950 lines / ~288 functions
  (was 5,067 / 293 at v7.88.3), expressed against main HEAD
- P6: fix header cross-reference — containment is P6, not P7
- header/README: bump verification stamp to 2026-07-05

6.0.0 (2026-07-04)
- scope: reorient the document around KVM configuration; demote the
  review workload to a single reference section; rename the artifact to
  cachyos-kvm-configuration
- title/purpose: rewrite header and README around a general-purpose
  isolated Linux guest, sized and contained for that workload
- P0: add /dev/kvm and CONFIG_KVM_AMD checks and a UEFI toggle table
  (SVM, IOMMU); note nested virt is a host module param, out of scope
- P2: add a post-install verification block (socket state, virsh
  version, net-list, /dev/kvm perms); resync package versions
- P2: correct the edk2-ovmf chain to include the qemu-base intermediate
- P2: sharpen the iptables optdepend claim — libvirt's NAT optdepend is
  on iptables-nft specifically, which core iptables provides
- P3: expand the definition table with vCPU topology, disk bus
  rationale, cache/IO modes, and virtio-rng; add optional hardening
  (TPM/swtpm, SEV); explain the host-passthrough migration tradeoff
- P4: refocus on virtio integration; move the review toolchain to P5
- P5: rename to "guest workspace preparation"; gather workload tooling
  here; keep the Claude Code install
- P6: former P7 — network containment, reworded around "reach
  Anthropic, not the source host"
- P7: new phase — persistence and autostart, with static vcpu pinning
  as optional tuning
- P8: former P6 — snapshot baseline promoted after persistence; add
  --description and snapshot-list; rename to pre-run-baseline
- P9: add "autostart wedged" and "whole environment suspect" rows
- target: resync to ry-install v7.88.3 (5,067 lines / 293 functions)

5.2.1 (2026-07-03)
- target: resync to ry-install v7.88.2 (counts unchanged)
- P4: confirm cachyos-rate-mirrors against the CachyOS FAQ and annotate
  it with its documented behaviour

5.2.0 (2026-07-03)
- P2: correct dependency provenance — libvirt is pulled by virt-manager
  (via virt-install/libvirt-glib), not by qemu-full; qemu-full pulls
  edk2-ovmf via qemu-system-x86
- P2: add a ufw route allow note for hosts running ufw
- P3: add the OS-autodetect fallback (Arch Linux for the CachyOS ISO)

5.1.0 (2026-07-03)
- P2: reconcile the host stack with the CachyOS virtualization wiki —
  enable libvirtd.socket, write firewall_backend = "iptables", add
  virsh net-autostart default, and set the package set
- P3: add the Q35 chipset row and the customize-before-install step

5.0.1 (2026-07-03)
- P4: fix a stale cross-reference for quiesced snapshots
- P4: drop nodejs/npm — the native Claude Code installer needs neither
- P4: justify retained nvme-cli/lm_sensors/iw against the target
  script's own package set and runtime probes
- P3: remove a dangling version-lineage note; normalize P0/P9 headings

5.0.0 (2026-07-03)
- scope: remove firmware remediation and hardware-revision detection;
  reduce scope to the KVM guest environment only
- P0: add read-only KVM prerequisites (AMD-V, kernel currency,
  disk/RAM headroom)
- structure: renumber phases to P0-P9; rebuild the phase map and
  recovery matrix; fix cross-references
- header: retain a one-line GTR9 Pro hardware base as the sizing basis

4.2.0 (2026-07-03)
- doc: remove the embedded changelog and sources appendix; retain
  inline source notes
- target: resync to v7.88.1 (5,067 lines / 293 functions)
- P2: switch to core iptables; clarify libvirt-group vs kvm-group
- P4: reframe the Claude Code native installer as primary; document
  --dangerously-skip-permissions as equivalent to --permission-mode
  bypassPermissions, and that it refuses to run as root
- firmware: update the Intel envelope to 31.2.1 / ixgbe 6.4.4; add the
  Secure Boot 2023 UEFI CA note

4.1.0 (2026-07-03)
- remove Appendix A (amdgpu kernel-parameter verdicts, Strix Halo
  tuning levers, sdboot-manage application)
- rewrite Pass 4 self-contained, with no appendix dependency
- relabel the sources appendix and prune appendix-only references

4.0.0 (2026-07-03)
- restructure into risk-rated phases, safest-first ordering
- hardware: add GTR9 Pro v1.0 (Intel E610) vs v2.2 (Realtek) revision
  detection; scope firmware remediation to v1.0 units
- firmware: E610 NVM 1.60 + driver pack 31.2; retain 1.30 as the
  validated-fix floor; add the EFI flash method and ethtool checks
- containment: add the net XML backup step; document the net-update
  limitation and the tap-detach / libvirtd-restart requirement
- shell: quote "$USER"; all blocks pass bash -n, shellcheck, xmllint

3.0.0 (2026-07-03)
- merge and deduplicate the two source documents; retarget to GTR9 Pro
  and CachyOS
- fix the clone URL, containment ordering, corrupted hostname XML, OVMF
  path, diffutils package name, dnsmasq service conflict, empty
  baseline commit, reachability check, and git identity scope
- replace the npm Claude Code install with the native installer
- correct target repo stats to v7.87.7 (5,066 lines / 292 functions)
