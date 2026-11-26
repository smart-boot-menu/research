# Minimal Linux Management Environment (`sbm-core`) Research

**Last verified:** 2026-11-26  
**Scope:** Selecting the Linux base, image builder, root filesystem format, update mechanism, and platform kernel strategy for `sbm-core` across **x86-64 PCs**, **generic ARM64 UEFI**, and **Apple Silicon** (keeping total footprint well under the 4 GB ceiling — targeting **< 1 GB**).

---

## 1. Image Builder & Base OS Comparison (November 2026)

| Tool / Base | Latest Verified Version (Nov 2026) | Build Speed | Hardware / Driver Breadth | GParted + GTK3 + Wayland | Apple Silicon (`linux-asahi`) | Immutable UKI + `dm-verity` + `sysext` | Verdict |
|---|---|---|---|---|---|---|---|
| **`mkosi` + Fedora Minimal** (x86-64/ARM64) & **Fedora Asahi Minimal** (Apple Silicon) | `mkosi v27.1` (Sep 28, 2026) | **Fast (2–5 min)** | **Excellent** (fresh mainline kernels, Mesa, `linux-firmware`, PipeWire/Wayland) | **First-class** (glibc, native RPMs, `labwc`, XWayland, `gparted 1.8.1`) | **First-class!** Official Asahi flagship (`kernel-16k`, `mesa-asahi`, `m1n1`, `uboot-asahi`) | **Native first-class** (UKI, EROFS/SquashFS, `dm-verity`, `systemd-sysext`, `systemd-sysupdate`) | **Recommended Primary Stack (ADR-003)** |
| **`mkosi` + Debian Minimal** (Trixie/testing) | `mkosi v27.1` / `mmdebstrap` | Fast | Very good (slightly older kernels unless backports) | First-class | Community/secondary (no official Asahi maintainer parity) | First-class | **Strong Runner-Up** (great for x86/ARM64; weaker for Apple Silicon) |
| **Alpine Linux (`apkovl` / `mkimage`)** | Alpine `3.22`/`3.23` (musl + BusyBox + OpenRC) | Very fast | Good (smaller size, ~150–300 MB) | Harder (musl libc incompatibilities with some tools, no `systemd-sysext`/`repart`/`sysupdate`) | Experimental / manual kernel packaging | Manual custom initramfs | **Study / Optional Ultra-Light Profile** (only if a < 200 MB sub-512 MB RAM profile is ever required) |
| **Buildroot** | `2026.08.2` (Oct 2026; `2026.11-rc2`) | Slow (compiles entire world from source: 45–90 min) | High maintenance burden for desktop GUI/Mesa/GParted | Painful (gtkmm3 + Mesa + Wayland + XWayland from source) | High maintenance burden | Manual | **Avoid** (wrong tool for desktop-recovery + broad PC driver coverage) |
| **Yocto / OpenEmbedded** | Active LTS/2026 releases | Very slow | High maintenance burden | Heavy | High maintenance burden | Good (via RAUC/Mender) | **Avoid** (overkill for open-source contributor velocity) |
| **NixOS** | `26.05` / `26.11 "Zokor"` (Nov 2026); Nix `2.36.0` | Fast with binary cache | Excellent | Good | Good (`nixos-apple-silicon`) | Good, but steep Nix DSL barrier & larger closure sizes | **Study** (strong reproducibility reference, steeper contributor barrier) |

---

## 2. Direct Architectural Answer: What Core OS for Apple Silicon vs x86/ARM64 PCs?

> **User Question:** *"The core OS on Apple Silicon would have to be Asahi, maybe a super light version of it that fits under 4 GB — and what would we use for x86 PCs?"*

### Answer: ONE Unified `mkosi` Build Pipeline, TWO Kernel/Driver Profiles — Both Under 1 GB!

Rather than maintaining two completely unrelated Linux distributions (e.g., Alpine on x86 and Fedora Asahi on Apple Silicon, which would double packaging, init scripts, systemd units, and QA work):

1. **x86-64 PCs & Generic ARM64 UEFI (`sbm-core-x86_64` / `sbm-core-aarch64`):**
   - **Base:** **Fedora Minimal** built declaratively via **`mkosi v27.1`** into an **immutable EROFS (zstd) image + Unified Kernel Image (UKI)**.
   - **Why Fedora Minimal for x86/ARM64 PCs:**
     - Ships **very recent upstream Linux kernels** (crucial for new Wi-Fi 6E/7 chips, NVMe controllers, and Intel/AMD GPUs on modern PCs *and* broad driver support for older HP laptops).
     - Shares **100% identical package names, systemd version, glibc, Wayland (`labwc` + XWayland), GParted `1.8.1`, `cryptsetup 2.8.9`, `partclone`, `wimlib`, and Rust binary ABI** with the Apple Silicon build!
     - **Estimated Compressed Size:** **~420–650 MB** (far below 4 GB; fits easily in RAM on 4 GB+ laptops, and can mount directly from disk/USB on 2 GB older laptops).
2. **Apple Silicon (`sbm-core-aarch64-apple`):**
   - **Base:** The **exact same `mkosi` image definition**, swapping ONLY the kernel/platform layer to **Fedora Asahi Remix repositories** (`kernel-16k` `linux-asahi`, `asahi-scripts`, `mesa-asahi`, plus `m1n1` + `uboot-asahi` bootstrap artifacts).
   - **Why This Solves Apple Silicon Cleanly:**
     - Fedora Asahi Remix is the **official flagship distribution of the Asahi Linux project** — meaning 16 KiB pages, M1/M2/M3 drivers, and upcoming M4/M5 `linux-asahi` updates arrive automatically without SBM maintaining a custom kernel fork.
     - Stripped of full KDE/GNOME desktops, browsers, and office apps, this **super-light Asahi `sbm-core` weighs ~500–850 MB compressed** (less than **1 GB**, easily fitting well under a 4 GB budget!).
     - Hardware-specific Wi-Fi/BT/SoC firmware blobs are **never redistributed** by SBM (respecting Apple copyright); they are lawfully extracted from the machine's own macOS partition during installation via `asahi-fwextract`.

---

## 3. Storage Format, Init System, Firmware & Update Architecture

### EROFS vs SquashFS (2025–2026 Kernel Reality)
- **EROFS (`zstd` / `lz4hc`)** optimizes for fixed **compressed output blocks** aligned to 4 KiB/16 KiB page boundaries -> dramatically lower read amplification, faster cold-start random reads, and lower page-cache RAM consumption than SquashFS.
- **SquashFS (`zstd`)** achieves slightly tighter absolute file size (~5–10% smaller) via large uncompressed input blocks (128 KiB–1 MiB), at the cost of higher memory overhead during random reads.
- **Recommendation:** Use **EROFS with `zstd` compression** as the primary `sbm-core` rootfs format (with `dm-verity` Merkle hash tree verified by the UKI initrd before switching root). Keep SquashFS as an optional compatibility fallback if older kernels are ever targeted.

### BusyBox vs `systemd`
- Although BusyBox + OpenRC shaves ~30–50 MB off an image, **`systemd` (v260–v262 era)** delivers decisive capabilities that directly map to SBM's architecture:
  - **Unified Kernel Images (`systemd-stub`)** + automatic TPM2 PCR measurement,
  - **Boot Loader Interface** (seamless handoff with `sbm-uefi`),
  - **`systemd-sysext`** (cryptographically verified modular recovery extensions mounted cleanly over an immutable rootfs — e.g., `gparted.raw`, `zfs.raw`, `remote-desktop.raw`),
  - **`systemd-repart`** & **`systemd-sysupdate`** (atomic A/B updates with automatic rollback on failed boot),
  - **`systemd-networkd`** + `iwd` (fast, clean wired + Wi-Fi bring-up without NetworkManager bloat).

### Managing `linux-firmware` Bloat (1 GB+ Uncompressed in 2026)
- By late 2026, full `linux-firmware` exceeds **1 GB uncompressed** (~5,000 files, dominated by huge NVIDIA GSP, AMD GPU, and Wi-Fi 7 blobs).
- **SBM Mitigation:**
  1. Use **zstd-compressed firmware blobs (`.zst`)** natively supported by the Linux kernel (cuts size by **50–67%**).
  2. Ship a **curated core firmware subset** in the base image (Intel/AMD/Realtek/Broadcom/Qualcomm Wi-Fi + Ethernet + NVMe/RAID + Intel/AMD display firmware) — keeping firmware around **~120–180 MB compressed** instead of 500+ MB.
  3. Ship heavy optional firmware (e.g., large NVIDIA GSP blobs or rare enterprise HBAs) in an optional **`firmware-extra.sysext`** extension.

### RAM-Resident Operation & Low-Memory Fallback
- When system RAM ≥ **4 GiB**: `sbm-core` initramfs supports `sbm.toram=auto` — copying the ~500 MB EROFS image into `tmpfs`/`zram` so the boot USB or internal disk can be completely unmounted, partitioned, cloned, or erased safely!
- When system RAM < **4 GiB** (e.g., an older 2 GiB x86 HP laptop): automatically mounts EROFS directly from the boot medium with a `zram`-backed overlayfs upper layer, staying comfortably within **~350–500 MB RAM** while running `labwc` + GParted.

### A/B Atomic Updates & Independent Recovery
- Internal installations maintain **two `sbm-core` slots (`Slot A` and `Slot B`)** as signed UKIs + `dm-verity`-protected EROFS images, managed via **`systemd-sysupdate`** (or SBM's TUF-verified updater) with `sbm-uefi` boot-attempt counting: if a newly updated `sbm-core` fails to confirm boot success, `sbm-uefi` automatically falls back to the previous known-good slot!
