# Media Creator & USB Flashing Tools Evaluation (Build vs. Reuse)

**Date:** 2026-11-26  
**Scope:** Primary-source evaluation of existing open-source USB/SD image flashing tools against Smart Boot Menu's end-user installation journey (`Download Media Creator -> flash USB -> boot target computer -> Install SBM`) per [`ADR-010`](https://github.com/smart-boot-menu/docs/blob/main/architecture-decisions/ADR-010-open-source-reuse-and-engineering-principles.md).  
**Target Milestones:** Immediate reuse in **SBM 0.0.1 / 0.0.2**; dedicated Media Creator decision for **SBM 0.0.3**.

---

## 1. Requirements for an SBM Media Creator

An end-user Media Creator for Smart Boot Menu must:

1. Run on **macOS, Windows, and Linux** without requiring terminal commands (`dd`, `diskutil`).
2. Select and download verified **architecture-specific** SBM images (`x86_64` and `aarch64` primary; optional experimental dual-arch only behind an advanced toggle), or accept a local offline image file (`.raw`, `.raw.zst`, or `.raw.xz`).
3. Verify cryptographic integrity (**SHA-256** of compressed download and/or extracted raw image, and future TUF/signature verification) and perform **post-write read-back verification**.
4. Safely **filter out internal system drives** by default and handle OS-specific raw-disk privilege elevation (macOS Authorization Services / DiskArbitration unmount, Windows UAC / volume lock & dismount, Linux `udisks2` / Polkit).
5. Remain **offline-capable** (local image flashing works with zero internet access), **privacy-respecting** (no mandatory telemetry or third-party ads), and **license-compatible** with SBM's open-source distribution policy (`MIT OR Apache-2.0` for original code; standalone desktop tools under `Apache-2.0` or `GPL` are acceptable if clearly isolated from SBM Rust libraries).

---

## 2. Primary-Source Evaluation Matrix

| Project | Repository / Source | Stack & Approx. Size | License (SPDX) | Platforms | `.zst` Streaming | Custom Online Catalog / Repo | Write Verify | Telemetry / Ads | Assessment for SBM |
|---|---|---|---|---|---|---|---|---|---|
| **Raspberry Pi Imager** (`rpi-imager`) | [`raspberrypi/rpi-imager`](https://github.com/raspberrypi/rpi-imager) | C++ / Qt6 (`libarchive` + `libzstd` + `drivelist`) | `Apache-2.0` (bundled Qt LGPLv3, `libarchive`/`zstd` BSD) | macOS, Windows, Linux | **Yes** (v1.7+; v2.0.11+ fixed static `libzstd` linking & multi-frame `.zst` size parsing) | **Yes (native):** `--repo <URL\|file>` CLI and Imager 2.0+ GUI **App Options -> Content Repository** using formal JSON Schema (`os-list-schema.json`, v4) | **Yes** (SHA-256 download + extract + read-back verify) | Optional upstream telemetry in stock builds (configurable/removable in custom forks) | **Top reuse candidate.** Immediate zero-code reuse in 0.0.1/0.0.2 via a published `sbm-os-list.json`; strongest candidate for a minimal branded Apache-2.0 fork in 0.0.3. |
| **Fedora Media Writer** | [`FedoraQt/MediaWriter`](https://github.com/FedoraQt/MediaWriter) | C++ / QML / Qt6 (`udisks2` on Linux, `xz-libs`) | `GPL-2.0-or-later` (`LGPL-2.0-or-later AND GPL-2.0-or-later`) | macOS, Windows, Linux | Local `.raw`/`.iso`/`.xz` (bundles `xz-libs`, not `zstd`-first) | No generic `--repo` override; hardcoded to Fedora feeds (supports custom local image) | **Yes** (download checksum + post-write read-back) | None | **Strong UX reference & third-party fallback.** Notable **Restore USB drive** feature (reformats flashed USB back to single FAT32). Forking requires maintaining a GPLv2+ Qt6 fork without native custom-repo JSON. |
| **balenaEtcher** | [`balena-io/etcher`](https://github.com/balena-io/etcher) | Electron (Chromium + Node.js), **~150–190 MB** | `Apache-2.0` | macOS, Windows, Linux | Yes (`.gz`, `.bz2`, `.xz`, `.zip`; check `.zst` per release) | No online OS catalog (local file or direct URL only) | **Yes** | Analytics code removed in v2.1.2 (May 2025); opt-out crash logs remain; **displays in-app Balena promotional panels** | **Do not adopt or recommend as primary.** Heavy Electron footprint (~150–190 MB) and unavoidable promotional UI panels conflict with SBM's minimal, ad-free UX. Document only as a compatible third-party tool with caveats. |
| **USBImager** | [`bztsrc/usbimager`](https://gitlab.com/bztsrc/usbimager) | Native C (GDI, Cocoa, GTK+/X11), **< 192 KB** | `MIT` | macOS, Windows, Linux | **Yes** (`.gz`, `.bz2`, `.xz`, `.zst`, `.zip`) | None (local files only) | **Yes** (optional verify checkbox) | None | **Best ultra-light offline third-party flasher.** Ideal recommendation for users who download `.raw.zst` manually and want zero bloat, zero telemetry, and native `.zst` streaming. |
| **GNOME Impression** | [`gitlab.gnome.org/World/impression`](https://gitlab.gnome.org/World/impression) (`io.github.adhami3310.Impression`) | Rust + GTK4 / libadwaita (`udisks2`), ~5–7 MB installed | `GPL-3.0-only` | **Linux only** | **Yes** (`.xz`, `.gz`, `.zst`) | Built-in distro list only (not custom repo) | Checksum verify | None | **Good Linux-only third-party recommendation**, but cannot serve macOS/Windows end users. |
| **KDE ISO Image Writer** | [`invent.kde.org/utilities/isoimagewriter`](https://invent.kde.org/utilities/isoimagewriter) | C++ / Qt6 + KDE Frameworks (~53–67 MB standalone) | `GPL-3.0-or-later` | Linux, Windows (**no macOS**) | Hybrid ISO focus | Checksum/GPG verification for supported distros | Yes | None | **Linux/Windows fallback only;** lacks macOS support and raw `.raw.zst` catalog flow. |
| **Rufus** | [`pbatard/rufus`](https://github.com/pbatard/rufus) | Native C (Win32) | `GPL-3.0-or-later` | **Windows only** | Yes (DD / raw image mode) | Fido script for Windows/UEFI Shell ISOs | Yes (SHA-256) | None | **Excellent Windows-only fallback** for manual raw/DD image flashing; not cross-platform. |
| **Caligula** | [`ifd3f/caligula`](https://github.com/ifd3f/caligula) | Rust TUI | `GPL-3.0-only` | Linux, macOS (no GUI) | Yes | Local files only | Yes | None | Useful developer TUI, not a consumer GUI Media Creator. |

---

## 3. Deep Dive: Raspberry Pi Imager Custom Repositories & Fork Trade-Offs

### 3.1 Stock `rpi-imager` with a Custom SBM Repository (Zero Custom Code)

`raspberrypi/rpi-imager` supports loading an external JSON repository without modifying or compiling Imager:
- **CLI:** `rpi-imager --repo https://smart-boot-menu.github.io/releases/os-list.json` (or a local file path for offline testing).
- **GUI (Imager 2.0+):** **App Options -> Content Repository -> Custom URL / Local File**.
- **Schema (`doc/json-schema/os-list-schema.json`):** Supports `name`, `description`, `url`, `icon`, `release_date`, `image_download_size`, `image_download_sha256`, `extract_size`, `extract_sha256`, `architecture`, and nested `subitems_url` groups.
- **`.zst` compatibility caveat:** Upstream `rpi-imager` has supported `.zst` since v1.7 via `libarchive` + `libzstd`, **but** regressions in v1.7.4.1 (#654), v1.9.0 (#1002), and macOS v2.0.7–v2.0.11rc1 (#1661) broke `.zst` when `libarchive` was built without static `libzstd`. This was fixed upstream in **v2.0.11** (Aug 2026, commit `e5bc232`), which also added multi-frame `.zst` header-size parsing (#1605). Therefore:
  - Require **`rpi-imager >= 2.0.11`** when serving `.raw.zst`, **or** verify compatibility before recommending `.zst` vs `.xz` in `sbm-os-list.json`.
  - Omit `init_format` for SBM images so Imager does **not** attempt Raspberry Pi OS `cloud-init` / `firstrun.sh` customization on SBM's ESP!

### 3.2 Limitations of Stock `rpi-imager` for Consumer 0.0.3 UX

1. **Branding & Mental Model:** Stock Imager opens with a "Choose Device (Raspberry Pi 5/4/3...)" step unless filtered/configured, and requires users to either pass `--repo` or manually paste a custom repository URL in App Options — adding friction to the target one-click UX (`Download Media Creator -> flash USB`).
2. **No Built-In USB Restore Button:** Unlike Fedora Media Writer, stock `rpi-imager` focuses on flashing (it does offer "Erase -> Format as FAT32" as an OS option, which works cleanly).
3. **No TUF Verification:** `rpi-imager` checks HTTPS + SHA-256 fields from the JSON list, not SBM's TUF metadata chain (`sbm-release`).

### 3.3 Minimal Branded Fork of `rpi-imager` vs. Custom Thin Rust/Iced Media Creator

Per `ADR-010`, we must compare any proposed original SBM Media Creator against the **strongest existing alternative** — a minimal Apache-2.0 fork of `rpi-imager` (precedents: `CardputerZero/m5stack-imager`, `Web3-Pi`, `guysoft/pi-imager`):

| Criterion | Minimal `rpi-imager` Fork (`Apache-2.0`) | Custom Thin Rust/Iced Media Creator (`MIT OR Apache-2.0`) |
|---|---|---|
| **Cross-platform raw disk I/O & elevation** | **Already solved & battle-tested** across macOS (DiskArbitration + helper auth), Windows (UAC + volume lock/dismount + `drivelist`), and Linux (`udisks2` + Polkit). | **High engineering & safety risk:** Must implement and maintain macOS Authorization/DADisk unmount, Windows DeviceIoControl volume locking + UAC manifest, and Linux `udisks2` D-Bus helpers across all three OSes. |
| **Maintenance & TCO** | Rebase small branding/config diff (`src/config.h` default `OSLIST_URL`, telemetry off, SBM icons/strings, hide RPi device list) onto upstream `rpi-imager` tags; requires CMake + Qt6 CI builds + macOS/Windows code signing. | Own 100% of GUI + raw block-device writer + OS quirk bugs forever; reuses `sbm-common` `ReleaseManifest` & `sbm-release` TUF crate directly; smaller binary than Qt6. |
| **User Experience** | Familiar 3-step wizard (`Choose SBM Image -> Choose USB -> Write & Verify`), native look on all 3 OSes, built-in streaming `.zst`/`.xz` + SHA-256 verify. | Tailored SBM-only UX (auto-detects architecture guidance, built-in TUF verification, "Restore USB" button like Fedora Media Writer, shared SBM theme). |
| **Licensing** | `Apache-2.0` (bundled Qt6 under LGPLv3 dynamic linking per upstream). Completely separate desktop binary from `sbm-uefi`/`sbm-core`. | `MIT OR Apache-2.0`. |

---

## 4. Phased Recommendation (Aligned With ADR-010)

1. **SBM 0.0.1 & 0.0.2 (Reuse Directly — Zero Custom Flasher Code):**
   - Publish a schema-validated **`sbm-os-list.json`** alongside each release referencing the independently qualified `x86_64` and `aarch64` images (`extract_sha256`, `image_download_sha256`, no `init_format`).
   - Document verified flashing instructions for **Raspberry Pi Imager (`>= 2.0.11`, `--repo` or App Options)**, **USBImager** (ultra-light offline `.raw.zst` on macOS/Windows/Linux), **Fedora Media Writer**, **GNOME Impression** (Linux), and **Rufus** (Windows).
   - Keep `install`'s Rust CLI (`sbm-install media`) focused on reproducible image assembly, verification, and developer/CI workflows — do **not** rush a custom GUI flasher into 0.0.1 or 0.0.2.
2. **SBM 0.0.3 (Empirical Gate Before Building Custom):**
   - **Default path (Preference Level 5 before Level 6):** Prototype a **minimal Apache-2.0 branded build/fork of `rpi-imager`** pointing to SBM's release JSON with telemetry disabled, RPi hardware prompts removed, and SBM's per-architecture images (`x86_64`, `aarch64`, optional experimental dual-arch toggle). Measure installer size, CI build complexity, and end-user UX across macOS, Windows, and Linux.
   - **Only justify an original Rust/Iced Media Creator** if the empirical `rpi-imager` trial proves unacceptable UX, Qt6 packaging/signing TCO, or security-verification gaps (such as mandatory TUF verification before writing) that cannot be cleanly solved upstream or in a thin fork.

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
