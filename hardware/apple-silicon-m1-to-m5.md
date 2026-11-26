# Apple Silicon Architecture, Boot Chain & M1–M5 Support Study

**Last verified:** 2026-11-26  
**Primary Sources:**
- Asahi Linux Feature Support Docs: `https://github.com/AsahiLinux/docs` (`m3.md`, `m4.md`, `m5.md`, `soc-codenames.md`)
- Asahi Linux Blog ("M2: Episode 1 — Asahi Linux on M3", Sep 2026): `https://asahilinux.org/2026/09/m2-episode-1/`
- `AsahiLinux/m1n1`: `https://github.com/AsahiLinux/m1n1`
- `AsahiLinux/u-boot` (`asahi` & `asahi-releng` branches, U-Boot `v2026.07` base): `https://github.com/AsahiLinux/u-boot`
- Apple Platform Security Guide: `https://support.apple.com/guide/security/` (Boot process for a Mac with Apple silicon, LocalPolicy, Startup Disk security policy, Paired recoveryOS)
- macOS 27.3 Golden Gate Beta Release Notes (`FB23916610`, Nov 2026)

---

## 1. Fundamental Reality: Apple Silicon Is NOT a Conventional UEFI Platform

Apple Silicon Macs do **not** boot UEFI firmware from SPI flash. Their hardware-enforced chain of trust is:

1. **Boot ROM** (immutable on-SoC silicon) -> verifies and loads **LLB** (Low-Level Bootloader) + **iBoot1** from internal NOR/NVMe (`iSCPreboot`).
2. **LLB** verifies system-paired firmware (storage, display, SMC, Thunderbolt) and validates a **Secure Enclave-signed `LocalPolicy`** per installed OS volume (protected with anti-replay counters in Secure Storage).
3. **iBoot2 (mBoot)** loads the OS boot object from the APFS Preboot container:
   - In **Full Security** (default for macOS): requires personalized Apple ECID-bound signatures.
   - In **Reduced Security**: allows global Apple signatures and AuxKC kexts.
   - In **Permissive Security** (only configurable in **1TR — One True Recovery**, entered by physically holding the power button, using `bputil`/`kmutil` with owner authentication): allows iBoot to boot a **Custom Boot Object** (an unsigned Mach-O payload blessed via `bputil -k`, specifically **`m1n1`**).
4. **Critical Safety Invariants for SBM:**
   - macOS **cannot** modify `LocalPolicy` or enroll a custom boot object from a normal running OS session; physical presence in 1TR + owner authentication is strictly required by Apple hardware design.
   - Apple Silicon internal NVMe contains platform-critical APFS containers (**`iSCPreboot`**, **macOS `System`/`Data`/`Preboot`/`Recovery`**, and **System Recovery `RecoveryOS`**). Corrupting or deleting `iSCPreboot` or System Recovery can require a **DFU restore from a second Mac**!
   - **SBM MUST NEVER** modify or auto-format Apple `iSCPreboot`, `RecoveryOS`, or macOS APFS containers without explicit, multi-step verified procedures (and must block them completely by default in `sbm-core`).

---

## 2. Core Upstream Projects Evaluated

| Project | Language(s) | License (SPDX) | Maintenance (Nov 2026) | Verdict | Role in SBM |
|---|---|---|---|---|---|
| **Asahi Linux (`linux-asahi` + `asahi-installer` + `asahi-scripts` + Mesa AGX/Honeykrisp)** | C, Rust (AGX DRM/kernel driver), Python, Shell | `GPL-2.0-only` (kernel) / `MIT` (installer/scripts/Mesa) | Very active | **Reuse / Build On** | Upstream kernel (`kernel-16k`), firmware extraction (`asahi-fwextract`), and GPU drivers for Apple Silicon `sbm-core` |
| **`m1n1`** (`AsahiLinux/m1n1`) | C, Python (proxy/tooling) | `MIT` | Very active | **Reuse** | Stage-1/Stage-2 Apple Silicon bootloader & hypervisor booted by iBoot; hands off to Asahi U-Boot |
| **Asahi U-Boot** (`AsahiLinux/u-boot`, `asahi-releng` branch) | C | `GPL-2.0-or-later` | Very active (`v2026.07` base; Sep 2026 commits for M3 `t8122` & NVMe) | **Reuse** | Provides `CONFIG_EFI_LOADER` UEFI Boot Services (GOP simple framebuffer, NVMe, USB xHCI, ESP detection) to launch the **standard `aarch64-unknown-uefi` `sbm-uefi` binary** |

### Why SBM Does NOT Duplicate `sbm-uefi` for Apple Silicon
Because `m1n1` chainloads **Asahi U-Boot**, and Asahi U-Boot implements UEFI Boot Services (`CONFIG_EFI_LOADER` with GOP framebuffer, keyboard/USB HID, NVMe/USB block I/O, and ESP scanning), **`sbm-as` can boot the exact same `aarch64-unknown-uefi` binary produced by `sbm-uefi`**! `sbm-as` only owns:
- Apple stub APFS container setup / integration with Asahi's `m1n1` + `u-boot` chain,
- Apple-specific platform capability quirks (e.g., no persistent UEFI NVRAM writeback — ESP config only; 16 KiB kernel page size requirement; no UEFI Secure Boot `db`/`KEK`),
- `asahi-fwextract` integration for Wi-Fi/BT/SoC firmware blobs lawfully extracted from the machine's own macOS volume,
- Apple partition protection guardrails.

---

## 3. Verified Apple Silicon Support Matrix (November 2026)

> **Strict Evidence Standard:** Every cell below reflects primary upstream `AsahiLinux/docs` and release notes verified on **2026-11-26**.

### SoC Generation Summary

| Generation | SoC IDs | Official Asahi Installer | Kernel / Boot Status (Nov 2026) | GPU / Display | USB / External Storage | Wi-Fi / BT |
|---|---|---|---|---|---|---|
| **M1 Series** | `T8103`, `T6000/1/2` | **Yes (Stable)** | Mature (`linux-asahi` + broad mainline) | **Verified:** AGX + Mesa OpenGL 4.6 / Vulkan 1.4 (Honeykrisp) + DCP | **Verified:** USB2/USB3 + NVMe in U-Boot & Linux | **Verified** |
| **M2 Series** | `T8112`, `T6020/1/2` | **Yes (Stable)** | Mature (`linux-asahi` + broad mainline) | **Verified:** AGX + Mesa OpenGL / Vulkan + DCP | **Verified:** USB2/USB3 + NVMe in U-Boot & Linux | **Verified** |
| **M3 Series** | `T8122`, `T6030/1/2/4` | **Yes (Merged Sep 2026)** | **Verified:** Almost full feature parity with M1/M2 (webcam, mics, USB3 10 Gbps, HW video decode incl. AV1, Wi-Fi, BT) | **Partial / WIP:** Boot/DCP works; performant 3D GPU acceleration still in progress | **Verified:** USB2/USB3 in `linux-asahi`; M3 U-Boot `t8122`/`t6032` merged May–Sep 2026 | **Verified** |
| **M4 Series** | `T8132`, `T6040/41` (+ A18 Pro `T8140`) | **No** | **Early bring-up:** `linux-asahi` (7.4/7.5 target) has AICv3, UART, Watchdog, I²C, GPIO, NVMe, PCIe, PMU WIP | **TBA:** Boot framebuffer only; DCP & GPU `TBA` | **Partial (M4 base only):** USB2/USB3 in `linux-asahi` for M4 `T8132`; M4 Pro/Max `TBA` | **TBA** |
| **M5 Series** | **`T8142` (M5)**, `T6050` (M5 Pro/Max/Ultra) | **No** | **Very early bring-up (M5 `T8142` only):** `linux-asahi` (7.5 target) has Devicetree, Boot framebuffer (`simpledrm`), internal **NVMe**, **Keyboard** & KB backlight, AICv3, UART, Watchdog, I²C, GPIO; PMU `WIP`. M5 Pro/Max/Ultra (`T6050`) mostly `TBA`. | **Boot framebuffer only** (`linux-asahi` 7.5); DCP & GPU **`TBA`** | **NOT YET SUPPORTED (`TBA`):** USB2, USB3, Thunderbolt, PCIe, and DART are all **`TBA`** on M5 (`T8142` & `T6050`) | **TBA** (Touchpad, Battery, Audio, Brightness, Webcam also `TBA`) |

### Crucial M5 Firmware Finding (Verified Nov 2026)
- **Apple `mBoot` Single-Core Bug (`FB23916610`):** Filed by Asahi Linux contributor Yureka. On **M5 Pro, M5 Max, and M5 Ultra**, booting Linux/custom raw boot objects originally exposed only **1 CPU core**.
- **Resolved in macOS 27.3 Golden Gate Beta** (Beta 1 `26C5035r`, Nov 3, 2026; Beta 2 `26C5047e`, Nov 17, 2026, `mBoot-20457.60.65.0.1`), allowing `m1n1` + Linux to bring up all cores. Crucially, existing Linux stubs must recreate their Preboot restore bundle (`/chosen/firmware-version`) under macOS 27.3+ to inherit the updated stage-2 `mBoot`.
- **Development Workstation Status (M5 MacBook Air, `T8142`, macOS 27.0 build `26A428`):**
  - M5 (`T8142`) has early `linux-asahi` bring-up (boot framebuffer, NVMe, keyboard), **NO official Asahi installer**, **NO USB/Thunderbolt/PCIe/DART/Wi-Fi/GPU/Touchpad drivers yet**.
  - **Therefore:** Physical M5 bare-metal SBM boot is **explicitly out of scope for SBM 0.1** and tracked as an active upstream dependency in `sbm-as`. The M5 MacBook Air serves as a high-speed ARM64 development & QEMU HVF virtualization host without modifying its macOS boot configuration or partitions.

---

## 4. Key Research Question: Internal Bootstrap -> External Storage OS Boot on Apple Silicon

> *"Can SBM provide an internally installed bootstrap that subsequently allows booting compatible operating systems from external storage?"*

### Verified Architectural Answer (Distinguishing What Works vs What Is Blocked)

1. **Why an Internal Stub Is Architecturally Mandatory on Apple Silicon:**
   - Apple Silicon's hardware Boot ROM / LLB / `LocalPolicy` architecture **does not allow booting arbitrary unsigned Linux/UEFI images directly from a generic external USB drive** the way an x86 PC does.
   - Apple's built-in external startup disk feature only works for **signed macOS installations** whose ownership and LocalPolicy are blessed through Apple's security model.
   - Therefore, installing a **tiny internal stub (~500 MB–1 GB APFS stub + small ESP)** containing `m1n1` + `Asahi U-Boot` + `sbm-uefi` (blessed once in 1TR) **IS the exact right architectural design** for booting third-party OSes from external storage!

2. **What Works TODAY (Verified on M1, M2, and M3):**
   - Internal `iBoot` -> internal `m1n1` -> internal `Asahi U-Boot` -> `sbm-uefi` (from internal ESP) -> scans **external USB-C (USB2/USB3 xHCI) mass storage**!
   - U-Boot (`apple_m1_defconfig`) explicitly enables `CONFIG_USB_XHCI_HCD`, `CONFIG_USB_XHCI_DWC3`, `CONFIG_USB_XHCI_PCI`, `CONFIG_USB_DWC3`, and `CONFIG_EFI_LOADER`.
   - Once `sbm-uefi` loads a `linux-asahi` kernel + initramfs (either from internal ESP or external USB ESP), the Linux kernel mounts the root filesystem from an **external USB-C SSD** reliably on M1, M2, and M3!
   - Even better: **Multiple Linux distributions** can live on external USB SSDs without consuming internal Mac NVMe space (only the tiny internal bootstrap stub + shared `sbm-core` recovery image needs internal space, or even `sbm-core` itself can live externally!).

3. **Current Technical Limitations & Blockers (Explicitly Documented):**
   - **M4 Pro/Max & M5 (`T8142`/`T6050`) Blocker (Nov 2026):** External USB/Thunderbolt boot is **currently impossible on M5** until upstream Asahi Linux lands **DART, PCIe, and USB2/USB3 (`apple-atc` / `dwc3`)** drivers for M5 in `m1n1`, `u-boot`, and `linux-asahi`.
   - **Thunderbolt PCIe Tunneling (External NVMe over pure TB3/TB4):** Still `WIP` on M3 and `TBA` on M4/M5. Standard **USB 3.2 Gen 2 (10 Gbps) enclosures** (using UASP over xHCI, NOT pure Thunderbolt PCIe tunneling) are vastly more compatible in U-Boot and `linux-asahi` than Thunderbolt-only enclosures!
   - **macOS Booting from `sbm-uefi` on Apple Silicon:** `sbm-uefi` (running inside `m1n1` -> U-Boot) **cannot directly chainload macOS** because macOS requires Apple's iBoot + signed `LocalPolicy` + paired firmware handoff. Switching back to macOS from SBM on Apple Silicon works by **rebooting with the Apple startup target (or NVRAM `boot-volume` / Apple Startup Picker)** — never by attempting to EFI-chainload `boot.efi`.
   - **External OS Compatibility:** External OSes booted on Apple Silicon **must** be ARM64 OSes with Apple Silicon 16 KiB page-size kernels + Apple devicetree/drivers (e.g., `linux-asahi` based distros, OpenBSD/arm64 on supported M1/M2 models). Generic ARM64 ACPI-only ISOs (like standard Windows 11 ARM64 or stock generic Linux kernels without Apple DTB/drivers) will NOT boot bare-metal on Apple Silicon without platform driver support.

---

## 5. Minimal Asahi Core OS Sizing (< 4 GB Goal -> Target < 1 GB)

The user rightly observed that `sbm-core` on Apple Silicon must use Asahi's Linux stack and stay compact (under 4 GB). Primary-source analysis shows we can do **even better**:
- **Asahi is a kernel + bootloader + firmware + Mesa layer, NOT a mandatory 8 GB desktop install.**
- Using **`mkosi` (v27.1)** with **Fedora Asahi Remix's minimal package repositories** (`kernel-16k` `linux-asahi`, `uboot-asahi`, `m1n1`, `mesa-asahi` when supported on the SoC, `systemd`, `labwc` + XWayland, `gparted`, `partclone`, `cryptsetup`, `sbm-core` Rust daemon/GUI, and selective firmware):
  - Compressed immutable **EROFS (`zstd`)** rootfs size: **~450–850 MB** (well under **1 GB**, less than one-quarter of the 4 GB budget!).
  - Combined with the APFS stub container (~500 MB required by Apple macOS Preboot/Recovery stub) + 512 MB ESP, the **entire internal Apple Silicon footprint** stays around **~1.5–2.0 GB total**, leaving internal NVMe free and allowing full RAM-resident recovery execution on 8–64 GB Macs.
