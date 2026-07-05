# CACHYOS KVM CONFIGURATION — ISOLATED LINUX GUEST ON A GTR9 PRO HOST

**Version:** 6.0.3 · **Date:** 2026-07-05 · **Purpose:** Comprehensive, risk-ordered procedure for building, containing, and operating an isolated CachyOS KVM guest on a Beelink GTR9 Pro. This revision reorients the document around **KVM configuration** as the subject; the guest's downstream use (auditing the Fish installer `ryanmusante/ry-install` with Claude Code) is retained only as the workload the environment is sized and contained for. Every factual claim re-verified line-by-line against live sources on 2026-07-05 — Arch package DB (package versions, dependency chains, optdepends, OVMF file path, the `iptables-nft`→`iptables` merge version), libvirt upstream (dnsmasq XML namespace), CachyOS wiki/FAQ (host stack, mirror ranking), Claude Code official docs (native installer, permission flag, endpoints), and target repo HEAD (line/function counts). This pass corrected one dependency-provenance error carried since v4.2.0 (the `iptables-nft` merge was attributed to the current version 1.8.13 rather than its actual landing at 1.8.11-3) and reconciled a stale internal verification date. All fenced blocks syntax-checked (`bash -n`, `shellcheck`, `xmllint`).

**Host hardware base:** Beelink GTR9 Pro — AMD Ryzen AI Max+ 395 (16 C / 32 T Zen 5, x86-64-v4), 128 GB LPDDR5X, Radeon 8060S (RDNA 3.5, `gfx1151`). OS base: CachyOS (rolling). The guest sees only virtio devices; all sizing below derives from these figures.

**Guest target:** CachyOS, virtio-only, systemd-boot layout.

**Intended workload (context only):** static and dynamic review of `ry-install.fish` — **v7.91.0 (2026-07-04) · 4,952 lines · 288 functions** (single script, rolling; figures reflect `main` HEAD as of 2026-07-05, verified via raw.githubusercontent.com — treat as an approximate rolling pin, since a single upstream push moves them). The workload dictates two configuration constraints and nothing more: enough vCPU/RAM to run an editor plus an AI agent comfortably (P3), and a containment posture that keeps Anthropic endpoints reachable while cutting the guest off from the source host after clone (P6). Everything else in this document is general-purpose KVM configuration.

---

## PHASE MAP — ORDERED BY SAFEST IMPLEMENTATION

Execution order = dependency order. Every mutating phase is preceded by a read-only or protective phase, and each phase states its rollback before you run it. Phases P0–P7 are the KVM configuration proper; P8 is the protective snapshot that gates any live workload; P9 is recovery reference.

```
╔═════╦═══════════════════════════╦═══════╦══════════════════════╗
║ PH  ║ ACTION                    ║ RISK  ║ ROLLBACK             ║
╠═════╬═══════════════════════════╬═══════╬══════════════════════╣
║ P0  ║ KVM prerequisites         ║ NONE  ║ n/a                  ║
║     ║ (read-only)               ║       ║                      ║
║ P1  ║ Host update hygiene       ║ LOW   ║ pacman cache         ║
║ P2  ║ Virtualization stack      ║ LOW   ║ disable svc, remove  ║
║     ║                           ║       ║ pkgs, gpasswd -d     ║
║ P3  ║ VM definition             ║ NONE  ║ delete domain        ║
║ P4  ║ Guest OS + virtio tooling ║ NONE* ║ reinstall guest      ║
║ P5  ║ Guest workspace prep      ║ NONE  ║ delete clone         ║
║ P6  ║ Network containment       ║ LOW   ║ restore net XML bkp  ║
║ P7  ║ Persistence + autostart   ║ LOW   ║ net/domain autostart ║
║ P8  ║ Snapshot baseline         ║ NONE  ║ n/a (protective)     ║
║ P9  ║ Recovery procedures       ║ —     ║ reference            ║
╚═════╩═══════════════════════════╩═══════╩══════════════════════╝
* = risk confined to guest
```

---

## P0 — KVM PREREQUISITES — RISK: NONE (READ-ONLY)

Nothing here mutates state — confirm the host can carry the guest before installing anything.

```
$ lscpu | grep -qw svm && echo "AMD-V: present"
$ grep -qw kvm /proc/misc && echo "/dev/kvm node: present"
$ zgrep -E 'CONFIG_KVM_AMD=[ym]' /proc/config.gz    # KVM AMD in-kernel
$ uname -r && pacman -Q linux-firmware
$ df -h / && free -h
```

Expect: `svm` listed (AMD-V — KVM cannot start without it); a `/dev/kvm` node available (proves the `kvm_amd` module is loaded); `CONFIG_KVM_AMD` built in or as a module; a current CachyOS rolling kernel; `linux-firmware` present; ≥ 70 GiB free for the 60 GiB qcow2 plus install ISO; and comfortable headroom over the 32 GiB guest allocation against the host's 128 GiB.

Two host-firmware toggles are prerequisites, not software steps — verify them in UEFI setup if `/dev/kvm` is absent:

```
╔══════════════════════════╦════════════════════════════════════╗
║ UEFI SETTING             ║ REQUIRED STATE                     ║
╠══════════════════════════╬════════════════════════════════════╣
║ SVM Mode (AMD-V)         ║ Enabled — hardware virtualization  ║
║ IOMMU / AMD Vi           ║ Enabled — needed only if you later ║
║                          ║ pass a physical device through;    ║
║                          ║ harmless to leave on for virtio    ║
╚══════════════════════════╩════════════════════════════════════╝
```

Nested virtualization (running KVM *inside* this guest) is out of scope; if ever needed, it is a host module parameter (`kvm_amd nested=1`), not a guest setting.

---

## P1 — HOST UPDATE HYGIENE — RISK: LOW

```
$ curl -s https://cachyos.org/news | head -40
$ sudo pacman -Syu
$ pacdiff
```

Check cachyos.org/news before `-Syu` (breaking-change notices); run `pacdiff` after to reconcile any `.pacnew` files. A current rolling kernel also means a current KVM/virtio stack — do not pin LTS for this host role. Full `-Syu` only; never a partial upgrade. Rollback for a regressing kernel/qemu bump: reinstall the prior package from the pacman cache (`/var/cache/pacman/pkg/`) with `pacman -U`.

---

## P2 — HOST VIRTUALIZATION STACK — RISK: LOW, REVERSIBLE

```
$ sudo pacman -S --needed qemu-full virt-manager swtpm dnsmasq
$ echo 'firewall_backend = "iptables"' | sudo tee -a /etc/libvirt/network.conf
$ sudo systemctl enable --now libvirtd.socket
$ sudo virsh net-autostart default
$ sudo usermod -aG libvirt "$USER"
```

**Package set and dependency provenance** (Arch package DB, verified 2026-07-05 — `qemu-full` 11.0.2, `libvirt` 12.5.0, `virt-manager` 5.1.0, `edk2-ovmf` 202605, `swtpm` 0.10.1, `dnsmasq` 2.93):

- `libvirt` is pulled by **`virt-manager`** (via `virt-install` → `libvirt-python`, and `libvirt-glib`), *not* by `qemu-full`.
- `qemu-full` pulls **`edk2-ovmf`** transitively (via `qemu-desktop` → `qemu-base` → `qemu-system-x86`) for UEFI guest firmware.
- `swtpm` supplies a guest TPM — optional, exercised only if you add a TPM device; not required for this Linux guest.
- `dnsmasq` is listed explicitly because libvirt declares it only as an *optional* dependency (package DB: "required for default NAT/DHCP for guests").
- libvirt's other NAT optdepend is on **`iptables-nft`** (package DB: "required for default NAT networking"). Core `iptables` (1.8.13-1, verified) both *provides* and *replaces* `iptables-nft` — the standalone `iptables-nft` package was folded into core `iptables` (nft backend made default) at `iptables` 1.8.11-3, which landed 2026-04-05; the name has pointed at the nft-backed build ever since — and it is present in a standard CachyOS install, so that optdepend is already satisfied. Hence no explicit `libvirt`/`iptables` line is needed.

**Daemon model — socket activation** (CachyOS wiki): enable `libvirtd.socket`, not the service — the daemon starts on first client connection. Enabling `libvirtd.service` as well is only needed for the optional LXC backend.

**Firewall backend** (CachyOS wiki): current libvirt defaults its firewall backend to `nftables`; the CachyOS-tested value is `iptables`, written to `/etc/libvirt/network.conf`. This governs how libvirt programs the NAT rules for `virbr0`.

**NAT network autostart:** `virsh net-autostart default` makes the built-in NAT network start on boot so guests get connectivity without a manual `net-start`.

**Firewall interaction:**
- If the host runs **ufw**, also permit guest transit: `sudo ufw route allow from 192.168.122.0/24` (libvirt's default subnet; CachyOS wiki). Without ufw, libvirt's own iptables rules for the NAT network suffice — no action needed.
- No `nftables.service` enable is required for NAT; libvirt manages its own firewall rules.

**Do not enable a global dnsmasq:** libvirt spawns its own per-network dnsmasq bound to `virbr0`. A global instance on `0.0.0.0:53` conflicts unless configured with `bind-interfaces`/`except-interface`. The package only needs to be *installed*, not enabled. (Ref: wiki.archlinux.org/title/Libvirt.)

**Group model:** log out/in (or `newgrp libvirt` per shell) after `usermod`. No reboot; no `kvm` group step — system-mode VMs run under libvirt's own qemu user, so your account needs only `libvirt` for daemon-socket access (CachyOS wiki; Arch wiki Libvirt, "Using libvirt group").

**Post-install verification:**

```
$ systemctl is-active libvirtd.socket
$ virsh -c qemu:///system version
$ virsh net-list --all              # default → active, autostart yes
$ ls -l /dev/kvm                    # crw-rw----+ root kvm
```

**Rollback:**

```
$ sudo systemctl disable --now libvirtd.socket
$ sudo gpasswd -d "$USER" libvirt
$ sudo rm /etc/libvirt/network.conf
$ sudo pacman -Rns qemu-full virt-manager swtpm dnsmasq
```

---

## P3 — VM DEFINITION (VIRT-MANAGER) — RISK: NONE TO HOST

The core of the KVM configuration. Every value below is chosen for a modern Linux guest on this specific silicon; deviations are noted where they matter.

```
╔══════════════════╦═══════════════════════════════════════════╗
║ SETTING          ║ VALUE (GTR9 PRO-TUNED)                    ║
╠══════════════════╬═══════════════════════════════════════════╣
║ Chipset          ║ Q35 (never i440FX for a modern Linux      ║
║                  ║ guest; Q35 gives native PCIe, needed for  ║
║                  ║ clean virtio and UEFI)                    ║
║ Firmware         ║ UEFI: /usr/share/edk2/x64/OVMF_CODE.4m.fd ║
║                  ║ (edk2-ovmf 202605, pulled by qemu-full;   ║
║                  ║ prefer virt-manager's UEFI dropdown)      ║
║ CPU model        ║ host-passthrough (exposes x86-64-v4 →     ║
║                  ║ guest CachyOS auto-selects the same       ║
║                  ║ optimized repos as bare metal)            ║
║ vCPU topology    ║ 16 vCPUs, 1 socket / 8 cores / 2 threads  ║
║                  ║ (raise to 24 for compile-heavy work; host ║
║                  ║ keeps the remainder of 32 threads)        ║
║ Memory           ║ 32 GiB fixed (host has 128; leave         ║
║                  ║ ballooning at default virtio-balloon)     ║
║ Disk bus         ║ VirtIO SCSI (virtio-scsi), not virtio-blk ║
║                  ║ — SCSI carries the discard/unmap path     ║
║ Disk cache/IO    ║ cache=writeback · discard=unmap ·         ║
║                  ║ io=native · 60 GiB qcow2                   ║
║ NIC              ║ virtio · network: default (NAT)           ║
║ Video            ║ virtio-gpu (no host GPU passthrough)      ║
║ Guest agent      ║ virtio channel org.qemu.guest_agent.0     ║
║ RNG              ║ virtio-rng backed by /dev/urandom         ║
║ Clipboard/resize ║ SPICE + spice-vdagent (set up in P4)      ║
╚══════════════════╩═══════════════════════════════════════════╝
```

**Wizard procedure:** in virt-manager's new-VM wizard, tick **Customize configuration before install** on the final step (CachyOS wiki) — that is where you confirm Q35 + UEFI and set the CPU/disk/NIC values above. If the wizard doesn't autodetect the CachyOS ISO, untick autodetection and select **Arch Linux** as the OS type (CachyOS wiki); this only tunes virtio defaults and has no runtime effect beyond that.

**Why these disk settings:** `virtio-scsi` (not `virtio-blk`) is chosen because the SCSI path carries `discard=unmap`, which lets the guest return freed blocks so the qcow2 stays thin. `cache=writeback` favors throughput and is safe here because the workload is disposable and snapshot-gated (P8); `io=native` pairs with a preallocated or raw-backed image but is fine on qcow2 for this profile. For a guest holding data you cannot lose, prefer `cache=none` + `io=native` and accept lower throughput.

**Why host-passthrough:** it forwards the Zen 5 feature bits (x86-64-v4: AVX-512, etc.) so guest CachyOS selects the same micro-optimized package repos it would on bare metal. The tradeoff is migration: a host-passthrough guest is not portable to a different CPU. That is irrelevant for a single-host sandbox.

**GPU scope:** the guest has **no amdgpu device** — video is virtio-gpu. Any amdgpu/`gfx1151` sysfs on the physical host is not present in the guest and is out of scope for this configuration; only virtio-gpu is exercised here.

**Optional hardening at definition time** (not required for a NAT sandbox, listed for completeness):
- Add a **TPM** device (`swtpm` backend) if the guest OS or a workload wants measured boot — this is the only reason `swtpm` was installed in P2.
- Set the disk **serial** and enable **launchSecurity** (SEV) only if you have a confidential-computing requirement; neither is needed here.

---

## P4 — GUEST OS + VIRTIO TOOLING — RISK: CONFINED TO GUEST

Install CachyOS in the guest with a **systemd-boot** layout, then bring the virtio integration and base tooling up:

```
$ sudo cachyos-rate-mirrors && sudo pacman -Syu   # ranks CachyOS mirrors into /etc/pacman.d/cachyos-mirrorlist (CachyOS FAQ)
$ sudo pacman -S --needed git diffutils fish \
      man-db pacman-contrib \
      qemu-guest-agent spice-vdagent
$ sudo systemctl enable --now qemu-guest-agent
$ sudo systemctl enable --now spice-vdagentd
```

**Virtio integration is the point of this phase:**
- `qemu-guest-agent` — enables host↔guest coordination: quiesced snapshots (P8, `--quiesce`), graceful shutdown from `virsh`, and IP reporting. Bound to the `org.qemu.guest_agent.0` channel defined in P3.
- `spice-vdagent` (+ `spice-vdagentd` service) — clipboard sharing and dynamic display resize over the SPICE channel.
- Package name is `diffutils` (core, 3.12 — verified), not `diff-utils`.

**Guest firmware/boot note:** the systemd-boot layout is chosen so the guest matches a bare-metal CachyOS install; nothing about it is virtio-specific, but keeping guest and host on the same boot manager simplifies muscle memory. Confirm the OVMF firmware set in P3 corresponds to a writable NVRAM varstore per-domain (virt-manager creates one automatically when you pick the UEFI dropdown).

**Post-install virtio verification:**

```
$ systemctl is-active qemu-guest-agent spice-vdagentd
$ lspci | grep -iE 'virtio'          # scsi, net, gpu, rng, balloon
$ lsblk -o NAME,TRAN                 # disk TRAN = virtio
$ ip -br link                        # NIC present, virtio-net
```

On the **host**, confirm the agent handshake:

```
$ virsh domifaddr cachyos-guest --source agent
$ virsh guestinfo cachyos-guest
```

Workload tooling (for the intended audit use only) is added in P5 alongside the workspace, keeping this phase purely about the KVM/guest integration.

---

## P5 — GUEST WORKSPACE PREPARATION — RISK: NONE

Run **before** P6 containment (the clone needs the source host reachable). This phase provisions the workload the sandbox exists for; it is not KVM configuration per se, so it is kept minimal.

```
$ sudo pacman -S --needed code shellcheck ripgrep fd \
      nvme-cli lm_sensors iw
$ git clone https://github.com/ryanmusante/ry-install
$ cd ry-install
$ git config --local user.name  "Sandbox"
$ git config --local user.email "sandbox@kvm.internal"
$ git switch -c sandbox
```

`code`, `shellcheck`, `ripgrep`, `fd` are the review toolchain; `nvme-cli`/`lm_sensors`/`iw` mirror the target script's own package set and runtime probes so its real code branches execute instead of "command absent" fallbacks. `main` HEAD is the working baseline; the `sandbox` branch isolates any edits. Node/npm are intentionally absent — the Claude Code native installer needs neither.

**Claude Code (native installer — auto-updating, per official setup docs):**

```
$ curl -fsSL https://claude.ai/install.sh | bash
$ claude --version && claude doctor
```

Installs to `~/.local/bin`, runs as the regular guest user, and self-updates. Authenticate on first run via browser OAuth in the guest, or with `ANTHROPIC_API_KEY`. Docs: code.claude.com/docs/en/setup. Launch with `--dangerously-skip-permissions` (equivalent to `--permission-mode bypassPermissions`) only inside this isolated guest — it refuses to start as root and must never be used on the host.

---

## P6 — NETWORK CONTAINMENT — RISK: LOW, REVERSIBLE

The containment posture is the second workload-driven configuration constraint. Goal: NAT stays up (Claude Code needs `api.anthropic.com` / `claude.ai` / `console.anthropic.com`); the guest can no longer reach the source host (GitHub) after the clone in P5.

**Backup first, then edit (host):**

```
$ virsh net-dumpxml default > "$HOME/default-net.$(date +%F).xml"
$ sudo virsh net-edit default
```

**Option 1 — libvirt dnsmasq passthrough (preferred; wildcards whole domains).** Namespace available since libvirt 5.6.0 (upstream marks XML namespaces "no support guarantees"):

```xml
<network xmlns:dnsmasq='http://libvirt.org/schemas/network/dnsmasq/1.0'>
  <!-- existing name/uuid/forward/bridge/ip elements unchanged -->
  <dnsmasq:options>
    <dnsmasq:option value='address=/github.com/127.0.0.1'/>
    <dnsmasq:option value='address=/githubusercontent.com/127.0.0.1'/>
    <dnsmasq:option value='address=/github.io/127.0.0.1'/>
  </dnsmasq:options>
</network>
```

Apply — note the operational constraint (verified on libvirt-users): `<dnsmasq:options>` **cannot** be changed via `virsh net-update`, and restarting a network detaches running guests' tap devices. So either shut the guest down first, or restart libvirtd afterward to re-plug taps:

```
$ sudo virsh net-destroy default && sudo virsh net-start default
$ sudo systemctl restart libvirtd
```

**Option 2 — guest-level fallback (`/etc/hosts`, no wildcards):**

```
127.0.0.1 github.com api.github.com codeload.github.com gist.github.com ssh.github.com
127.0.0.1 raw.githubusercontent.com objects.githubusercontent.com gist.githubusercontent.com
```

**Verification (guest):**

```
$ getent hosts github.com
$ git -C ~/ry-install push --dry-run
$ curl -sI https://api.anthropic.com | head -1
```

Expected: `127.0.0.1` · connection failure · any HTTP status line (401/405 = reachable; `anthropic.com` itself redirects, so it is the wrong health check).

**Residual risk:** DNS spoofing does not stop IP-literal remotes or non-GitHub exfil targets. If that matters, switch to `<forward mode='none'/>` plus a host-side nft allowlist to Anthropic endpoints — materially more setup for marginal gain when the workload is your own public repo.

**Rollback:**

```
$ sudo virsh net-define "$HOME/default-net.<date>.xml"
$ sudo virsh net-destroy default && sudo virsh net-start default
```

---

## P7 — PERSISTENCE + AUTOSTART — RISK: LOW

Make the configured environment survive host reboots deterministically, so the sandbox is reproducible rather than hand-rebuilt each session.

```
$ virsh autostart cachyos-guest          # domain starts with libvirtd
$ virsh net-info default | grep Autostart # network already yes (P2)
$ virsh dominfo cachyos-guest            # confirm Autostart: enable
```

**Considerations:**
- Domain autostart is optional and off by default. Enable it only if you want the sandbox available immediately after a host boot; leave it off to keep the guest dormant until explicitly started with `virsh start cachyos-guest`.
- The NAT network's autostart was already set in P2; a guest with autostart enabled but its network not autostarting will fail to obtain connectivity at boot, so keep both consistent.
- To pin the guest to a subset of host CPUs (isolating the remaining threads for the host), add `<vcpu placement='static' cpuset='0-15'/>` in `virsh edit`; this is optional tuning, not required for correctness.

**Rollback:** `virsh autostart --disable cachyos-guest` (and `virsh net-autostart --disable default` if you no longer want the network to auto-start).

---

## P8 — SNAPSHOT BASELINE — RISK: NONE (PROTECTIVE)

On the **host**, before any live execution of a workload inside the guest, take a clean baseline you can revert to:

```
$ virsh snapshot-create-as --domain cachyos-guest --name pre-run-baseline \
      --description "clean configured guest, pre-workload" --live --quiesce
$ virsh snapshot-list cachyos-guest
```

With `qemu-guest-agent` running (P4), `--live --quiesce` gives a consistent snapshot without shutdown; the agent flushes and freezes guest filesystems for the instant of capture. Without the agent, snapshot a powered-off guest instead (drop `--live --quiesce` and stop the domain first). This snapshot is the single most important protective step: everything the workload does afterward is reversible to this point (P9).

---

## P9 — RECOVERY PROCEDURES — REFERENCE

```
╔═══════════════════════════╦══════════════════════════════════╗
║ FAILURE                   ║ ACTION                           ║
╠═══════════════════════════╬══════════════════════════════════╣
║ Guest OS state corrupted  ║ virsh snapshot-revert --domain   ║
║ by workload execution     ║ cachyos-guest --snapshotname     ║
║                           ║ pre-run-baseline                 ║
║ Guest workspace mangled   ║ git reset --hard &&              ║
║                           ║ git clean -fd  (guest)           ║
║ Containment misapplied /  ║ Restore net XML backup (P6       ║
║ NAT broken                ║ rollback)                        ║
║ Autostart wedged at boot  ║ virsh autostart --disable        ║
║                           ║ cachyos-guest; start manually    ║
║ Whole environment suspect ║ virsh destroy + undefine, then   ║
║                           ║ rebuild from P3 (definition is   ║
║                           ║ the only irreplaceable artifact) ║
╚═══════════════════════════╩══════════════════════════════════╝
```

---

## INTENDED WORKLOAD — REFERENCE ONLY

The environment above is general-purpose; it was sized and contained for a specific job, recorded here so the P3/P6 choices have context. This is not part of the KVM configuration and can be ignored if you repurpose the guest.

The guest is used to review `ry-install.fish` (4,952 lines) with Claude Code, reading the checkout directly rather than pasting the file — reference paths and line ranges. A single review sweep looks at variable scoping (`set -l` discipline in nested blocks), Fish 1-based array/index semantics, error propagation (`$status` / `; or return 1`, `argparse` flag definitions against the Fish 3.6+ compat floor), and static cross-referencing of the script's kernel-parameter assignments against `gfx1151` / mainline `amdgpu` definitions. The last of these is **static-only** in this guest — there is no amdgpu device, so any functional sysfs validation is bare-metal-only and outside this environment's remit. None of that changes the guest's configuration; it only justifies the vCPU/RAM sizing (P3) and the "reach Anthropic, not GitHub" containment (P6).
