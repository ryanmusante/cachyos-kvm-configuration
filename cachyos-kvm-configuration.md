# CachyOS KVM Configuration — Isolated Linux Guest on a GTR9 Pro Host

**Version:** 6.4.0 · **Date:** 2026-09-27

A risk-ordered procedure for building, containing and operating an isolated CachyOS KVM guest on a Beelink GTR9 Pro without exposing the physical host. The subject is **KVM configuration**; the guest's workload appears only as the reason for its sizing and containment.

- **Host:** Beelink GTR9 Pro — AMD Ryzen AI Max+ 395 (16 C / 32 T Zen 5, x86-64-v4), 128 GiB LPDDR5X unified memory, Radeon 8060S (RDNA 3.5, `gfx1151`); CachyOS, rolling. The OS sees 128 GiB minus the BIOS UMA carveout; all sizing below derives from these figures.
- **Guest:** CachyOS, virtio devices only, systemd-boot layout.
- **Workload (context only):** reviewing the Fish installer `ryanmusante/ry-install` and its verifier `ryanmusante/ry-verify` with Claude Code — see [Intended workload](#intended-workload). It sets two constraints and nothing more: vCPU/RAM headroom for an editor plus an AI agent (P3), and containment that keeps Anthropic reachable while cutting the guest off from GitHub after the clone (P6).

**Conventions:** every shell block pastes unchanged into bash or fish (the CachyOS default shell) and passes `bash -n`, `shellcheck` and `fish --no-execute`; XML blocks pass `xmllint`. Once P2 is done, `virsh` without `-c` addresses `qemu:///system`.

**Sources** (checked 2026-09-27): Arch package database (versions, dependency chains, optdepends, file lists, unit files); libvirt 12.7.0 source and documentation (daemon activation, default-URI probing, network forward modes, dnsmasq namespace, disk validation, SPICE agent features, snapshot and undefine rules); QEMU documentation and source (cache modes, virtio queue defaults); Linux 7.2 (`/dev/kvm` registration, `kvm_amd` defaults); systemd (`/dev/kvm` mode); util-linux (`lsblk` transport); dnsmasq(8) (`address=`); CachyOS wiki and FAQ (host stack, VM wizard, mirror ranking, optimized repositories); Claude Code documentation (setup, network access requirements, permission modes); the two workload repositories' `main`.

---

## Phase map

Execution order is dependency order: read-only checks first, then host changes, then the VM and guest, and a protective snapshot before any live workload. Every mutating phase states its rollback before it runs. P0–P7 are the KVM configuration proper; P8 gates any live workload; P9 is recovery reference.

```text
╔════╦═════════════════════╦══════════════╦═════════════════╗
║ PH ║ PHASE               ║ RISK         ║ ROLLBACK        ║
╠════╬═════════════════════╬══════════════╬═════════════════╣
║ P0 ║ KVM prerequisites   ║ none         ║ n/a             ║
║    ║                     ║ (read-only)  ║                 ║
╟────╫─────────────────────╫──────────────╫─────────────────╢
║ P1 ║ Host update hygiene ║ low          ║ pacman -U from  ║
║    ║                     ║              ║ cache           ║
╟────╫─────────────────────╫──────────────╫─────────────────╢
║ P2 ║ Host virtualization ║ low          ║ reverse block   ║
║    ║ stack               ║              ║                 ║
╟────╫─────────────────────╫──────────────╫─────────────────╢
║ P3 ║ VM definition       ║ none         ║ undefine domain ║
╟────╫─────────────────────╫──────────────╫─────────────────╢
║ P4 ║ Guest OS and virtio ║ guest-only   ║ reinstall guest ║
║    ║ integration         ║              ║                 ║
╟────╫─────────────────────╫──────────────╫─────────────────╢
║ P5 ║ Guest workspace     ║ guest-only   ║ delete clones   ║
║    ║ preparation         ║              ║                 ║
╟────╫─────────────────────╫──────────────╫─────────────────╢
║ P6 ║ Network containment ║ low          ║ restore net XML ║
╟────╫─────────────────────╫──────────────╫─────────────────╢
║ P7 ║ Persistence and     ║ low          ║ reverse block   ║
║    ║ autostart           ║              ║                 ║
╟────╫─────────────────────╫──────────────╫─────────────────╢
║ P8 ║ Snapshot baseline   ║ none         ║ n/a             ║
║    ║                     ║ (protective) ║                 ║
╟────╫─────────────────────╫──────────────╫─────────────────╢
║ P9 ║ Recovery procedures ║ reference    ║ n/a             ║
╚════╩═════════════════════╩══════════════╩═════════════════╝
```

---

## P0 — KVM prerequisites

**Risk:** none (read-only) · **Rollback:** n/a

Nothing here changes state — confirm the host can carry the guest before installing anything.

```sh
lscpu | grep -qw svm && echo "AMD-V: present"
grep -qw kvm /proc/misc && echo "/dev/kvm: present"
zgrep -E 'CONFIG_KVM_AMD=[ym]' /proc/config.gz   # y = built in, m = module
uname -r && pacman -Q linux-firmware
df -h /var/lib                                  # filesystem for the image pool
free -h                                         # OS-visible memory
```

Expect:

- `svm` in the CPU flags — AMD-V; KVM cannot start without it.
- `/dev/kvm` present — `kvm_amd` registers it through KVM's `kvm_init()` as it loads.
- `CONFIG_KVM_AMD` set to `y` or `m`.
- A current CachyOS kernel and `linux-firmware`.
- At least 70 GiB free where `/var/lib/libvirt/images` will live — the 60 GiB qcow2 is thin-provisioned and grows toward that ceiling — plus room for the install ISO.
- A `free -h` total comfortably above the 32 GiB guest. The OS sees 128 GiB minus the BIOS UMA carveout, and host GPU work (GTT) draws on the same pool.

Two firmware toggles are prerequisites rather than software steps — check them in UEFI setup if `/dev/kvm` is missing:

```text
╔══════════════════╦═══════════════════════════════════╗
║ SETTING          ║ REQUIRED STATE                    ║
╠══════════════════╬═══════════════════════════════════╣
║ SVM Mode (AMD-V) ║ Enabled - hardware virtualization ║
╟──────────────────╫───────────────────────────────────╢
║ IOMMU (AMD-Vi)   ║ Enabled - needed only for device  ║
║                  ║ passthrough; harmless for virtio  ║
╚══════════════════╩═══════════════════════════════════╝
```

Nested virtualization (KVM inside this guest) is out of scope. `kvm_amd` already defaults to `nested=1` (`/sys/module/kvm_amd/parameters/nested`), so a host-passthrough guest sees `svm`; nothing needs setting.

---

## P1 — Host update hygiene

**Risk:** low · **Rollback:** `pacman -U` with the prior package from `/var/cache/pacman/pkg/`

```sh
curl -s https://cachyos.org/rss.xml | grep -oP '(?<=<title>)[^<]+' | head -n 6
sudo pacman -Syu
pacdiff
```

Read the headlines before `-Syu` for breaking-change notices — the feed title plus the five newest posts, which render at `cachyos.org/blog` (`cachyos.org/news` serves the home page). Run `pacdiff` afterwards to reconcile `.pacnew` files. Full `-Syu` only, never a partial upgrade. A current rolling kernel means a current KVM/virtio stack, so do not pin LTS for this host role.

---

## P2 — Host virtualization stack

**Risk:** low · **Rollback:** reverse block at the end of this phase

```sh
sudo pacman -S --needed qemu-full virt-manager swtpm dnsmasq
grep -q '^firewall_backend' /etc/libvirt/network.conf \
    || echo 'firewall_backend = "iptables"' | sudo tee -a /etc/libvirt/network.conf
sudo systemctl enable --now libvirtd.socket
sudo virsh net-autostart default
sudo virsh net-list --inactive --name | grep -qx default && sudo virsh net-start default
sudo usermod -aG libvirt "$USER"
mkdir -p "$HOME/.config/libvirt"
grep -q '^uri_default' "$HOME/.config/libvirt/libvirt.conf" 2>/dev/null \
    || echo 'uri_default = "qemu:///system"' >> "$HOME/.config/libvirt/libvirt.conf"
```

**Package set and provenance** (Arch package database, 2026-09-27: `qemu-full` 11.1.1, `libvirt` 1:12.7.0, `virt-manager` 5.1.0, `edk2-ovmf` 202608, `swtpm` 0.10.2, `dnsmasq` 2.93):

- `libvirt` arrives with **`virt-manager`** (via `virt-install` → `libvirt-python`, and via `libvirt-glib`), not with `qemu-full`.
- `qemu-full` pulls **`edk2-ovmf`** for UEFI guest firmware (via `qemu-desktop` → `qemu-base` → `qemu-system-x86`).
- `swtpm` supplies an emulated TPM — used only if you add one (P3); a Linux guest does not need it.
- `dnsmasq` is listed because libvirt declares it only as an optional dependency ("required for default NAT/DHCP for guests").
- libvirt's other NAT optional dependency, `iptables-nft`, is already met: core `iptables` (1:1.8.13) provides and replaces it, and it ships in a standard CachyOS install.

**Daemon model:** `libvirtd.socket` starts the daemon on the first client connection (CachyOS wiki). The wiki presents `libvirtd.service` as the optional LXC backend; it is also what starts the daemon at boot, which domain autostart needs — P7 enables it.

**Firewall backend:** current libvirt defaults to `nftables`; the CachyOS-tested value is `iptables`, set in `/etc/libvirt/network.conf`. It governs how libvirt programs the NAT rules for `virbr0`.

**NAT network:** the package defines the `default` network without autostart. `net-autostart` starts it whenever the daemon starts; the guarded `net-start` brings it up now, because the daemon was already running when the flag was set.

**Firewall interaction:** on a host running **ufw**, also permit guest transit with `sudo ufw route allow from 192.168.122.0/24` (libvirt's default subnet; CachyOS wiki). Without ufw, libvirt's own rules suffice, and NAT needs no `nftables.service`.

**No global dnsmasq:** libvirt spawns its own per-network dnsmasq on `virbr0`; a global instance on `0.0.0.0:53` conflicts unless it uses `bind-interfaces`/`except-interface` (Arch wiki, Libvirt). Install the package; do not enable its service.

**Group model:** log out and back in (or `newgrp libvirt` per shell) after `usermod`; no reboot and no `kvm` group step. System-mode guests run as libvirt's own qemu user, so your account needs only `libvirt` for socket access (CachyOS wiki; Arch wiki, Libvirt).

**Connection URI — why the last two lines exist:** libvirt probes to `qemu:///system` only for a privileged caller; anyone else gets `qemu:///session` (`src/qemu/qemu_conf.c`), which holds none of the domains or networks virt-manager creates on the system connection, so an unqualified `virsh dominfo` would report "domain not found". `uri_default` in `$XDG_CONFIG_HOME/libvirt/libvirt.conf` (default `~/.config`) settles that once for unprivileged users; per shell, `LIBVIRT_DEFAULT_URI=qemu:///system` does the same (fish: `set -Ux LIBVIRT_DEFAULT_URI qemu:///system`). Both `grep` guards keep a re-run from duplicating a line.

**Verification** (after the re-login):

```sh
systemctl is-active libvirtd.socket
virsh -c qemu:///system version
virsh net-list --all   # default: active, autostart yes
ls -l /dev/kvm         # crw-rw-rw- root kvm (systemd dev-kvm-mode 0666)
```

**Rollback** (reverse order):

```sh
sed -i '/^uri_default = "qemu:\/\/\/system"$/d' "$HOME/.config/libvirt/libvirt.conf"
sudo gpasswd -d "$USER" libvirt
sudo virsh net-autostart --disable default
sudo systemctl disable --now libvirtd.service libvirtd.socket
sudo sed -i '/^firewall_backend = "iptables"$/d' /etc/libvirt/network.conf
sudo pacman -Rns qemu-full virt-manager swtpm dnsmasq
```

---

## P3 — VM definition

**Risk:** none · **Rollback:** `virsh undefine cachyos-guest --nvram --remove-all-storage`

The core of the KVM configuration, built in virt-manager. Each value suits a modern Linux guest on this silicon; the notes below explain the choices that matter.

```text
╔══════════════════╦═══════════════════════════════════════════╗
║ SETTING          ║ VALUE                                     ║
╠══════════════════╬═══════════════════════════════════════════╣
║ Name             ║ cachyos-guest - every virsh command       ║
║                  ║ below uses this name                      ║
╟──────────────────╫───────────────────────────────────────────╢
║ Chipset          ║ Q35 - native PCIe; i440FX only for        ║
║                  ║ legacy guests (CachyOS wiki)              ║
╟──────────────────╫───────────────────────────────────────────╢
║ Firmware         ║ UEFI: /usr/share/edk2/x64/OVMF_CODE.4m.fd ║
║                  ║ (edk2-ovmf 202608, pulled by qemu-full);  ║
║                  ║ pick UEFI in virt-manager's firmware menu ║
╟──────────────────╫───────────────────────────────────────────╢
║ CPU model        ║ host-passthrough - full Zen 5 feature     ║
║                  ║ set (x86-64-v4); see the note below       ║
╟──────────────────╫───────────────────────────────────────────╢
║ vCPU topology    ║ 16 vCPUs = 1 socket / 8 cores / 2 threads ║
║                  ║ (compile-heavy work: 24 = 1 / 12 / 2);    ║
║                  ║ the host keeps the rest of its 32 threads ║
╟──────────────────╫───────────────────────────────────────────╢
║ Memory           ║ 32 GiB fixed; the default virtio-balloon  ║
║                  ║ device stays in place                     ║
╟──────────────────╫───────────────────────────────────────────╢
║ Disk bus         ║ SCSI on a virtio-scsi controller, not     ║
║                  ║ virtio-blk (see the note below)           ║
╟──────────────────╫───────────────────────────────────────────╢
║ Disk cache/IO    ║ cache=writeback, discard=unmap,           ║
║                  ║ io=threads; 60 GiB qcow2                  ║
╟──────────────────╫───────────────────────────────────────────╢
║ NIC              ║ virtio on network 'default' (NAT)         ║
╟──────────────────╫───────────────────────────────────────────╢
║ Video            ║ virtio-gpu, 3D off; no host GPU           ║
║                  ║ passthrough                               ║
╟──────────────────╫───────────────────────────────────────────╢
║ Guest agent      ║ virtio channel org.qemu.guest_agent.0     ║
╟──────────────────╫───────────────────────────────────────────╢
║ RNG              ║ virtio-rng backed by /dev/urandom         ║
╟──────────────────╫───────────────────────────────────────────╢
║ Clipboard/resize ║ SPICE + spice-vdagent (P4)                ║
╚══════════════════╩═══════════════════════════════════════════╝
```

**Wizard procedure:** in virt-manager's new-VM wizard, name the guest `cachyos-guest` on the last step and tick **Customize configuration before install**; confirm Q35 and UEFI under Overview, then set the values above (CachyOS wiki). If the CachyOS ISO is not autodetected, untick autodetection and choose **Arch Linux** (CachyOS wiki) — the OS choice only steers device defaults.

**Why this disk setup:** virtio-scsi and virtio-blk both carry discard, and current QEMU sizes both to one queue per vCPU by default, so neither trim nor multi-queue decides it. virtio-scsi is kept because one controller carries any number of disks, where virtio-blk spends a PCI device per disk. `discard=unmap` passes the guest's trims through, so the qcow2 stays thin. `cache=writeback` is QEMU's default: writes complete once they reach the host page cache, which is safe for a guest that flushes correctly, as modern Linux filesystems do (QEMU documentation).

**Cache and AIO must pair correctly:** libvirt rejects `io='native'` unless the cache mode is `none` or `directsync`, so a `cache=writeback` + `io=native` disk is refused. Valid pairings: `cache=writeback` + `io=threads` (shipped above); `cache=none` + `io=native`, which bypasses the host page cache so guest data is not cached twice; or `io=io_uring` with any cache mode (Arch's QEMU links liburing).

**Why host-passthrough:** it forwards the full Zen 5 feature set (x86-64-v4, AVX-512), so the guest qualifies for the same optimized CachyOS repositories as bare metal — `x86-64-v4`, or `znver4`, which is built for Zen 4 and Zen 5 (CachyOS wiki, Optimized Repositories). Qualifying is not selecting: the installer picks the repository set, and moving an existing install between `x86-64-v3`, `x86-64-v4` and `znver4` is a manual `/etc/pacman.conf` edit; v4 and znver4 share `/etc/pacman.d/cachyos-v4-mirrorlist`. The trade-off is migration: a host-passthrough guest cannot move to a different CPU, which is irrelevant for a single-host sandbox.

**GPU scope:** the guest has no amdgpu device and no host `gfx1151` sysfs; video is virtio-gpu with 3D off. The wiki's 3D options (EGL headless, Venus) stay off here: both hand guest GPU commands to the host GPU stack, which this sandbox avoids exposing.

**Optional hardening at definition time** (not required for a NAT sandbox):

- **TPM:** add a TPM device (`swtpm` backend) only if the guest OS or a workload wants measured boot — the only reason `swtpm` is installed in P2.
- **SEV (`launchSecurity`):** applicable only where the host reports support (`/sys/module/kvm_amd/parameters/sev` reads `Y`); not needed here.
- **SPICE agent channels:** clipboard sharing and file transfer are on by default, so while the console is open, whatever the host copies becomes readable in the guest session — where the agent runs with permission prompts bypassed (P5). To close both, add the two children below to the existing SPICE element with `virsh edit cachyos-guest`; clipboard sharing from P4 then stops.

```xml
<graphics type='spice' autoport='yes'>
  <!-- existing children unchanged -->
  <clipboard copypaste='no'/>
  <filetransfer enable='no'/>
</graphics>
```

---

## P4 — Guest OS and virtio integration

**Risk:** guest-only · **Rollback:** reinstall the guest

Install CachyOS in the guest with a **systemd-boot** layout, then bring up the virtio integration and base tooling:

```sh
sudo cachyos-rate-mirrors && sudo pacman -Syu   # ranks CachyOS mirrors (CachyOS FAQ)
sudo pacman -S --needed man-db pacman-contrib qemu-guest-agent spice-vdagent
sudo systemctl start qemu-guest-agent spice-vdagentd
```

- `qemu-guest-agent` — host↔guest coordination: quiesced snapshots (P8, `--quiesce`), agent-mode shutdown, IP reporting. It serves the `org.qemu.guest_agent.0` channel from P3.
- `spice-vdagent` — clipboard sharing and dynamic display resize over SPICE: the `spice-vdagentd` daemon plus a per-session agent that the desktop autostarts.
- `man-db` and `pacman-contrib` (`pacdiff`, `paccache`) — base tooling for maintaining the guest.

Neither unit can be enabled — their `[Install]` sections are empty or absent — because udev starts both whenever their virtio port appears (`99-qemu-guest-agent.rules`, `70-spice-vdagentd.rules`). `start` covers only the current boot.

**Boot layout:** systemd-boot matches the host's boot manager; nothing about it is virtio-specific. The UEFI firmware from P3 gets a per-domain writable NVRAM varstore, which virt-manager creates when UEFI is selected.

**Verification (guest):**

```sh
systemctl is-active qemu-guest-agent spice-vdagentd
lspci | grep -i virtio                        # scsi, net, gpu, console, rng, balloon
lsblk -o NAME,TYPE                            # disk is sda (virtio-scsi), not vda
grep . /sys/class/scsi_host/host*/proc_name   # one host reads virtio_scsi
readlink /sys/class/net/*/device/driver       # NIC bound to virtio_net
```

`lsblk`'s `TRAN` column is deliberately not the check: it reports `virtio` only for `vd*` names (virtio-blk), so a virtio-scsi disk appears as `sd*` with an empty transport (util-linux `lsblk`). The SCSI host's `proc_name` is the direct evidence — globbed rather than `host0`, because the SATA CD-ROM's AHCI controller registers its own SCSI host and may take the lower number.

**Verification (host)** — the agent handshake:

```sh
virsh domifaddr cachyos-guest --source agent
virsh guestinfo cachyos-guest
```

Workload tooling arrives in P5, keeping this phase to the guest OS and its virtio integration.

---

## P5 — Guest workspace preparation

**Risk:** guest-only · **Rollback:** delete the clones

Run **before** P6 — the clones need GitHub reachable. This phase provisions the workload the sandbox exists for; it is not KVM configuration, so it stays minimal.

```sh
sudo pacman -S --needed git diffutils fish code shellcheck ripgrep fd \
    nvme-cli lm_sensors iw
git clone -c user.name=Sandbox -c user.email=sandbox@kvm.internal \
    https://github.com/ryanmusante/ry-install ~/ry-install
git clone -c user.name=Sandbox -c user.email=sandbox@kvm.internal \
    https://github.com/ryanmusante/ry-verify ~/ry-verify
git -C ~/ry-install switch -c sandbox
git -C ~/ry-verify switch -c sandbox
```

`git`, `diffutils` and `fish` run the workload (both scripts are Fish); `code`, `shellcheck`, `ripgrep` and `fd` are the review toolchain; `nvme-cli` and `lm_sensors` belong to the installer's own package set and `iw` backs its Wi-Fi probes, so its real code paths run instead of "command absent" fallbacks. `-c` writes a local identity into each new repository, and the `sandbox` branch keeps edits off `main`. Node and npm stay absent — the native Claude Code installer needs neither.

**Claude Code** (native installer, auto-updating):

```sh
curl -fsSL https://claude.ai/install.sh | bash
claude --version && claude doctor
```

The installer manages the launcher at `~/.local/bin/claude`, a symlink into `~/.local/share/claude/versions/`; run it as the regular guest user. Sign in on first launch through the browser, or approve a set `ANTHROPIC_API_KEY` once. Launch with `--dangerously-skip-permissions` (equivalent to `--permission-mode bypassPermissions`) only inside this guest: it removes permission prompts, so containment (P6) and the baseline snapshot (P8) are the safety net. The first interactive launch asks for a one-time acceptance, and Claude Code refuses the mode as root or under sudo. Docs: `code.claude.com/docs/en/setup`, `code.claude.com/docs/en/permission-modes`.

---

## P6 — Network containment

**Risk:** low · **Rollback:** restore the network XML backup (block at the end of this phase)

The second workload-driven constraint. NAT stays up, so Claude Code keeps its endpoints, while the guest loses the source host (GitHub) once P5 has cloned. The endpoints behind model traffic, sign-in and updates include no GitHub domain (Claude Code docs, network access requirements); the accepted collateral below covers the two GitHub-hosted features:

```text
╔═══════════════════════╦═════════════════════════════════════╗
║ ENDPOINT              ║ USED FOR                            ║
╠═══════════════════════╬═════════════════════════════════════╣
║ api.anthropic.com     ║ API requests                        ║
╟───────────────────────╫─────────────────────────────────────╢
║ claude.ai, claude.com ║ claude.ai account sign-in           ║
╟───────────────────────╫─────────────────────────────────────╢
║ platform.claude.com   ║ Console sign-in; OAuth token        ║
║                       ║ exchange and refresh (all accounts) ║
╟───────────────────────╫─────────────────────────────────────╢
║ downloads.claude.ai   ║ native installer, auto-updater,     ║
║                       ║ update checks                       ║
╚═══════════════════════╩═════════════════════════════════════╝
```

**Backup, then edit (host):**

```sh
virsh net-dumpxml --inactive default > "$HOME/default-net.$(date +%F).xml"
virsh net-edit default
```

**Option 1 — libvirt dnsmasq passthrough (preferred; wildcards whole domains).** The namespace exists since libvirt 5.6.0, and upstream gives XML namespaces no support guarantee. An `address=` domain is answered locally and never forwarded — AAAA as well as A, per dnsmasq(8) — so the block cannot leak over IPv6:

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

**Apply:** `<dnsmasq:options>` cannot be changed with `virsh net-update`, and restarting the network detaches running guests' tap devices (libvirt-users). Shut the guest down first, or restart the daemon afterwards so it re-attaches them:

```sh
virsh net-destroy default && virsh net-start default
sudo systemctl restart libvirtd.service   # only if the guest was running
```

**Option 2 — guest-level fallback (`/etc/hosts`, no wildcards):**

```text
127.0.0.1 github.com api.github.com codeload.github.com gist.github.com ssh.github.com
127.0.0.1 raw.githubusercontent.com objects.githubusercontent.com gist.githubusercontent.com
```

**Accepted collateral:** blocking GitHub also disables two Claude Code features — the changelog feed on `raw.githubusercontent.com` (`/release-notes`, plus a startup fetch after an update) and GitHub-hosted plugin marketplaces on `github.com`, including the official Anthropic marketplace. Model traffic, sign-in and updates use no GitHub domain.

**Verification (guest):** if the guest resolves through systemd-resolved, run `sudo resolvectl flush-caches` first, so a cached pre-block answer cannot mask the result.

```sh
getent hosts github.com
git -C ~/ry-install ls-remote origin
curl -sI https://api.anthropic.com | head -n 1
```

Expected: `127.0.0.1`; a connection failure; any HTTP status line, which proves reachability rather than authorization (`anthropic.com` itself redirects, so it is the wrong probe). `ls-remote` is the probe because it opens a connection; `push --dry-run` aborts locally on the upstream-less `sandbox` branch.

**Residual risk:** DNS blocking does not stop IP-literal remotes or non-GitHub destinations. If that matters, enforce egress on the host instead — an nftables allowlist on traffic forwarded from `virbr0`, or an isolated network (no `<forward>` element) behind an allowlisting proxy on the host. Both are materially more setup for marginal gain when the workload is your own public repository.

**Rollback** (same tap caveat as Apply):

```sh
virsh net-define "$HOME/default-net.<date>.xml"
virsh net-destroy default && virsh net-start default
```

---

## P7 — Persistence and autostart

**Risk:** low · **Rollback:** reverse block at the end of this phase

Make the configured environment survive host reboots deterministically. Domain autostart fires when the libvirt daemon starts, and a socket-activated daemon starts on the first client connection, not at boot — so boot-time autostart also needs the service enabled (libvirt daemons documentation).

```sh
sudo systemctl enable libvirtd.service
virsh autostart cachyos-guest
virsh net-info default | grep Autostart        # yes (set in P2)
virsh dominfo cachyos-guest | grep Autostart   # enable
```

**Considerations:**

- Domain autostart is optional and off by default. Skip this phase to keep the guest dormant until `virsh start cachyos-guest`.
- A domain whose network is inactive fails to start, so keep the network's autostart (P2) on while domain autostart is on.
- CPU pinning is optional tuning, not needed for correctness: `<vcpu placement='static' cpuset='0-7,16-23'>16</vcpu>` in `virsh edit cachyos-guest` keeps the 16 vCPUs on eight whole cores, matching the 8-core/2-thread topology. The example assumes SMT siblings enumerate as N and N+16 — confirm with `lscpu -e=CPU,CORE,CACHE` (the last CACHE field is the L3, one per CCD).

**Rollback:** disabling the service also disables the sockets in its `Also=` list, so socket activation is re-enabled afterwards:

```sh
virsh autostart --disable cachyos-guest
sudo systemctl disable libvirtd.service
sudo systemctl enable libvirtd.socket
```

---

## P8 — Snapshot baseline

**Risk:** none (protective) · **Rollback:** n/a

On the **host**, before the first live run of any workload in the guest, take a baseline you can revert to; everything the workload does afterwards is reversible to this point (P9). Save the domain definition first — P9's rebuild path relies on it.

**Preferred — powered-off internal snapshot.** It lives inside the qcow2 and reverts with no external files to track. `virsh shutdown` returns as soon as the request is sent, so wait for `shut off`: libvirt refuses an internal snapshot of the running UEFI guest while its NVRAM varstore is raw. The wait is bounded at two minutes — `timeout` exits 124 if the guest never stops, and a wedged guest needs `virsh destroy cachyos-guest` before the snapshot. An offline internal snapshot does not capture the UEFI varstore (libvirt `qemu_snapshot.c`), so boot entries are left as they are on revert.

```sh
virsh dumpxml --inactive cachyos-guest > "$HOME/cachyos-guest.$(date +%F).xml"
virsh shutdown cachyos-guest
timeout 120 sh -c 'until virsh domstate cachyos-guest | grep -qx "shut off"; do sleep 2; done'
virsh domstate cachyos-guest   # must read: shut off
virsh snapshot-create-as --domain cachyos-guest --name pre-run-baseline \
    --description "clean configured guest, pre-workload"
virsh snapshot-list cachyos-guest
```

**Alternative — running guest, quiesced disk-only snapshot.** With `qemu-guest-agent` running (P4), the agent flushes and freezes the guest filesystems for the instant of capture, so no shutdown is needed:

```sh
virsh snapshot-create-as --domain cachyos-guest --name pre-run-baseline \
    --description "clean configured guest, pre-workload" \
    --disk-only --quiesce --atomic
```

**Flag constraints** — enforced by libvirt, not advisory:

- `--quiesce` requires `--disk-only`; alone it is refused, and it fails outright without a running guest agent.
- `--live` is accepted only for a full-system external snapshot, i.e. together with `--memspec`; it cannot combine with the internal snapshot above.
- The disk-only form creates an external overlay and saves no memory state. libvirt deletes external snapshots from 9.0.0 and reverts to them from 9.9.0 (libvirt NEWS).

---

## P9 — Recovery procedures

**Risk:** reference · **Rollback:** n/a

```text
╔═══════════════════════════╦══════════════════════════════════════╗
║ FAILURE                   ║ ACTION                               ║
╠═══════════════════════════╬══════════════════════════════════════╣
║ Guest OS corrupted by     ║ virsh snapshot-revert --domain       ║
║ the workload              ║ cachyos-guest --snapshotname         ║
║                           ║ pre-run-baseline                     ║
╟───────────────────────────╫──────────────────────────────────────╢
║ Workspace mangled         ║ git reset --hard && git clean -fd    ║
║ (either checkout)         ║ in that checkout (guest)             ║
╟───────────────────────────╫──────────────────────────────────────╢
║ Containment misapplied    ║ P6 rollback: restore the network     ║
║ or NAT broken             ║ XML backup                           ║
╟───────────────────────────╫──────────────────────────────────────╢
║ Autostart wedged at boot  ║ virsh autostart --disable            ║
║                           ║ cachyos-guest; start it manually     ║
╟───────────────────────────╫──────────────────────────────────────╢
║ Whole environment suspect ║ virsh destroy cachyos-guest, then    ║
║                           ║ virsh undefine cachyos-guest --nvram ║
║                           ║ --snapshots-metadata                 ║
║                           ║ --remove-all-storage; rebuild from   ║
║                           ║ P3 (the P8 XML backup keeps the      ║
║                           ║ definition)                          ║
╚═══════════════════════════╩══════════════════════════════════════╝
```

---

## Intended workload

Reference only — not part of the KVM configuration; ignore it if you repurpose the guest. The environment was sized and contained for this job, recorded here so the P3 and P6 choices have context.

Two repositories, both v7.219.0 at `main` on 2026-09-27: `ryanmusante/ry-install` (`ry-install.fish`, 3,479 lines, 209 functions) and `ryanmusante/ry-verify` (`ry-verify.fish`, 3,099 lines, 232 functions). The figures are a rolling pin — `wc -l` and column-0 `function` definitions — and any upstream push moves them.

Claude Code reads each checkout in place and cites paths and line ranges instead of pasting files. A review sweep covers variable scoping (`set -l` discipline in nested blocks), Fish's 1-based indexing, error propagation (`$status`, `; or return 1`, `argparse` flag specs against the Fish 3.6+ floor), the installer's kernel-parameter assignments against mainline documentation, and the scripts' amdgpu/`gfx1151` sysfs checks. Those sysfs checks are static-only here: the guest has no amdgpu device, so functional validation belongs on bare metal. None of this changes the guest's configuration; it only justifies the vCPU/RAM sizing (P3) and the containment (P6).
