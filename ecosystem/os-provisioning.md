# OS Installation & Bare-Metal Provisioning Research

**Last verified:** 2026-11-26  
**Scope:** Multi-OS installers, universal boot media, network provisioning servers, OS catalogs, and cross-OS installation mechanics from a Linux management environment (`sbm-core`).

---

## Summary Comparison Matrix

| Project | Primary Language(s) | License (SPDX) | Maintenance (Nov 2026) | Verdict | Key Takeaway for SBM |
|---|---|---|---|---|---|
| **PINN** | C++ (Qt), Shell | `BSD-2-Clause` | Active | **Study** | Best UX precedent for multi-OS catalog + partition + install on a single device |
| **NOOBS** | C++ (Qt), Shell | `BSD-2-Clause` | Archived / Deprecated | **Avoid** | Replaced by Raspberry Pi Imager and PINN |
| **Ventoy** | C (GRUB2/EDK2 fork + blobs) | `GPL-3.0-or-later` (with opaque binary blobs) | Active (slower cadence) | **Study / Avoid embedding** | Clever ISO loopback/ACPI hooks, but opaque prebuilt blobs & GPL-3.0 create supply-chain/licensing risk |
| **iVentoy** | Closed-source binary | Proprietary | Active | **Avoid** | Proprietary network PXE server; study UX only |
| **netboot.xyz** | Jinja2/Ansible, iPXE scripts | `Apache-2.0` | Very active | **Study** | Clean declarative OS catalog (`endpoints.yml`) and mirror verification model |
| **Canonical MAAS** | Python, Go, TypeScript | `AGPL-3.0-only` | Very active | **Study** | Strong bare-metal state machine & Curtin image deployment model; too heavy for consumer/self-hosted single-binary server |
| **Tinkerbell** | Go | `Apache-2.0` | Active (CNCF Sandbox) | **Study** | Clean composable workflow actions (`HookOS` in-RAM Linux worker) — closely mirrors `sbm-core` |
| **FOG Project** | PHP, Shell, C | `GPL-3.0-only` | Active | **Study / Avoid** | Proves Partclone-based disk imaging over HTTP/NFS works reliably at scale |
| **OpenStack Ironic** | Python | `Apache-2.0` | Very active | **Study** | Mature hardware inspection & Redfish/BMC abstraction patterns (`ironic-python-agent`) |
| **Foreman** | Ruby (Rails), JS | `GPL-3.0-or-later` | Active | **Avoid** | Enterprise Puppet/Katello stack; mismatched complexity for SBM |
| **Calamares** | C++ (Qt6), Python | `GPL-3.0-or-later` | Very active | **Study** | Strong interactive partitioning/locale/user-setup UX; GPL-3.0/Qt6 prevents direct embedding in Rust core |
| **`bootc` / `ostree`** | Rust (`bootc`), C (`libostree`) | `MIT OR Apache-2.0` (`bootc`) / `LGPL-2.0-or-later` (`ostree`) | Very active (CNCF Sandbox) | **Reuse / Integrate** | Ideal target deployment format for modern OCI-based immutable Linux images |
| **`systemd-repart`** | C | `LGPL-2.1-or-later` | Very active | **Reuse / Wrap** | Declarative GPT partition creation, encryption, and image copying |
| **`wimlib` (`wimlib-imagex`)** | C | `GPL-3.0-or-later` (lib: `LGPL-3.0-or-later` / GPL) | Active (`v1.14.5`, Jan 2026) | **Wrap** | Essential tool for applying Windows `install.wim`/`install.esd` directly from Linux |
| **`libosinfo/osinfo-db`** | XML / RelaxNG | `GPL-2.0-or-later` | Very active | **Study / Reference** | Canonical OS IDs, hardware requirements, and unattended install script templates |
| **Agama (`agama-project/agama`)** | Rust, TypeScript, Ruby | `GPL-2.0-only` | Very active (openSUSE) | **Study** | Modern Rust-backed headless+web/CLI Linux installer architecture |
| **`quickget` / `distrobuilder`** | Bash (`quickget`), Go (`distrobuilder`) | `MIT` / `Apache-2.0` | Active | **Study** | Upstream ISO URL discovery & checksum verification automation |

---

## Detailed Project Analyses

### 1. PINN & NOOBS
- **Primary URLs:** https://github.com/procount/pinn | https://github.com/raspberrypi/noobs (deprecated)
- **Architecture:** PINN boots a tiny Linux + Qt UI on Raspberry Pi, fetches JSON OS metadata (`os_list_v3.json`), partitions storage, downloads compressed tarballs/images, and writes a multi-boot menu.
- **Verdict — Study (PINN), Avoid (NOOBS):** PINN directly validates SBM's "install once, manage many OSes" UX. SBM improves on PINN by separating the fast native UEFI menu (`sbm-uefi`) from the Linux manager (`sbm-core`), enforcing cryptographic signature verification (TUF/Ed25519), and supporting x86-64/ARM64 PCs and Apple Silicon rather than only Raspberry Pi.

### 2. Ventoy & iVentoy
- **Primary URLs:** https://github.com/ventoy/Ventoy | https://www.iventoy.com
- **Architecture:** Ventoy patches GRUB2/EDK2 and injects an ACPI/device-mapper hook (`ventoy_hook`) into guest initramfs/WinPE environments to mount raw `.iso`/`.wim`/`.vhd` files directly from an exFAT partition. iVentoy is a closed-source network PXE equivalent.
- **Limitations & Supply-Chain Risks:** Ventoy ships numerous precompiled binary blobs in its source tree (a well-documented barrier to distro packaging and security auditing), has fragile per-distro hook breakage when OS initramfs formats change, and uses `GPL-3.0-or-later`. iVentoy is proprietary.
- **Verdict — Study / Avoid embedding:** Do NOT embed Ventoy blobs. Instead, `sbm-core` can cleanly implement verified OS installation from ISOs/WIMs/OCI images/tarballs using transparent open-source tools (`wimlib`, `kexec`, `unsquashfs`, `bootc`, loop-mounted EFI chainloading where appropriate).

### 3. netboot.xyz
- **Primary URLs:** https://netboot.xyz | https://github.com/netbootxyz/netboot.xyz
- **Architecture:** Declarative YAML catalog (`endpoints.yml`) rendered into iPXE boot menus (`Apache-2.0`), fetching kernels/initrds directly from upstream distro mirrors over HTTP/HTTPS.
- **Verdict — Study:** Excellent inspiration for `sbm-catalog`'s separation of upstream mirror definitions from runtime binaries. SBM improves on netboot.xyz by requiring strict SHA-256/SHA-512 verification and signed catalog manifests (`sbm-catalog` + TUF) rather than trusting unsigned HTTP endpoints.

### 4. Bare-Metal Provisioners: MAAS, Tinkerbell, FOG, OpenStack Ironic, Foreman
- **URLs:** https://github.com/canonical/maas | https://github.com/tinkerbell | https://github.com/FOGProject/fogproject | https://opendev.org/openstack/ironic | https://github.com/theforeman/foreman
- **Architectural Lessons:**
  - **Tinkerbell's HookOS** (an in-RAM minimal Linux worker executing declarative provisioning steps reported back to a control plane) is the closest datacenter analog to **`sbm-core` + `sbm-server`**.
  - **MAAS Curtin** proves fast image-based deployment (streaming a compressed rootfs image + installing bootloader) beats running interactive installers for unattended provisioning.
  - **FOG Project** proves `partclone` + `zstd` image capture/restore is fast and reliable across Windows and Linux partitions.
  - **OpenStack Ironic** provides the reference model for optional Redfish BMC integration in later SBM phases.
- **Verdict:** **Study** Tinkerbell, MAAS, FOG, and Ironic architectures; **Avoid** adopting their heavy enterprise stacks (Kubernetes/Rails/OpenStack) as SBM dependencies.

### 5. Linux Installers: Calamares, Agama, `bootc`, `systemd-repart`, `nixos-anywhere`
- **URLs:** https://codeberg.org/Calamares/calamares | https://github.com/agama-project/agama | https://github.com/bootc-dev/bootc | https://github.com/systemd/systemd | https://github.com/nix-community/nixos-anywhere
- **Key Findings (2026):**
  - **Agama** (openSUSE's modern installer) proves the power of separating a **Rust backend service** (storage/installation engine) from both web UI and local TUI/GUI frontends.
  - **`bootc`** (CNCF Sandbox, `MIT OR Apache-2.0`, written in Rust) allows installing immutable Linux OSes directly from signed OCI container images (`bootc install to-disk` / `to-filesystem`).
  - **`systemd-repart`** provides declarative, reproducible GPT partition provisioning with optional LUKS2/TPM2 enrollment.

---

## Cross-OS Installation Mechanics from `sbm-core` (Linux)

A core capability of `sbm-core` is installing Windows, Linux, and BSD without requiring the user to manually flash USB drives:

1. **Linux Installation (3 supported strategies in `sbm-catalog`):**
   - *Strategy A — Direct Installer Boot (`kexec` or EFI chainload):* Download and verify upstream kernel + initrd (or netboot ISO) and launch directly with optional preseed/Kickstart/AutoYaST/cloud-init/Autoinstall answers.
   - *Strategy B — Image / Rootfs Extraction:* Partition with `sfdisk`/`systemd-repart`, format (`mkfs.ext4`/`mkfs.btrfs`/`mkfs.fat`), extract verified rootfs image/squashfs/tarball, generate `/etc/fstab` + initramfs, install EFI stub or bootloader, and register boot target.
   - *Strategy C — Container/Immutable (`bootc` / OSTree):* Deploy signed OCI bootable container images directly to disk via `bootc`.
2. **Windows 10/11 Installation from Linux (Fully Open-Source Pipeline):**
   - Verify official Microsoft ISO / ESD / WIM checksum.
   - Create GPT layout: EFI System Partition (FAT32, 260–512 MiB), Microsoft Reserved Partition (MSR, 16 MiB, type `E3C9E316-0B5C-4DB8-817D-F92DF00215AE`), Windows NTFS partition (`mkntfs -f -L Windows`).
   - Apply the selected image index directly to NTFS using **`wimlib-imagex apply install.wim <index> /dev/nvme0n1pX`** (`wimlib 1.14.5`, supports WIM and encrypted-free ESD).
   - Inject optional `Autounattend.xml` (into `Windows/Panther/Unattend.xml`) and optional VirtIO/storage drivers.
   - Populate EFI boot files (`\EFI\Microsoft\Bootootmgfw.efi` extracted from `Windows/Boot/EFI/`) and generate a valid BCD hive (either via WinPE stage-1 boot — safest for official Microsoft BCD creation — or verified `hivex` BCD template patching; **WinPE first-stage handoff is recommended initially** to guarantee 100% Windows Update / BitLocker compatibility).
3. **BSD Installation (FreeBSD, OpenBSD, NetBSD):**
   - Download & verify official `.img` / `.iso` or base sets (`base.txz`, `kernel.txz`) + UEFI `loader.efi`, or chainload the verified BSD EFI installer image cleanly via a temporary staging partition.
