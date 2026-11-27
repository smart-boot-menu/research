# SBM Manager & USB Flashing Tools Evaluation (Build vs. Reuse)

**Date:** 2026-11-26  
**Scope:** Primary-source evaluation of existing open-source USB/SD image flashing tools against Smart Boot Menu's user-facing provisioning and configuration application — **`SBM Manager`** (formerly referred to as "Media Creator") — per [`ADR-010`](https://github.com/smart-boot-menu/docs/blob/main/architecture-decisions/ADR-010-open-source-reuse-and-engineering-principles.md) and [`ADR-012`](https://github.com/smart-boot-menu/docs/blob/main/architecture-decisions/ADR-012-profiles-network-recovery-modularity-and-ux.md).  
**Target Milestones:** Immediate zero-code flashing reuse in **SBM 0.0.1 / 0.0.2**; **`SBM Manager` MVP** in **SBM 0.0.3**; full profile, module & ephemeral remote-recovery configuration in **SBM 0.1**.

---

## 1. Requirements for `SBM Manager`

The user-facing desktop application **`SBM Manager`** starts by simplifying verified USB creation and progressively expands into SBM's full provisioning and recovery companion:

1. Run on **macOS, Windows, and Linux** without requiring terminal commands (`dd`, `diskutil`).
2. Select and download verified **architecture-specific** SBM images (`x86_64` and `aarch64` primary; optional experimental dual-arch only behind an advanced toggle), or accept a local offline image file (`.raw`, `.raw.zst`, or `.raw.xz`).
3. Verify cryptographic integrity (**SHA-256** of compressed download and/or extracted raw image, plus TUF/signature verification) and perform **post-write read-back verification**.
4. Safely **filter out internal system drives** by default and handle OS-specific raw-disk privilege elevation (macOS Authorization Services / DiskArbitration unmount, Windows UAC / volume lock & dismount, Linux `udisks2` / Polkit).
5. Progressively configure **installation & recovery profiles** (`efi-only`, `net-ondemand`, `cached-usb`/`cached-ssd`, `ram-only`, `offline-full`, `modular-custom`), **optional signed modules**, **cache policies**, **pre-configured Wi-Fi/Ethernet recovery settings**, and **ephemeral, revocable remote-recovery session grants** (never permanent shared credentials).
6. Remain **offline-capable** (local image flashing works with zero internet access), **privacy-respecting** (no mandatory telemetry or third-party ads), and **license-compatible** with SBM's open-source policy.

---

## 2. Primary-Source Evaluation Matrix

| Project | Repository / Source | Stack & Approx. Size | License (SPDX) | Platforms | `.zst` Streaming | Custom Online Catalog / Repo | Write Verify | Telemetry / Ads | Assessment for SBM |
|---|---|---|---|---|---|---|---|---|---|
| **Raspberry Pi Imager** (`rpi-imager`) | [`raspberrypi/rpi-imager`](https://github.com/raspberrypi/rpi-imager) | C++ / Qt6 (`libarchive` + `libzstd` + `drivelist`) | `Apache-2.0` (bundled Qt LGPLv3, `libarchive`/`zstd` BSD) | macOS, Windows, Linux | **Yes** (v1.7+; v2.0.11+ fixed static `libzstd` linking & multi-frame `.zst` size parsing) | **Yes (native):** `--repo <URL\|file>` CLI and Imager 2.0+ GUI **App Options -> Content Repository** using formal JSON Schema (`os-list-schema.json`, v4) | **Yes** (SHA-256 download + extract + read-back verify) | Optional upstream telemetry in stock builds (configurable/removable in custom forks) | **Top immediate reuse tool.** Zero-code reuse in 0.0.1/0.0.2 via a published `sbm-os-list.json`; strong baseline for 0.0.3 flashing evaluation, though its RPi-centric UI does not natively model SBM's profile/module/remote-recovery configuration. |
| **Fedora Media Writer** | [`FedoraQt/MediaWriter`](https://github.com/FedoraQt/MediaWriter) | C++ / QML / Qt6 (`udisks2` on Linux, `xz-libs`) | `GPL-2.0-or-later` (`LGPL-2.0-or-later AND GPL-2.0-or-later`) | macOS, Windows, Linux | Local `.raw`/`.iso`/`.xz` (bundles `xz-libs`, not `zstd`-first) | No generic `--repo` override; hardcoded to Fedora feeds (supports custom local image) | **Yes** (download checksum + post-write read-back) | None | **Strong UX reference & third-party fallback.** Notable **Restore USB drive** feature (reformats flashed USB back to single FAT32). Forking requires maintaining a GPLv2+ Qt6 fork without native custom-repo JSON. |
| **balenaEtcher** | [`balena-io/etcher`](https://github.com/balena-io/etcher) | Electron (Chromium + Node.js), **~150–190 MB** | `Apache-2.0` | macOS, Windows, Linux | Yes (`.gz`, `.bz2`, `.xz`, `.zip`; check `.zst` per release) | No online OS catalog (local file or direct URL only) | **Yes** | Analytics code removed in v2.1.2 (May 2025); opt-out crash logs remain; **displays in-app Balena promotional panels** | **Do not adopt or recommend as primary.** Heavy Electron footprint (~150–190 MB) and unavoidable promotional UI panels conflict with SBM's minimal, ad-free UX. Document only as a compatible third-party tool with caveats. |
| **USBImager** | [`bztsrc/usbimager`](https://gitlab.com/bztsrc/usbimager) | Native C (GDI, Cocoa, GTK+/X11), **< 192 KB** | `MIT` | macOS, Windows, Linux | **Yes** (`.gz`, `.bz2`, `.xz`, `.zst`, `.zip`) | None (local files only) | **Yes** (optional verify checkbox) | None | **Best ultra-light offline third-party flasher.** Ideal recommendation for users who download `.raw.zst` manually and want zero bloat, zero telemetry, and native `.zst` streaming. |
| **GNOME Impression** | [`gitlab.gnome.org/World/impression`](https://gitlab.gnome.org/World/impression) (`io.github.adhami3310.Impression`) | Rust + GTK4 / libadwaita (`udisks2`), ~5–7 MB installed | `GPL-3.0-only` | **Linux only** | **Yes** (`.xz`, `.gz`, `.zst`) | Built-in distro list only (not custom repo) | Checksum verify | None | **Good Linux-only third-party recommendation**, but cannot serve macOS/Windows end users. |
| **KDE ISO Image Writer** | [`invent.kde.org/utilities/isoimagewriter`](https://invent.kde.org/utilities/isoimagewriter) | C++ / Qt6 + KDE Frameworks (~53–67 MB standalone) | `GPL-3.0-or-later` | Linux, Windows (**no macOS**) | Hybrid ISO focus | Checksum/GPG verification for supported distros | Yes | None | **Linux/Windows fallback only;** lacks macOS support and raw `.raw.zst` catalog flow. |
| **Rufus** | [`pbatard/rufus`](https://github.com/pbatard/rufus) | Native C (Win32) | `GPL-3.0-or-later` | **Windows only** | Yes (DD / raw image mode) | Fido script for Windows/UEFI Shell ISOs | Yes (SHA-256) | None | **Excellent Windows-only fallback** for manual raw/DD image flashing; not cross-platform. |
| **Caligula** | [`ifd3f/caligula`](https://github.com/ifd3f/caligula) | Rust TUI | `GPL-3.0-only` | Linux, macOS (no GUI) | Yes | Local files only | Yes | None | Useful developer TUI, not a consumer GUI provisioning app. |

---

## 3. Deep Dive: `rpi-imager` Reuse vs. Purpose-Built `SBM Manager`

### 3.1 Stock `rpi-imager` with a Custom SBM Repository (Zero Custom Code for 0.0.1–0.0.2)

`raspberrypi/rpi-imager` supports loading an external JSON repository without modifying or compiling Imager:
- **CLI:** `rpi-imager --repo https://smart-boot-menu.github.io/releases/os-list.json` (or a local file path for offline testing).
- **GUI (Imager 2.0+):** **App Options -> Content Repository -> Custom URL / Local File**.
- **Schema (`doc/json-schema/os-list-schema.json`):** Supports `name`, `description`, `url`, `icon`, `release_date`, `image_download_size`, `image_download_sha256`, `extract_size`, `extract_sha256`, `architecture`, and nested `subitems_url` groups.
- **`.zst` compatibility caveat:** Upstream `rpi-imager` has supported `.zst` since v1.7 via `libarchive` + `libzstd`, **but** regressions in v1.7.4.1 (#654), v1.9.0 (#1002), and macOS v2.0.7–v2.0.11rc1 (#1661) broke `.zst` when `libarchive` was built without static `libzstd`. Fixed upstream in **v2.0.11** (Aug 2026, commit `e5bc232`), which also added multi-frame `.zst` header-size parsing (#1605). Require **`rpi-imager >= 2.0.11`** when serving `.raw.zst` and omit `init_format` so Imager does not inject Raspberry Pi OS `cloud-init` / `firstrun.sh` customization into SBM's ESP.

### 3.2 Evaluating Minimal `rpi-imager` Fork vs. Thin Native `SBM Manager` (0.0.3 -> 0.1)

Because **`SBM Manager`** must eventually configure **recovery profiles, network settings, optional signed `systemd-sysext` modules, cache policies, installations, and ephemeral revocable remote-recovery grants** (in addition to flashing USB drives), the build-vs-reuse decision under `ADR-010` weighs two concrete options:

| Criterion | Option 1: Minimal `rpi-imager` Fork (`Apache-2.0`) | Option 2: Thin Native Rust/Iced `SBM Manager` (Reusing OS Flashing Backends/Helpers) |
|---|---|---|
| **Cross-platform raw disk I/O & elevation** | **Already solved & battle-tested** across macOS (DiskArbitration + helper auth), Windows (UAC + volume lock/dismount + `drivelist`), and Linux (`udisks2` + Polkit). | Must carefully reuse or wrap proven platform flashing backends (`udisks2` on Linux, DiskArbitration helper on macOS, Win32 DeviceIoControl + UAC on Windows, or invoking a bundled verified flasher backend) rather than reinventing fragile raw-disk elevation from scratch. |
| **SBM-specific configuration (profiles, modules, network recovery, ephemeral remote grants, TUF)** | **Poor structural fit beyond pure flashing:** `rpi-imager`'s customization model is built around Raspberry Pi `init_format` (`cloudinit-rpi`) and does not understand SBM's `ReleaseManifest`, TUF verification, `ModuleDescriptor` selection, cache policies, or ephemeral `sbm-server` session grants. | **Natural fit:** Shares `sbm-common` types, `sbm-release` TUF verifier, `sbm-media` ESP writer, and SBM's first-principles Iced design system (testable natively in Track A!). |
| **Maintenance & TCO** | Lowest initial effort for **pure flashing**, but diverges sharply once profile/module/remote-recovery configuration is added; requires maintaining a Qt6/CMake C++ fork alongside SBM's Rust workspace. | Higher initial flashing-backend integration care, but single unified Rust workspace, zero Qt6 packaging bloat, and direct TUF/manifest/module reuse. |

---

## 4. Phased Recommendation (Aligned With ADR-010 & ADR-012)

1. **SBM 0.0.1 & 0.0.2 (Reuse Directly — Zero Custom Flasher Code):**
   - Publish a schema-validated **`sbm-os-list.json`** alongside each release referencing the independently qualified `x86_64` and `aarch64` images (`extract_sha256`, `image_download_sha256`, no `init_format`).
   - Document verified flashing instructions for **Raspberry Pi Imager (`>= 2.0.11`, `--repo` or App Options)**, **USBImager** (ultra-light offline `.raw.zst` on macOS/Windows/Linux), **Fedora Media Writer**, **GNOME Impression** (Linux), and **Rufus** (Windows).
2. **SBM 0.0.3 (`SBM Manager` MVP — Empirical Gate Before Custom Flashing Engine):**
   - Benchmark a **minimal `Apache-2.0` branded `rpi-imager` trial** for pure USB flashing against a **thin Rust/Iced `SBM Manager` UI** backed by existing OS flashing primitives (`udisks2`, DiskArbitration, Win32 dismount / verified helper) so we never reinvent low-level raw-disk quirks while still supporting SBM's **recovery profiles, cache policies, optional modules, and ephemeral remote-recovery configuration** cleanly.
   - Start `SBM Manager` MVP focused on verified per-arch download, profile selection, USB flashing + read-back verification, and USB drive restore; expand in **SBM 0.1** to preconfigure network recovery credentials, signed `systemd-sysext` module bundles, and revocable remote-recovery session grants.

---

## 5. Primary Sources

- Raspberry Pi Imager repository, license, JSON schema & schema notes: https://github.com/raspberrypi/rpi-imager (`doc/json-schema/os-list-schema.json`, `doc/schema-notes.md`, `license.txt`)
- Raspberry Pi Imager custom repository documentation: https://www.raspberrypi.com/documentation/computers/getting-started.html#raspberry-pi-imager
- Raspberry Pi Imager zstd/libarchive fixes (`#654`, `#1002`, `#1605`, `#1661`, v2.0.11): https://github.com/raspberrypi/rpi-imager/releases
- Fedora Media Writer repository & license: https://github.com/FedoraQt/MediaWriter
- balenaEtcher repository & analytics removal (`#4460`, v2.1.2): https://github.com/balena-io/etcher
- USBImager repository & documentation: https://gitlab.com/bztsrc/usbimager
- GNOME Impression: https://gitlab.gnome.org/World/impression
- KDE ISO Image Writer: https://invent.kde.org/utilities/isoimagewriter
- Rufus: https://github.com/pbatard/rufus and https://rufus.ie/
- Caligula: https://github.com/ifd3f/caligula
