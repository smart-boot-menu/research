# Recovery & Storage Tools Research

**Last verified:** 2026-11-26  
**Scope:** Partition managers, recovery live distributions, disk cloning/imaging tools, filesystem utilities, encryption tooling, and Rust storage libraries for `sbm-core`.

---

## Summary Comparison Matrix

| Project | Primary Language(s) | License (SPDX) | Latest Verified Release (Nov 2026) | Verdict | Integration Strategy in `sbm-core` |
|---|---|---|---|---|---|
| **GParted / GParted Live** | C++ (`gtkmm3`, `libparted`) | `GPL-2.0-or-later` | GParted `1.8.1` (Mar 2026); Live `1.8.1-6` (Aug 2026) | **Wrap / Bundle** | Bundle `gparted` package as a standalone launchable recovery app in `sbm-core` (requires XWayland under Wayland compositor) |
| **SystemRescue** | Shell, Python (Arch Linux base) | `GPL-2.0-or-later` (mixed distro) | Active (`11.x` / `12.x` series) | **Study** | Study curated CLI/GUI recovery tool list and RAM-copy (`copytoram`) boot workflow |
| **Rescuezilla** | Python (GTK3), Shell | `GPL-3.0-only` | Active (`2.5.x` / `2.6.x`) | **Study** | Study user-friendly backup/restore UX and Clonezilla image format compatibility |
| **Clonezilla** | Shell, Perl | `GPL-2.0-only` | Active | **Study** | Maintain compatibility with Clonezilla/Partclone image directory format where practical |
| **Partclone** | C | `GPL-2.0-or-later` | Active (`0.3.x`) | **Wrap** | Invoke `partclone.*` binaries from `sbm-core` Rust backup engine for smart used-block cloning |
| **GNU Parted (`libparted`)** | C | `GPL-3.0-or-later` | Active (`3.6`+) | **Wrap** | Used by GParted; prefer `util-linux` (`sfdisk`) + `gptfdisk` (`sgdisk`) for scripted GPT manipulation |
| **`util-linux`** (`sfdisk`, `lsblk`, `blkid`, `wipefs`, `mount`) | C | `GPL-2.0-or-later` / `LGPL-2.1-or-later` / `BSD-3-Clause` | Active (`2.40`+) | **Reuse / Wrap** | Canonical JSON discovery (`lsblk -J`, `sfdisk -J`, `blkid`) and safe partition scripting |
| **TestDisk & PhotoRec** | C | `GPL-2.0-or-later` | `7.2` (Feb 2024; `7.3-WIP` active) | **Wrap / Bundle** | Include in `sbm-core` recovery module for lost partition & file recovery |
| **GNU `ddrescue`** | C++ | `GPL-2.0-or-later` | Active (`1.28`+) | **Wrap / Bundle** | Include in `sbm-core` for failing-disk sector rescue with mapfiles |
| **ZFSBootMenu** | Perl, Shell (Linux `kexec`) | `MIT` | `v3.1.0` (2026) | **Study / Optional Chainload** | Study ZFS Boot Environment discovery; `sbm-uefi` can chainload ZFSBootMenu EFI builds directly |
| **Parted Magic** (rebranding *Erase Station* Nov 2026) | Mixed Linux distro | Commercial / Proprietary distribution | Active commercial (Nov 2026 rebrand notice) | **Avoid** (Study feature checklist) | Reference for recovery UX completeness (NVMe Secure Erase, SMART diagnostics) |
| **BootIt UEFI** (TeraByte Unlimited) | Proprietary C/C++ | Commercial / Proprietary | Active commercial | **Avoid** (Study feature checklist) | Reference for pre-OS partitioning/imaging UX |
| **`cryptsetup`** (LUKS2 + BitLocker `bitlk`) | C | `GPL-2.0-or-later` (lib: `LGPL-2.1-or-later`) | `2.8.9` (2026-11-23) | **Wrap / Reuse** | Native kernel `dm-crypt` unlock for LUKS2 AND Windows BitLocker (`bitlk`) |
| **`e2fsprogs`** | C | `GPL-2.0-or-later` / `LGPL` / `MIT` | `1.47.4` (2026-03-06) | **Wrap** | `mkfs.ext4`, `e2fsck`, `resize2fs` — never rewrite in Rust |
| **`dosfstools` / `ntfs-3g` / `exfatprogs` / `btrfs-progs` / `gptfdisk`** | C / C++ | `GPL-2.0-or-later` / `GPL-3.0-or-later` | Active | **Wrap** | Mature, battle-tested filesystem format/check/resize tools |
| **`lvm2` / `mdadm` / OpenZFS** | C | `GPL-2.0` / `LGPL-2.1` / `CDDL-1.0` (ZFS) | Active | **Wrap** (ZFS as optional extension due to CDDL) | Storage stack discovery and recovery |

---

## Key Architectural Findings

### 1. Do Not Rewrite Mature Filesystem & Partition Resize Tools in Rust
Resizing NTFS (`ntfsresize`), ext4 (`resize2fs`), Btrfs (`btrfs filesystem resize`), XFS (`xfs_growfs`), FAT32 (`libparted`), or cloning allocated filesystem blocks (`partclone`) carries decades of edge-case bug fixes where a single bug causes irreversible data loss.

**SBM Policy:**
- **Implement in Rust (`sbm-core` / `sbm-common`):** Read-only disk/partition discovery, stable disk identity matching (GPT Disk GUID, Partition GUID, `/dev/disk/by-id`, WWN/NVMe EUI-64, serial + model), **Storage Operation Plan** validation, platform-critical partition guardrails (e.g., blocking modification of Apple `iSCPreboot`/`Recovery` or active ESP without explicit override), dry-run simulation, progress streaming, and audit logging.
- **Wrap mature C/C++ CLI utilities (via structured argv, NEVER shell interpolation):** `lsblk --json`, `blkid`, `sfdisk --json`, `sgdisk`, `partclone`, `e2fsprogs`, `ntfs-3g`/`ntfsprogs`, `dosfstools`, `btrfs-progs`, `cryptsetup`, `ddrescue`, `testdisk`, `wimlib-imagex`, and `gparted` (as an interactive graphical utility).

### 2. BitLocker (`bitlk`) Support via `cryptsetup 2.8.9`
- `cryptsetup` supports mapping Windows BitLocker and BitLocker To Go volumes directly in kernel `dm-crypt` via `cryptsetup open --type bitlk`, which is significantly faster and more reliable than FUSE-based `dislocker`.
- **Important Limitations:**
  - Read/write access works when unlocked with the **48-digit BitLocker recovery password** or BEK key file (for standard AES-CBC/XTS BitLocker volumes).
  - TPM-sealed auto-unlock is **not** available from Linux (PCRs differ in recovery OS); users must supply their recovery key.
  - Hardware-encrypted OPAL SED BitLocker or newer Windows 11/BitLocker metadata variants require explicit verification before write operations; **`sbm-core` MUST treat BitLocker partitions as read-only / protected by default**.

### 3. Rust Storage Crates Assessment
- **`gptman` (`3.x`, `MIT OR Apache-2.0`):** Clean pure-Rust GPT header/table parser, useful for read-only inspection in `sbm-uefi` or `sbm-lab` disk fixture builders.
- **`devicemapper` / `libcryptsetup-rs` / `libblkid-rs`:** Available, though for `sbm-core` invoking `lsblk --json`, `sfdisk --json`, and `cryptsetup` as isolated subprocesses simplifies licensing separation (keeping SBM Rust crates `MIT OR Apache-2.0` cleanly isolated from GPL-2.0/3.0 storage tools) and crash containment!

### 4. GParted Wayland/XWayland Reality (Verified Nov 2026)
- As of **GParted `1.8.1`** (March 2026), GParted remains a **GTK3 (`gtkmm3`)** application that runs its entire GUI process as `root` (`gpartedbin`).
- Because Wayland compositors disallow arbitrary root GUI clients without explicit `xhost` / XWayland socket authorization, **`sbm-core`'s Wayland compositor MUST ship XWayland** (e.g., `labwc` or `cage` with XWayland enabled) and provide a controlled launcher wrapper for GParted.
