# Boot Management Ecosystem Research

**Last verified:** 2026-11-26  
**Scope:** Existing UEFI boot managers, network bootloaders, firmware stacks, LinuxBoot environments, and modern Rust UEFI projects evaluated for Smart Boot Menu (SBM).

---

## Summary Comparison Matrix

| Project | Primary Language(s) | License (SPDX) | Maintenance (Nov 2026) | Latest Verified Release / State | Verdict |
|---|---|---|---|---|---|
| **rEFInd** | C (GNU-EFI / TianoCore) | `GPL-3.0-only` (drivers vary) | Active (low-cadence) | `0.14.2` (2024; SourceForge active) | **Study** |
| **RefindPlus** | C (EDK2) | `GPL-3.0-only` / `BSD-3-Clause` | Active | Active GitHub releases (Mac/UEFI fork) | **Study** |
| **Meridian** | C (EDK2) | `GPL-3.0-only` | Active (2026) | `seuros/meridian` active (x86_64/AArch64) | **Study** |
| **Visor** | C (EDK2) | Open source (verify repo LICENSE) | Active (2026) | `v1.6.0` observed in 2026 | **Study** |
| **GRUB 2** | C, Assembly | `GPL-3.0-or-later` | Active | `2.12` / `2.14` dev cycle | **Study / Avoid embedding** |
| **systemd-boot** | C (meson) | `LGPL-2.1-or-later` | Very active | Ships with `systemd` (v260–v262 era) | **Study** (adopt Boot Loader Spec) |
| **Limine** | C | `BSD-2-Clause` | Very active | `v12.11.1` (2026) | **Study** |
| **Clover** | C / C++ (EDK2) | `BSD-2-Clause` | Active (maintenance) | `r5176` (2026-10-26) | **Avoid** |
| **OpenCore** | C (EDK2) | `BSD-3-Clause` | Very active | `v1.0.8` (2026-09-27) | **Study** (macOS/EFI rigor) |
| **iPXE** | C, Assembly | `GPL-2.0-only` (UBDL) | Active (rolling) | Rolling git `master` | **Study / Optional Wrap** |
| **Das U-Boot** | C | `GPL-2.0-or-later` | Very active | `v2026.10` (2026-10-05) | **Reuse** (via Asahi U-Boot on Apple Silicon & ARM boards) |
| **TianoCore EDK II** | C, Python | `BSD-2-Clause-Patent` | Very active | `edk2-stable202611` (2026-11-23) | **Reuse** (OVMF/AAVMF test firmware; study protocols) |
| **Petitboot** | C | `GPL-2.0-only` | Low cadence | `v1.15` (2024-02-13) | **Study** (kexec boot-UI model) |
| **LinuxBoot / Heads** | C, Go, Shell | `GPL-2.0-only` / `BSD-3-Clause` | Active | Rolling / board releases | **Study** (long-term ROM concept) |
| **u-root** | Go | `BSD-3-Clause` | Active | `v0.16.0` (2026) | **Study / Avoid** (Go stack vs Rust) |
| **`uefi-rs` (`uefi`)** | Rust | `MPL-2.0` | Very active | `uefi 0.41.0` / `uefi-raw 0.17.0` (Sep 2026) | **Reuse** (core foundation for `sbm-uefi`) |
| **`r-efi`** | Rust | `MIT OR Apache-2.0 OR LGPL-2.1-or-later` | Active | `v5.x` | **Study** (minimal FFI alternative) |
| **`crdyboot`** | Rust | `BSD-3-Clause` | Very active (Google) | Active ChromeOS Flex loader | **Study** (best-in-class Rust UEFI architecture reference) |
| **`oreboot`** | Rust | `GPL-2.0-only` | Active | Rolling | **Study** |
| **Patina** | Rust | `Apache-2.0` | Active (ODP/Microsoft, 2025–2026) | Active `OpenDevicePartnership/patina` | **Study** (next-gen Rust UEFI DXE reference) |
| **Redox Bootloader** | Rust, Assembly | `MIT` | Active | Rolling (`redox-os/bootloader`) | **Study** |

---

## Detailed Project Analyses

### 1. rEFInd
- **Primary URLs:** https://www.rodsbooks.com/refind/ | https://sourceforge.net/projects/refind/
- **Architecture:** Native UEFI C boot manager (not a kernel loader itself except via Linux EFISTUB). Scans all FAT/ESP partitions and additional partitions exposed by loaded UEFI filesystem drivers (`ext4`, `btrfs`, `hfs`, `iso9660`, etc.), renders a graphical icon row via GOP, and chainloads `.efi` binaries via `LoadImage`/`StartImage`.
- **Languages & License:** C; `GPL-3.0-only` (separate drivers carry mixed GPL/BSD licenses).
- **Maintenance:** Maintained by Roderick W. Smith; low-cadence stability releases (`0.14.2`).
- **Capabilities Relevant to SBM:** Automatic OS discovery heuristics across ESPs, clean visual boot menu, per-entry boot options, shim/MOK awareness, icon theming, EFI driver loading.
- **Limitations:** C codebase built on aging GNU-EFI/TianoCore patterns; no remote management, no built-in HTTPS networking, no recovery OS lifecycle management, `GPL-3.0-only` license conflicts with SBM's `MIT OR Apache-2.0` original code baseline.
- **Verdict — Study:** Study rEFInd's OS discovery heuristics (Windows Boot Manager, macOS, Linux distributions, EFISTUB auto-detection, fallback filenames) and UX simplicity; reimplement cleanly in Rust in `sbm-uefi`.

### 2. RefindPlus
- **Primary URLs:** https://github.com/RefindPlusRepo/RefindPlus
- **Architecture:** EDK2-based fork of rEFInd adding extensive workarounds for legacy Intel Mac firmware, GOP quirks, APFS/HFS+ loading, and pre-boot console/graphics initialization.
- **Languages & License:** C; `GPL-3.0-only` (with BSD-3-Clause portions).
- **Capabilities & Limitations:** Valuable catalog of real-world Apple/Intel UEFI GOP and NVRAM bugs; heavily tailored to older Intel Macs and EDK2 C build system.
- **Verdict — Study:** Reference its firmware quirk workarounds (GOP mode selection, screen clearing, pointer reset) when hardening `sbm-uefi`.

### 3. Meridian (`seuros/meridian`)
- **Primary URLs:** https://github.com/seuros/meridian
- **Architecture:** Modern EDK2-based UEFI boot manager targeting both `x86_64` and `AArch64`, descended from RefindPlus/rEFInd. Bundles built-in filesystem drivers (`ext4`, `btrfs`, `bcachefs`, `HFS+`, `UFS`, `NTFS`), supports EFISTUB and Limine kernel loading, and targets multi-OS discovery (Linux, BSDs, macOS, Windows, Haiku, 9front).
- **Languages & License:** C (EDK2); `GPL-3.0-only`.
- **Capabilities & Limitations:** Strong proof of modern dual-architecture (`x86_64` + `AArch64`) graphical boot management and multi-filesystem scanning; still bound to EDK2 C toolchain and GPL-3.0.
- **Verdict — Study:** Study its multi-OS discovery rules (including BSD/Haiku/modern Linux) and AArch64 UEFI compatibility practices.

### 4. Visor (`IO-ZetZor/Visor-BootManager`)
- **Primary URLs:** https://github.com/IO-ZetZor/Visor-BootManager
- **Architecture:** Modern graphical UEFI boot manager built on EDK2 focusing on polished visual UX and contemporary themes (`v1.6.0` observed in 2026 documentation). Exact latest release tag should be re-verified directly against GitHub API if deeper study is needed.
- **Languages & License:** C (EDK2); open source (exact SPDX to confirm in repo before any code reference).
- **Capabilities & Limitations:** Useful UX/rendering reference for modern high-resolution GOP menus; lacks SBM's remote-management, recovery-OS, and Rust safety architecture.
- **Verdict — Study:** Study UI layout and theme ergonomics; build SBM's UI natively in Rust.

### 5. GRUB 2
- **Primary URLs:** https://www.gnu.org/software/grub/ | https://git.savannah.gnu.org/cgit/grub.git
- **Architecture:** Monolithic multi-platform bootloader supporting BIOS, UEFI, OpenFirmware, coreboot, and IEEE1275 with an embedded scripting language, filesystem drivers, crypto/LUKS unlocking, and custom graphics engine.
- **Languages & License:** C, Assembly; `GPL-3.0-or-later`.
- **Capabilities & Limitations:** Broad legacy and filesystem support, but notoriously complex C codebase, large CVE attack surface (Boothole and subsequent Secure Boot SBAT revocations), slow graphical rendering, fragile config generation (`grub-mkconfig`).
- **Verdict — Study / Avoid embedding:** Chainload existing GRUB `.efi` installations transparently when discovered, but do NOT base `sbm-uefi` on GRUB.

### 6. systemd-boot (`sd-boot`)
- **Primary URLs:** https://github.com/systemd/systemd (under `src/boot/`) | https://uapi-group.org/specifications/specs/boot_loader_specification/
- **Architecture:** Minimalist, fast, UEFI-only boot manager implementing the UAPI **Boot Loader Specification (BLS)** (Type #1 drop-in `.conf` entries and Type #2 Unified Kernel Images in `/EFI/Linux/*.efi`) and the **Boot Loader Interface** (EFI variables under vendor GUID `4a67b082-0a4c-41cf-b6c7-440b29bb8c4f` for `LoaderEntrySelected`, `LoaderTimeInitUSec`, `LoaderEntryOneShot`, etc.).
- **Languages & License:** C; `LGPL-2.1-or-later`.
- **Capabilities & Limitations:** Text-only menu (no graphical UI, no network boot, no recovery management), but sets the modern Linux industry standard for BLS, UKIs, `StubFeatures`, and EFI-to-Linux handoff variables.
- **Verdict — Study (Adopt Standards):** `sbm-uefi` MUST parse Boot Loader Specification Type #1 and Type #2 (UKI) entries and implement compatible EFI handoff semantics so standard Linux distros work automatically.

### 7. Limine
- **Primary URLs:** https://github.com/Limine-Bootloader/Limine | https://limine-bootloader.org/
- **Architecture:** Clean, portable C multiprotocol bootloader (`v12.11.1` in 2026) supporting UEFI (`x86_64`, `aarch64`, `riscv64`, `loongarch64`) and legacy x86 BIOS, Linux/Multiboot/Limine protocols, and clean framebuffer UI.
- **Languages & License:** C; `BSD-2-Clause`.
- **Capabilities & Limitations:** Permissively licensed, very clean architecture and BIOS fallback experience; lacks SBM's Rust ecosystem, remote policy engine, and recovery integration.
- **Verdict — Study:** Excellent reference if SBM ever adds legacy x86 BIOS support in a later phase, and a clean reference for framebuffer rendering discipline.

### 8. Clover Bootloader
- **Primary URLs:** https://github.com/CloverHackyColor/CloverBootloader
- **Architecture:** Legacy EDK2/C++ Hackintosh-focused UEFI bootloader (`r5176`, Oct 2026; `BSD-2-Clause`).
- **Limitations:** Heavy macOS-patching legacy baggage, complex C++ codebase largely superseded by OpenCore.
- **Verdict — Avoid.**

### 9. OpenCore (`acidanthera/OpenCorePkg`)
- **Primary URLs:** https://github.com/acidanthera/OpenCorePkg
- **Architecture:** Security-focused, strictly engineered EDK2 UEFI booter (`v1.0.8`, Sep 2026; `BSD-3-Clause`) designed for macOS and multiboot environments. Features `OpenCanopy` graphical picker, strict UEFI protocol conformance, vault/signing verification, and exhaustive documentation of real-world firmware bugs (`Configuration.pdf`).
- **Capabilities & Limitations:** Industry benchmark for defensive UEFI engineering (handling buggy AMI/Insyde/Apple firmwares, HiDPI GOP scaling, Apple BootPicker quirks). C/EDK2 only.
- **Verdict — Study:** Mine OpenCore's firmware quirk documentation and `OpenCanopy` HiDPI design principles when building `sbm-uefi`.

### 10. iPXE
- **Primary URLs:** https://ipxe.org/ | https://github.com/ipxe/ipxe
- **Architecture:** Mature open-source network boot firmware (`GPL-2.0-only` with UBDL) supporting HTTP/HTTPS, TFTP, iSCSI, DNS, DHCP, Wi-Fi (limited legacy cards), and scripting.
- **Capabilities & Limitations:** Battle-tested NIC drivers and HTTP boot, but GPL-2.0 C/assembly codebase, limited modern Wi-Fi support, no graphical OS management or local disk tooling.
- **Verdict — Study / Optional Wrap:** SBM can be chainloaded *from* iPXE in enterprise PXE labs, and `sbm-uefi` can study iPXE's retry/timeout resilience, while relying on `sbm-core` (Linux) for full Wi-Fi/NIC driver coverage.

### 11. Das U-Boot
- **Primary URLs:** https://source.denx.de/u-boot/u-boot | https://github.com/u-boot/u-boot
- **Architecture:** Universal embedded/ARM bootloader (`v2026.10`, Oct 2026; `GPL-2.0-or-later`) with a full **UEFI Boot Services subsystem (`CONFIG_EFI_LOADER`)** supporting GOP framebuffer, block devices, USB mass storage, network, and executing standard `aarch64` `.efi` applications.
- **Capabilities & Limitations:** Critical bridge on Apple Silicon (via `AsahiLinux/u-boot`, based on `v2026.07`) and ARM SBCs that lack native UEFI firmware. Some UEFI protocols (e.g., advanced pointer/HTTP/NVRAM persistence) are more limited than EDK2 OVMF/AAVMF.
- **Verdict — Reuse (Upstream Platform Layer):** `sbm-as` reuses `AsahiLinux/u-boot` (downstream of `m1n1`) as the UEFI provider that launches the standard `aarch64-unknown-uefi` `sbm-uefi` binary — avoiding a separate Apple-only boot menu!

### 12. TianoCore EDK II
- **Primary URLs:** https://github.com/tianocore/edk2
- **Architecture:** Reference UEFI firmware implementation (`edk2-stable202611`, Nov 2026; `BSD-2-Clause-Patent`) producing **OVMF** (x86-64/IA32) and **AAVMF** (ARM64) virtual machine firmwares.
- **Verdict — Reuse (Testing & Reference):** Reuse OVMF/AAVMF binaries in `sbm-lab` for automated QEMU testing; do NOT build `sbm-uefi` using the EDK2 C build system.

### 13. Petitboot
- **Primary URLs:** https://github.com/open-power/petitboot
- **Architecture:** Linux `kexec`-based boot manager (`v1.15`, Feb 2024; `GPL-2.0-only`) originally for PS3 and OpenPOWER servers. Runs a tiny Linux kernel + `pb-discover` daemon + ncurses/framebuffer UI, then `kexec`s into the target Linux kernel.
- **Capabilities & Limitations:** Proves Linux-as-bootloader works well for Linux-to-Linux, but **cannot chainload Windows or macOS UEFI bootloaders cleanly** (no UEFI Boot Services after `ExitBootServices()`).
- **Verdict — Study:** Validates SBM's **two-tier architecture** — `sbm-uefi` MUST remain a native UEFI fast path to chainload Windows/Linux/EFI targets with sub-second latency, while `sbm-core` handles rich Linux-powered recovery/installation.

### 14. LinuxBoot & Heads
- **Primary URLs:** https://www.linuxboot.org/ | https://github.com/linuxboot/heads
- **Architecture:** Replaces UEFI DXE/BDS firmware stages in SPI flash with a Linux kernel + minimal initramfs (`u-root` or BusyBox/Heads) for tamper-evident measured boot.
- **Capabilities & Limitations:** Requires flashing motherboard SPI ROM (out of scope for user-installable SBM on consumer laptops/Macs).
- **Verdict — Study:** Useful security reference for TPM measured boot and immutable recovery designs.

### 15. u-root
- **Primary URLs:** https://github.com/u-root/u-root
- **Architecture:** Go-based userland and initramfs builder (`v0.16.0`, 2026; `BSD-3-Clause`) compiling Unix utilities into a single `gobusybox` binary for LinuxBoot.
- **Limitations:** Go runtime ecosystem diverges from SBM's Rust stack and does not run desktop GUI/GParted/Wayland workloads needed by `sbm-core`.
- **Verdict — Study / Avoid.**

### 16. Modern Rust UEFI Ecosystem (2025–2026)
- **`uefi-rs` (`uefi` v0.41.0, `uefi-raw` v0.17.0, Sep 2026, `MPL-2.0`):** https://github.com/rust-osdev/uefi-rs — De facto standard safe Rust UEFI library. Uses modern global module functions (`uefi::boot`, `uefi::system`, `uefi::runtime`) rather than legacy `SystemTable` parameter passing. MSRV Rust 1.91.0. **Verdict: Reuse** as the foundation of `sbm-uefi`.
- **`r-efi` (`MIT OR Apache-2.0 OR LGPL-2.1-or-later`):** https://github.com/r-efi/r-efi — Zero-abstraction raw UEFI FFI bindings. Useful reference, but `uefi-rs` provides safer ergonomics, `alloc` integration, and active protocol wrappers. **Verdict: Study.**
- **`crdyboot` (`BSD-3-Clause`):** https://github.com/google/crdyboot — Google's production Rust UEFI bootloader for ChromeOS Flex (`x86_64` and `ia32` / `aarch64`), built on `uefi-rs`. Demonstrates production patterns for Secure Boot, SBAT, TPM measurement, `no_std` unit testing via mocked UEFI traits, and clean PE loading. **Verdict: Study closely** — best real-world Rust UEFI engineering blueprint available.
- **Patina (`Apache-2.0`):** https://github.com/OpenDevicePartnership/patina — Announced Oct 2025 by Microsoft / Open Device Partnership; pure-Rust UEFI DXE firmware components. Validates Rust as a tier-1 UEFI language going into 2026+. **Verdict: Study.**
- **`oreboot` (`GPL-2.0-only`):** https://github.com/oreboot/oreboot — Downstream Rust fork of coreboot for SPI flash firmware. **Verdict: Study.**
- **`redox-os/bootloader` (`MIT`):** https://github.com/redox-os/bootloader — Rust UEFI+BIOS loader for Redox OS. **Verdict: Study.**
