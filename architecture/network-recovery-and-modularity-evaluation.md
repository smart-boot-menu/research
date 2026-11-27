# Network Recovery Bootstrap & Maximum Modularity Evaluation

**Date:** 2026-11-26  
**Scope:** Primary-source build-vs-reuse investigation of (1) standard network boot and minimal Linux network-bootstrap mechanisms for remote OS repair, and (2) `0SFS`-style modular OS extensions using `mkosi`, `systemd-sysext`, `systemd-confext`, signed `dm-verity` DDIs, and `systemd-sysupdate`, per [`ADR-010`](https://github.com/smart-boot-menu/docs/blob/main/architecture-decisions/ADR-010-open-source-reuse-and-engineering-principles.md) and [`ADR-012`](https://github.com/smart-boot-menu/docs/blob/main/architecture-decisions/ADR-012-profiles-network-recovery-modularity-and-ux.md).

---

## Part 1: Network Recovery Bootstrap (Existing Standards Over Custom Protocols)

### 1.1 Primary Product Scenario

A user's target computer has a completely broken installed operating system. On a working machine, the user runs **SBM Manager** to prepare a recovery USB — optionally preconfiguring network credentials (Ethernet/Wi-Fi), required modules, cache policy, and an explicitly authorized, revocable remote-recovery session grant. The broken computer boots SBM from USB (or an existing `< 1 MiB` minimal EFI-only install), downloads and authenticates `sbm-core`, executes it from **RAM** or explicitly authorized storage, establishes an authenticated encrypted remote session, and lets the owner repair the broken OS.

### 1.2 Primary-Source Comparison of Network Boot & Bootstrap Layers

| Approach | Primary Source / Standard | License | Wired Ethernet | Consumer Wi-Fi | TLS / CA Trust | Secure Boot & Integrity Chain | Assessment for SBM |
|---|---|---|---|---|---|---|---|
| **Native UEFI HTTP(S) Boot** | UEFI 2.5+ (`HttpBootDxe` in EDK II, U-Boot HTTP Boot) | Firmware built-in (`BSD-2-Clause-Patent` / `GPL-2.0`) | **Yes** (most modern servers & PCs) | **Rare** (proprietary OEM business laptops only; commodity PCs/boards lack UEFI WLAN drivers & firmware blobs) | Supported, but firmware ships **no public CA store** (requires manual NVRAM CA enrollment); HTTP `301`/`302` redirects frequently unsupported; pre-Secure-Boot stack CVEs exist (e.g., EDK II `CVE-2026-97631`) | Firmware verifies downloaded PE/COFF `.efi` or signed UKI | **Reuse directly for wired Ethernet zero-Linux bootstrap** (fetching a signed SBM EFI or minimal Linux Bootstrap UKI over standard HTTPS/HTTP Boot without inventing a custom protocol). Cannot serve consumer Wi-Fi recovery alone. |
| **iPXE** (`ipxe.efi` / `snponly.efi`) | [`ipxe.org`](https://ipxe.org/) | `GPL-2.0-or-later` | **Yes** | **No** (native iPXE Wi-Fi drivers are obsolete; only works if firmware already exposes SNP Wi-Fi) | Supports HTTPS & inherits UEFI HTTPS certs | Pre-signed Microsoft-shim binaries forbid custom embedded scripts; Secure Boot does not verify DHCP/`autoexec.ipxe`/separate initrds unless locked with `imgtrust`/`imgverify` | **Optional third-party / enterprise interop only.** Useful if an operator's existing PXE/iPXE infrastructure chainloads a signed SBM UKI, but do **not** embed or fork iPXE as SBM's universal consumer recovery bootstrap because it lacks modern Wi-Fi drivers and complicates Secure Boot signing. |
| **Signed UKI (`systemd-stub`)** | [`systemd-stub(7)`](https://www.freedesktop.org/software/systemd/man/latest/systemd-stub.html) | `LGPL-2.1-or-later` | N/A (boot payload) | N/A (boot payload) | N/A | Single PE signature covers kernel (`.linux`), initrd (`.initrd`), and `.cmdline`; measures into **TPM2 PCR 11**; ignores external cmdline overrides under Secure Boot | **Core building block.** Both the minimal Linux Network-Bootstrap UKI and full `sbm-core` UKI use standard signed UKIs. |
| **Signed Minimal Linux Network-Bootstrap UKI + `dm-verity` `switch-root`** | Linux kernel + `systemd` initrd + `iwd`/`wpa_supplicant` + `linux-firmware` + HTTPS client | `GPL-2.0` / `LGPL-2.1` / `MIT OR Apache-2.0` | **Yes** (full Linux NIC drivers) | **Yes** (full Linux `iwlwifi`/`rtw89`/`ath11k`/`brcmfmac` + WPA2/WPA3/EAP via `iwd`) | Full Linux TLS stack + standard CA bundle + pinned SBM TUF/Ed25519 keys | Firmware verifies Bootstrap UKI -> initrd verifies downloaded `sbm-core` EROFS + `dm-verity` roothash/PKCS#7 signature -> `switch-root` directly into verified `dm-verity` root | **Recommended universal consumer network-recovery engine.** Solves consumer Wi-Fi, standard TLS/HTTP redirects, HTTP `Range` download resume, and strict RAM-only execution without `kexec_file_load(2)` Secure Boot / UKI limitations! |
| **Minimal Linux Bootstrap + `kexec_file_load(2)`** | Linux `kexec_file_load(2)` (`kexec -s`) | `GPL-2.0` | Yes | Yes | Full Linux TLS | Under Secure Boot lockdown, legacy `kexec_load(2)` is blocked; `kexec_file_load(2)` verifies only the kernel FD (not separate initrds) and cannot directly invoke a UKI's `systemd-stub` EFI entry point after `ExitBootServices()` | **Avoid for Stage-2 UKI handoff when `switch-root` suffices.** Keeping the same verified kernel from the Bootstrap UKI and `switch-root`-ing into the downloaded, `dm-verity`-verified EROFS root + signed `systemd-sysext` modules preserves the entire Secure Boot chain cleanly. |

### 1.3 Handling Real-World Network Recovery Failure Modes

| Condition | Required SBM Architecture Behavior |
|---|---|
| **Ethernet vs. Wi-Fi** | Wired Ethernet works via standard UEFI HTTP(S) Boot (for minimal `< 1 MiB` EFI-only profile) **and** inside the signed Linux Network-Bootstrap UKI. Consumer Wi-Fi (WPA2/WPA3, USB pre-configured by **SBM Manager** or interactive picker) is handled exclusively inside the Linux Network-Bootstrap UKI using standard Linux drivers (`iwd`/`wpa_supplicant` + curated firmware) — never by writing custom Wi-Fi drivers in `sbm-uefi`. |
| **Insufficient RAM (`≤ 2 GiB`)** | Detect physical RAM before downloading. If RAM cannot safely hold the full graphical EROFS in `tmpfs`/`zram` (`< 2.5–4 GiB`), refuse RAM-only graphical download before wasting bandwidth and offer: (a) **basic low-RAM CLI/minimal recovery mode** (direct EROFS mount from USB/authorized cache without `copytoram`, or headless minimal diagnostic + repair shell fitting in `≤ 2 GiB`), or (b) explicitly authorized USB cache (never silently falling back to internal SSD!). |
| **Unavailable Networking** | Degrade cleanly with actionable diagnostics (NIC link state, driver/firmware status, Wi-Fi scan/auth failure, DHCP/DNS/TLS error) and automatically offer any locally cached or offline-bundled `sbm-core` / local UEFI OS chainloading without hanging. |
| **Interrupted Downloads** | Fetch EROFS base and `.sysext.raw` DDIs over standard HTTPS with **HTTP `Range` resume** + chunked SHA-256 / `dm-verity` Merkle verification. Partial downloads stay in RAM (`tmpfs`) or explicitly authorized USB staging — never on unauthorized internal disks. |
| **Zero Unauthorized Disk Writes (`ram-only`)** | In RAM-only mode, `sbm-core` MUST NOT mount target internal SSD partitions read-write for staging, MUST NOT write cache files to internal disks, and MUST NOT activate swap on any internal disk partition (`zram` in RAM only, or zero swap). |
| **Remote Recovery Session Security** | Recovery USBs prepared by **SBM Manager** MUST NEVER store permanent shared admin credentials or enable unattended arbitrary command execution. Remote access requires an **ephemeral, time-bounded, single-session grant** (short-lived token + pinned server/relay public key + explicit least-privilege scope), end-to-end encryption, visible local on-screen active-session indicator with instant local disconnect/kill-switch, and automatic revocation on completion, timeout, or reboot. |

---

## Part 2: Maximum Modularity (`0SFS`-Style — Zero Custom Package Manager)

### 2.1 Why SBM Must NOT Invent a General-Purpose Package Manager

Per `ADR-010`, writing a custom package format, dependency SAT solver, or mutable file-level installer inside an immutable recovery OS creates high security risk, breaks `dm-verity` root immutability, and duplicates mature upstream Linux mechanisms. Instead, SBM composes **`mkosi` + `systemd-sysext` + `systemd-confext` + signed `dm-verity` DDIs + `systemd-sysupdate`**.

### 2.2 Primary-Source Architecture of the `systemd` Extension Stack

1. **Build Time (`mkosi` `Format=sysext` / `Format=confext`):**
   - Using `mkosi.images/` with `Dependencies=base`, `Overlay=yes`, and `BaseTrees=%O/base`, `mkosi` packages **only the delta files** for each optional capability into a Discoverable Disk Image (DDI) backed by compressed read-only EROFS, generates `/usr/lib/extension-release.d/extension-release.<NAME>` (or `/etc/extension-release.d/` for `confext`), builds the `dm-verity` hash partition, and embeds a **PKCS#7 signature** of the verity root hash in the `verity-sig` partition (`VerityKey=` / `VerityCertificate=`).
2. **Compatibility & Versioning (`SYSEXT_LEVEL=`):**
   - `systemd-sysext` natively validates `ID=sbm-core`, `SYSEXT_LEVEL=` (decoupling extension ABI compatibility from minor base patches or locking strictly per base build), `ARCHITECTURE=` (`x86-64` / `arm64`), and `SYSEXT_SCOPE=` (`initrd`, `system`, `portable`).
3. **Production Verification (`--image-policy=` & TPM2 PCR 13):**
   - Host `systemd-sysext` accepts unsigned images by default — **SBM production MUST override this** with `--image-policy='root=signed+absent:usr=signed+absent:=unused+absent'` (configured in `sysext.conf`) so unsigned or tampered extensions are rejected fail-closed against pinned `/usr/lib/verity.d/*.crt` or kernel keyrings (`CONFIG_DM_VERITY_VERIFY_ROOTHASH_SIG`).
   - Early-boot initrd extensions in ESP `/loader/extensions/*.sysext.raw` are automatically verified as signed DDIs (`root=signed+absent:usr=signed+absent`) and measured by `systemd-stub` into **TPM2 PCR 13**, keeping **PCR 11** clean for base UKI policies.
4. **Atomic Delivery & Lifecycle (`systemd-sysupdate` + `systemd-sysext`):**
   - **Download & cache:** `systemd-sysupdate` (with `Features=` for version-locked modules or `--component=` for independent modules) plus TUF-signed `ModuleDescriptor` metadata downloads verified `.sysext.raw` DDIs into `/run/extensions/` (RAM-only) or explicitly authorized `/var/lib/extensions/` (persistent USB/SSD cache).
   - **Activate / Deactivate / Remove:** `systemd-sysext merge` / `refresh` / `unmerge` (with `EXTENSION_RELOAD_MANAGER=1` / `EXTENSION_RESTART_UNITS=` in `extension-release`) activates or deactivates immutable modules cleanly on top of read-only `/usr` via `overlayfs`; deleting the `.sysext.raw` file removes the cached module with zero leftover files on the immutable root.

### 2.3 Comparison With Alternative Packaging Mechanisms

| Mechanism | Strengths | Why Selected / Rejected for SBM Core Modules |
|---|---|---|
| **`systemd-sysext` / `systemd-confext` (Signed `dm-verity` DDIs) + `mkosi`** | Built into `systemd` & `mkosi v27.1` already used by `sbm-core`; zero new runtime daemon; preserves read-only `dm-verity` immutability; native `SYSEXT_LEVEL` + arch + PKCS#7 signature enforcement + TPM2 PCR 13; works identically in RAM (`/run/extensions`) and authorized cache (`/var/lib/extensions`). | **Selected as canonical SBM Core module format.** |
| **`rpm` / `dnf` / `apt` mutable packages** | Huge upstream repos. | Breaks immutable `dm-verity` root, requires mutable overlay/package DB, slow/fragile offline and RAM-only lifecycle. Keep RPMs strictly as **build-time inputs to `mkosi`**, never as a runtime package manager inside `sbm-core`. |
| **OCI / `bootc`** | Excellent for installing full immutable Linux *target OSes* onto user disks (`sbm-catalog` / SBM 0.1+). | Heavyweight for toggling individual recovery tools inside RAM-only `sbm-core` compared to single-file `.sysext.raw` DDIs. Keep `bootc` for target OS installation (Spec 04 §12). |
| **Flatpak / AppImage** | Good for desktop apps on mutable systems. | Requires sandbox portal/FUSE overhead; poorly suited for privileged system/storage tools (`gparted`, `ntfsresize`, `partclone`, `wayvnc`, `linux-firmware`). |

### 2.4 Proposed Modular Boundary (`sbm-core` Base Target `< 500 MiB` Compressed)

- **Mandatory Base (`sbm-core-base`, target `< 500 MiB` compressed):** Linux kernel + `systemd` + `dm-verity`/EROFS + networking (`iwd`/DHCP/Ethernet) + minimal `labwc` + Iced `sbm-ui` + `sbm-cored` + read-only disk discovery/diagnostics + `systemd-sysext`/`confext` + `systemd-sysupdate`.
- **Optional Signed Modules (`.sysext.raw` DDIs, independently selectable/cacheable/removable):**
  - `gparted` (XWayland + GTK3 + GParted `1.8.1`) — pre-bundled in the `0.0.1` self-contained offline profile, decoupled into an optional signed `sysext` in the modular profile (0.0.3+).
  - `storage-pro` (`ntfs-3g`, `btrfs-progs`, `partclone`, `ddrescue`, `testdisk`, `cryptsetup`, `lvm2`).
  - `os-install` (`wimlib-imagex`, `bootc`, `squashfs-tools`).
  - `remote` (`wayvnc`, authenticated remote relay client, WireGuard/Iroh).
  - `diagnostics` (`smartmontools`, `nvme-cli`, `memtester`, `ethtool`, `pciutils`, `usbutils`).
  - `firmware-wifi-extra` (extended wireless firmware pack beyond core Intel/Realtek/MediaTek/Qualcomm baseline).

---

## Primary Sources

- UEFI Specification (HTTP Boot, Wireless MAC Connection): https://uefi.org/specifications
- EDK II `HttpBootDxe` & `WifiConnectionManagerDxe`: https://github.com/tianocore/edk2
- iPXE HTTPS, Secure Boot & `imgtrust`: https://ipxe.org/crypto and https://ipxe.org/cmd/imgverify
- `systemd-stub(7)` (UKI, PCR 11/12/13, `/loader/extensions/`): https://www.freedesktop.org/software/systemd/man/latest/systemd-stub.html
- `systemd-sysext(8)` & `systemd-confext(8)` (`--image-policy=`, `SYSEXT_LEVEL=`): https://www.freedesktop.org/software/systemd/man/latest/systemd-sysext.html
- `systemd-sysupdate(8)` & `sysupdate.d(5)` (`Features=`, `--component=`): https://www.freedesktop.org/software/systemd/man/latest/systemd-sysupdate.html
- `mkosi` `Format=sysext` & `Overlay=yes`: https://github.com/systemd/mkosi (`docs/sysext.md`)
- Linux `kexec_file_load(2)` & Secure Boot lockdown constraints: https://docs.kernel.org/
