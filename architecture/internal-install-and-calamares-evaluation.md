# Internal SSD Installation, Partitioning Tools & Calamares Evaluation

**Date:** 2026-11-26  
**Scope:** Primary-source build-vs-reuse evaluation of **Calamares** and mature Linux partitioning/filesystem/UEFI utilities for **SBM 0.0.2** (persistent internal SSD installation into available space, firmware boot registration, uninstall & recovery) and **SBM 0.0.3** (guided partition resizing for explicitly supported configurations), per [`ADR-006`](https://github.com/smart-boot-menu/docs/blob/main/architecture-decisions/ADR-006-storage-safety-and-tool-wrapping.md) and [`ADR-010`](https://github.com/smart-boot-menu/docs/blob/main/architecture-decisions/ADR-010-open-source-reuse-and-engineering-principles.md).

---

## 1. Problem Scope: What Internal SBM Installation Actually Requires

Installing Smart Boot Menu onto an internal SSD alongside existing operating systems is **fundamentally different** from installing a traditional desktop Linux distribution:

- **SBM 0.0.2 (Safe Available-Space Install + Uninstall + Recovery):**
  - Inspect existing GPT disks, EFI System Partitions (ESPs), OS bootloaders (Windows Boot Manager, Linux BLS/GRUB/`systemd-boot`), encryption markers (BitLocker `bitlk`, LUKS2), and Apple protected containers (`ProtectedAppleSystem`) — **strictly read-only during planning**.
  - Allocate dedicated SBM partitions **only inside existing unallocated GPT space** (or an explicitly selected empty disk) — **zero resizing or shrinking of existing OS partitions in 0.0.2**.
  - Install `sbm-uefi` (`\EFI\sbm\sbm-<arch>.efi`) and `sbm-core` UKI + `dm-verity` EROFS root/hash partitions (either coexisting cleanly on an existing ESP with verified free space without touching `\EFI\Microsoft\` or `\EFI\<distro>\`, or creating a dedicated SBM ESP when appropriate).
  - Register an EFI NVRAM boot entry via `efibootmgr`, record an immutable install receipt (`\EFI\sbm\install-receipt.json` capturing prior `BootOrder`, created PARTUUIDs, and ESP files), and support clean **Uninstall** (restoring prior `BootOrder`, deleting SBM's NVRAM entry, removing `\EFI\sbm\`, and optionally removing SBM's labeled recovery partitions) and **Recovery** (re-registering SBM if Windows Update resets `BootOrder`).
- **SBM 0.0.3 (Guided Partition Resizing for Explicitly Supported Configurations):**
  - When a disk has no unallocated space, allow the user to inspect partitions and optionally shrink a single **explicitly supported, clean, unencrypted** OS partition to free space (~1.5–2.5 GiB for single-arch SBM recovery + headroom) before running the exact same 0.0.2 installation pipeline.

---

## 2. Critical Evaluation of Calamares (`3.4.3`)

**Primary Source:** [`codeberg.org/Calamares/calamares`](https://codeberg.org/Calamares/calamares) (Calamares `3.4.3`, released September 9, 2026; migrated from GitHub to Codeberg).  
**Stack:** C++17, CMake, **Qt 6** (`qt6-base`, `qt6-declarative`, `qt6-svg`, `polkit-qt6`), **KDE Frameworks 6** (`kconfig`, `kcoreaddons`, `ki18n`, `kio`, `kdbusaddons`, `kcrash`, `solid`), **KPMcore** (KDE Partition Manager Core + `pmsfdiskbackendplugin`), `yaml-cpp`, and Python (`pybind11`/`boost`).  
**License:** Primarily **`GPL-3.0-or-later`** (with mixed `BSD-2-Clause` / `LGPL` / `MIT` subcomponents).

### 2.1 Architectural Strengths Worth Studying

1. **Strict Deferred Execution (`show` vs. `exec`):** Calamares separates interactive planning (`show` view modules: `welcome`, `partition`, `summary`) from non-interactive mutation (`exec` job modules). Zero disk writes occur until the user reviews the summary and confirms.
2. **Pre-Execution Quiescing:** Before touching partition tables, Calamares runs `AutoMountManagementJob` (disables desktop automounting) and `ClearMountsJob` (unmounts target filesystems and closes active device mappers) before invoking KPMcore resize/create/format jobs.
3. **"Install Alongside" UX:** When an eligible partition has enough free space above `requiredStorage`, Calamares presents a visual partition bar with a bounded drag slider showing before/after sizes rather than exposing raw sector math to consumers.
4. **Real-World Partitioning Edge-Case Lessons:** Even mature frameworks hit subtle bugs — e.g., Calamares 3.4.3 fixed an issue where `/boot` could be inadvertently marked as an EFI System Partition, breaking `systemd-gpt-auto-generator`.

### 2.2 Why SBM Should NOT Adopt the Full Calamares Framework

| Dimension | Calamares Full-Framework Adoption | SBM Rust Planner (`sbm-plan`/`sbm-exec`) + Mature CLI Tools |
|---|---|---|
| **Image Size & RAM Footprint** | Pulls **Qt 6 + KDE Frameworks 6 + KPMcore + Python** into `sbm-core`, adding a third GUI toolkit stack alongside `labwc` + Iced (Rust) + GTK3 (GParted) and pushing against `sbm-core`'s `< 650 MiB` compressed / `≤ 1 GiB` low-RAM targets. | Zero additional GUI/framework stack — renders directly inside the existing **Iced `sbm-ui`** shell (also runnable natively in Track A host preview on macOS/Linux!). |
| **Domain Fit** | Engineered for full Linux distro installation (`unpackfs`, `locale`, `users`, `fstab`, `initramfs`, `grubcfg`, `packages`). Has no native model for SBM's immutable `dm-verity` EROFS slots, Spec 03 recovery manifests, non-destructive ESP coexistence, install receipts, or **uninstall + NVRAM restoration**. | Purpose-built for SBM's exact domain while sharing `sbm-common` typed contracts (`DiskIdentity`, `PartitionProtectionClass`, `StorageOperationPlan`). |
| **Security & Safety Model** | Uses untyped in-memory `GlobalStorage` key-value maps across C++/Python modules; lacks SBM's optimistic concurrency lock (`expected_partition_table_sha256`), composite `DiskIdentity` matching (`wwn`/`eui64`/`gpt`), and Apple/BitLocker hard-block policy types. | Preserves SBM's deterministic, unit-testable, property-testable Rust planner and strict security boundaries (already verified with 61/61 tests in `install`!). |
| **Licensing Boundary** | `GPL-3.0-or-later`; cannot link into SBM's `MIT OR Apache-2.0` Rust crates, and maintaining custom C++/QML/Python modules introduces cross-language glue. | Original planner/UI stays `MIT OR Apache-2.0`; GPL-2.0/3.0 partitioning and filesystem tools remain cleanly isolated as `execve` subprocesses (per `ADR-006` and `ADR-010`). |
| **Testability (Tier 1 & Tier 2)** | Requires a full Qt6/KPMcore/Linux desktop session to run end-to-end UI tests. | Planner & Iced wizard run natively on macOS/Linux in milliseconds (Tier 1 simulation) and execute real CLI tools against disposable QEMU QCOW2 disks (Tier 2). |

**Verdict on Calamares:** **Study UX and quiescing patterns; do NOT adopt the Calamares framework or KPMcore stack into `sbm-core`.**

---

## 3. Mature Open-Source Partitioning, Filesystem & Firmware Tools to Reuse

In strict compliance with `ADR-006` and `ADR-010`, SBM **must NOT rewrite** low-level partition table writers, filesystem resizers, or EFI NVRAM managers in Rust. Instead, `sbm-exec` / `sbm-cored` invokes mature, distribution-maintained utilities via direct `execve` (`Command::args`, zero shell interpolation):

| Capability | Primary Tool(s) | License | Milestone | Role & Safety Notes |
|---|---|---|---|---|
| **Read-only disk & FS discovery** | `lsblk --json`, `blkid`, `sfdisk --json`, `sgdisk -p`, `cryptsetup isLuks` / `bitlk` probe | `GPL-2.0` / `LGPL-2.1` | 0.0.2 | Populate `DiskIdentity`, `PartitionIdentity`, `PartitionProtectionClass`, and compute `expected_partition_table_sha256` before any plan is built. |
| **GPT backup & partition creation/edit** | `sgdisk` (`gptfdisk`) and/or `sfdisk` (`util-linux`), `partprobe` / `udevadm settle` | `GPL-2.0` | 0.0.2 & 0.0.3 | Always save a binary GPT backup (`sgdisk --backup=...`) to the install staging/receipt bundle before mutating. Set exact DPS type GUIDs, unique PARTUUIDs, and partition attributes (`sbm-common#2`) for `dm-verity` root/hash pairs. Never mis-tag non-ESP partitions with the ESP type GUID. |
| **Declarative GPT provisioning (evaluated)** | `systemd-repart` | `LGPL-2.1-or-later` | 0.0.2+ (optional) | Excellent for declarative image/partition creation on empty disks or unallocated tail space; verify whether `sfdisk`/`sgdisk` gives finer control over exact placement and rollback receipts alongside existing OS partitions. |
| **FAT32 ESP creation & check** | `mkfs.fat -F 32`, `fsck.fat` (`dosfstools`) | `GPL-3.0-or-later` | 0.0.2 | Used only when creating a new dedicated SBM ESP; never formats an existing OS ESP (`ProtectedEsp`). |
| **UEFI NVRAM registration & uninstall** | `efibootmgr` | `GPL-2.0-or-later` | 0.0.2 | Read existing `BootOrder` before mutation; create SBM entry (`-c -d ... -p ... -L "Smart Boot Menu" -l '\EFI\sbm\...'`); restore prior `BootOrder` and delete entry (`-b XXXX -B`) on uninstall; re-register on recovery. |
| **ext4 offline shrink/expand** | `e2fsck -f`, `resize2fs` (`e2fsprogs`) | `GPL-2.0-only` | **0.0.3** | Requires unmounted partition + forced `e2fsck -f` before shrinking. |
| **NTFS offline shrink/expand** | `ntfsresize` (`ntfs-3g` / `ntfsprogs`) | `GPL-2.0-or-later` | **0.0.3** | Requires unmounted, clean NTFS (refuses dirty bit or Windows Fast Startup / hibernation). Always run `--check`, `--info`, and `--no-action` dry-run first; inform user that Windows will run `chkdsk` on next boot. |
| **Btrfs shrink/expand** | `btrfs filesystem usage` / `resize` (`btrfs-progs`) | `GPL-2.0-only` | **0.0.3** | Unlike ext4/NTFS, `btrfs filesystem resize` requires the filesystem to be **mounted** (mount -> shrink -> unmount -> shrink partition -> remount -> `resize max`). Single-device volumes only in 0.0.3. |
| **Interactive manual escape hatch** | **GParted `1.8.1`** (already bundled & verified in `sbm-core`) | `GPL-2.0-or-later` | 0.0.1+ | For unsupported/complex layouts (LUKS, LVM, multi-partition moves), users can launch GParted directly from `sbm-core` to prepare unallocated space manually, then run SBM's 0.0.2 free-space installer! |
| **Higher-level D-Bus/C wrapper (evaluated)** | `libblockdev` (`LGPL-2.1-or-later`) + `udisks2` (`GPL-2.0-or-later`) | `LGPL-2.1` / `GPL-2.0` | Revisit post-0.0.3 | Provides uniform D-Bus/plugin resize wrappers (`bd_part_resize_part`, `bd_fs_resize`, `CanResize`), which themselves invoke `e2fsprogs`/`ntfs-3g`/`btrfs-progs`/`libfdisk`. Direct `execve` from `sbm-exec`/`sbm-cored` avoids a D-Bus/Polkit daemon dependency inside minimal `sbm-core`, keeps dry-run/logging transparent, and simplifies Tier 1/Tier 2 testing. Re-evaluate only if storage backend complexity outgrows direct CLI wrapping. |

---

## 4. Safety Invariants & Dual-Boot Pitfalls for 0.0.2 and 0.0.3

### 4.1 The Golden Rule of Safe Partition Shrinking (SBM 0.0.3)

Filesystem tools (`resize2fs`, `ntfsresize`, `btrfs`) modify **only the filesystem**, never the GPT partition table. Shrinking a partition before shrinking its filesystem truncates data immediately. Every 0.0.3 guided shrink **MUST** execute the following exact sequence (matching GParted/KPMcore best practice):

1. **Quiesce & lock:** Confirm target partition is unmounted (except Btrfs during its online shrink step), verify live `DiskIdentity` and `expected_partition_table_sha256` match the approved plan, and back up GPT (`sgdisk --backup`).
2. **Pre-flight integrity check:** Run read-only FS consistency and minimum-size probes (`e2fsck -f`, `ntfsresize --check --info --no-action`, `btrfs filesystem usage`). Abort cleanly with zero writes on any dirty/hibernated/insufficient-headroom condition.
3. **Shrink filesystem with safety margin:** Shrink the filesystem to `target_partition_size - safety_margin` (e.g., 64–128 MiB margin to eliminate SI decimal vs. binary IEC rounding hazards, especially since `ntfsresize` uses SI units).
4. **Shrink partition keeping identical start sector:** Update only the partition **end sector** (1 MiB aligned, never moving the start sector), then run `partprobe` / `udevadm settle`.
5. **Expand filesystem to exact partition boundary:** Run `resize2fs <part>` / `ntfsresize <part>` / `btrfs filesystem resize max <mnt>` so the filesystem fills the exact new partition boundary cleanly.
6. **Verify:** Re-read partition table and filesystem metadata before proceeding to SBM partition creation.

### 4.2 Explicitly Supported vs. Hard-Blocked Resize Configurations (SBM 0.0.3)

- **Explicitly Supported (allowlisted in 0.0.3):**
  - Unencrypted, clean **ext4** partition on a standard GPT disk.
  - Unencrypted, clean **NTFS** partition on a standard GPT disk with **Windows Fast Startup / hibernation disabled** (no `hiberfil.sys` / dirty flag) and **BitLocker disabled**.
  - Unencrypted, clean, single-device **Btrfs** partition on a standard GPT disk with sufficient unallocated headroom.
- **Explicitly Hard-Blocked in Guided Resize (must refuse with clear remediation guidance):**
  - **BitLocker (`bitlk`) / Windows Device Encryption** — cannot be safely shrunk by Linux tools; modifying NVRAM/ESP may also trigger a BitLocker Recovery Key prompt. Direct users to back up their BitLocker Recovery Key and shrink from inside Windows Disk Management first, then use SBM's 0.0.2 free-space install.
  - **LUKS / dm-crypt**, **LVM**, **MDADM software RAID**, **ZFS**, **Windows Dynamic Disks / Storage Spaces**.
  - **NTFS with dirty flag or Windows Fast Startup hibernation state.**
  - **XFS** (cannot be shrunk).
  - **Apple APFS / `ProtectedAppleSystem`** (`iSCPreboot`, `RecoveryOS`, macOS containers — always hard-blocked).
  - **Legacy MBR / BIOS partition tables** (UEFI GPT only).
  - **Active ESP (`ProtectedEsp`)** or currently booted SBM recovery medium (`ProtectedSbmRecovery`).

### 4.3 Windows Coexistence & Firmware Boot Registration Pitfalls (SBM 0.0.2)

- **Never overwrite `\EFI\Microsoft\Boot\bootmgfw.efi` or `\EFI\BOOT\BOOTX64.EFI` on an existing OS ESP:** Install SBM into `\EFI\sbm\` and register a distinct NVRAM entry via `efibootmgr`, preserving existing Windows and Linux bootloaders untouched so both chainloading and uninstall are clean.
- **BitLocker warning before `BootOrder` change:** Even non-destructive `efibootmgr` boot-order changes can trigger a BitLocker Recovery Key prompt on systems measuring PCR 4 / boot order. SBM's 0.0.2 pre-install check MUST detect BitLocker partitions on the machine and display an explicit warning advising the user to confirm they have their BitLocker Recovery Key before proceeding.
- **Windows Updates resetting `BootOrder`:** Windows feature updates routinely move `Windows Boot Manager` back to first in UEFI `BootOrder`. SBM's **Recovery / Repair Boot Entry** action (boot SBM USB -> **Repair SBM Boot Registration**) solves this cleanly in seconds using `efibootmgr` without reinstalling partitions.

---

## 5. Primary Sources

- Calamares repository & releases (`3.4.3`, Sep 2026): https://codeberg.org/Calamares/calamares and https://calamares.io/
- KPMcore (KDE Partition Manager Core): https://invent.kde.org/system/kpmcore
- `util-linux` (`sfdisk`, `lsblk`, `blkid`): https://github.com/util-linux/util-linux
- `gptfdisk` (`sgdisk`): https://www.rodsbooks.com/gdisk/
- `e2fsprogs` (`e2fsck`, `resize2fs`): https://e2fsprogs.sourceforge.net/
- `ntfs-3g` / `ntfsresize`: https://github.com/tuxera/ntfs-3g
- `btrfs-progs` (`btrfs-filesystem(8)`): https://btrfs.readthedocs.io/
- `efibootmgr`: https://github.com/rhboot/efibootmgr
- `libblockdev` & `udisks2` (`storaged-project`): https://github.com/storaged-project/libblockdev and https://github.com/storaged-project/udisks
- `systemd-repart(8)`: https://www.freedesktop.org/software/systemd/man/latest/systemd-repart.html
