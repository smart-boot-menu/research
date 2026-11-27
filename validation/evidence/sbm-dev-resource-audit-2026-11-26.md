# SBM Developer Resource Audit & QEMU HVF Measurements (2026-11-26)

**Date:** 2026-11-26  
**Host:** Fanless Apple Silicon M5 MacBook Air (`arm64`, 10 CPUs, 16,384 MiB unified memory, 460 GiB APFS SSD, macOS Darwin 27.0.0)  
**Companion landscape research:** [Development Infrastructure & Testing Landscape (Build vs. Reuse Audit)](../../ecosystem/dev-infrastructure-and-testing-landscape-2026-11-26.md)

All figures below are **direct measurements** captured on the development workstation on 2026-11-26. No estimates are mixed with measured values.

---

## 1. Host Baseline & Toolchain Inventory

- **Memory & pressure (idle baseline):** 16,384 MiB unified RAM, `memory_pressure` 63–64% free (`normal`), swap `683–691 / 2048 MiB`, 1/5/15m load `2.36 2.15 2.29`.
- **Disk (`/`):** 460 GiB APFS volume, **154 GiB available** at audit start; APFS `cp -c` (`clonefile(2)`) verified functional (`rc=0`).
- **Installed tools:** `rustc 1.100.0`, `cargo 1.100.0` (2026-11-05), Homebrew `qemu-system-{aarch64,x86_64}` 11.1.3 (`virtio-balloon-pci` + `virtio-9p-pci` compiled in), `qemu-img`, `swtpm 0.10.2`, `zstd`, `xz`. Rust targets installed: `aarch64-apple-darwin`, `aarch64-unknown-uefi`, `x86_64-unknown-uefi`, `wasm32-unknown-unknown`.
- **Not installed locally:** `lima` (`limactl`), `vfkit`, `krunkit`, `podman`, `docker`, `colima`, `sccache`, `cargo-nextest`, `mkosi` (runs inside the Fedora 45 builder VM at `v27.1`), `just`. `/System/Library/Frameworks/Virtualization.framework` is present.

---

## 2. Phase 1 Workspace Storage Audit (Before Optimization)

Total workspace footprint across `~/gh/smart-boot-menu` before cleanup: **54.53 GiB allocated / 65.72 GiB apparent** (178,350 files).

| Repository / Directory | Allocated (GiB) | Apparent (GiB) | Primary Consumers |
|---|---:|---:|---|
| `sbm-core` | 24.15 | — | `build/` 22.62 GiB (`builder/disk.qcow2` 14.55 GiB, Fedora base qcow2 0.57 GiB, `images/` 7.20 GiB incl. two `-software.raw` 2.56 GiB, `diag/` 0.28 GiB, `evidence/` 0.06 GiB); `target/` 1.53 GiB |
| `install` | 16.02 | 27.52 | `build/` 12.86 GiB (`repro/{a,b}` 6.70 GiB, `out/release` 3.30 GiB, `inputs-0.0.1` 2.60 GiB, `fixtures` 0.20 GiB); `target/` 3.16 GiB alloc / **12.10 GiB apparent** (due to leaked `OnceLock<TempDir>` sparse test media) |
| `sbm-server` | 4.95 | — | `target/` 4.87 GiB; `web/` 80.7 MiB |
| `sbm-catalog` | 4.57 | — | `build/cache/` 3.53 GiB downloaded OS ISOs; `target/` 1.04 GiB |
| `sbm-lab` | 2.54 | — | `var/` 1.66 GiB (`runs/` 1.00 GiB [5 `int-*` runs 762 MiB, 45 older runs 251 MiB], `src/` 458 MiB, `fixtures/` 199 MiB); `target/` 791 MiB |
| `sbm-uefi` | 0.94 | — | `build/` 497 MiB; `target/` 470 MiB |
| `sbm-as` | 0.73 | — | `build/` 525 MiB; `target/` 222 MiB |
| `sbm-common` | 0.41 | — | `target/` 423 MiB |
| `.integration-worktrees` | 0.23 | — | 4 clean, fully-merged worktrees (`docs`, `research`, `sbm`, `sbm-common`, 0 unmerged commits vs `main`) |

- **Total Cargo `target/` across all repos:** **15.14 GiB** allocated (independent per-repo `target/` directories recompiling shared crates such as `serde`, `clap`, `fatfs`, `sha2`, `tokio`).
- **Home caches outside workspace:** `~/.cargo` 1.4 GiB, `~/.rustup` 4.9 GiB, `~/.cache` 2.2 GiB, `~/Library/Caches` 8.3 GiB.

### Exact SHA-256 Large-File Duplicates (>= 10 MiB, Distinct Inodes): **10.05 GiB Wasted**

1. `install/build/{out/release,repro/a,repro/b}/sbm-0.0.1-uefi-dualarch.raw` — 3 identical copies (`sha256: 7be07b851f1e...`), **3.42 GiB wasted**.
2. `install/build/{out/release,repro/a,repro/b}/sbm-0.0.1-uefi-dualarch.raw.zst` — 3 identical copies (`sha256: e18a41b08ac7...`), **3.27 GiB wasted**.
3. `install/build/inputs-0.0.1/sbm-core-{arm64,x86-64}.raw` exact copies of `sbm-core/build/images/sbm-core-{arm64,x86-64}.raw` (`dd85499142ed...`, `fbfd9e5b9ad3...`), **2.56 GiB wasted**.
4. Remaining ~0.80 GiB across duplicate 64 MiB `vars.fd` copies, duplicate 49 MiB fixture `.raw` images (`install/build/fixtures` vs `sbm-lab/var/fixtures`, 196 MiB), duplicate `u-boot-v2026.10.tar.gz` (44.5 MiB), and identical `.rlib`s across Cargo `target/` directories.

---

## 3. Measured Build & Validation Latencies

### 3.1 `sbm-core` Full Build Log Breakdown (2026-11-26 21:43:42–21:54:41)

- **Total wall time:** **659 s** (builder VM uptime `646.66 s` per `serial.log`).
- **`zstd -19 -T1` compression:** **260 s (39% of build)** — `arm64`: 121 s (`21:46:46` -> `21:48:47`), `x86-64`: 139 s (`21:52:22` -> `21:54:41`). `install` reads only `.raw`, never `.raw.zst`.
- **Cross-arch `x86-64` `mkosi` (under `qemu-user-static-x86`):** **215 s (33% of build)**.
- **Warm incremental native ARM64 Cargo build (`cargo build --release -p sbm-cored` inside builder):** **5.09 s** (5.42 s total over SSH) — **121× faster** than a full image rebuild.

### 3.2 Level 0 Native Host Unit Tests & Preview Timings (M5 macOS)

| Repository | Command | Result | Wall Time |
|---|---|---|---:|
| `sbm-as` | `cargo test --offline` | 17 passed | **0.13 s** |
| `sbm-uefi` | `cargo test --offline` (default-members) | 31 passed | **0.14 s** |
| `sbm-uefi` | `cargo run -p xtask -- preview` (1280×800 PNG, mock Level 0) | 1 PNG | **0.17 s** |
| `sbm-core` | `cargo test --offline --workspace` (incl. `sbm-ui`) | 22 passed | **0.41 s** |
| `sbm-catalog` | `cargo test --offline` | 22 passed | **0.69 s** |
| `sbm-common` | `cargo test --offline` | 8 passed | **1.54 s** |
| `sbm-server` | `cargo test --offline` | 19 passed | **6.05 s** |
| `install` (`sbm-plan`) | `cargo test --offline -p sbm-plan` | 9 passed | **0.00 s** (0.69 s compile) |
| `install` (full workspace) | `cargo test --offline --workspace` | 61 passed | **83.74 s** (dominated by repeated 512 MiB FAT32 ESP hashing in `sbm-media`/`sbm-exec`/`sbm-release` e2e tests) |

### 3.3 Full-System Graphical Integration (`integrated-0.0.1-v2`, 9/9 Gates Passed)

- **`aarch64` (HVF):** **61.3 s** (`uefi_menu_1: 6.8 s`, `core_ui_ready: 14.8 s`, `gparted: 18.1 s`).
- **`x86_64` (TCG):** **159.1 s** (`uefi_menu_1: 18.8 s`, `core_ui_ready: 74.4 s`, `gparted: 80.9 s`) — **2.6× slower** wall time and substantially higher CPU/thermal load.

---

## 4. Phase 6 QEMU HVF Memory & Disk Discard Benchmarks

Two sequential controlled runs of the 4,096 MiB / 2 vCPU Fedora 45 ARM64 builder VM (`sbm-core/build/builder/disk.qcow2`) were executed under `sbm-lab resources` admission using incompressible `/dev/urandom` guest allocations (`512 MiB`) to prevent macOS memory-compressor false positives.

| Metric | Run A: Baseline (No Balloon, No Discard) | Run B: `virtio-balloon-pci` (`free-page-reporting=on`) + `discard=unmap,detect-zeroes=unmap` |
|---|---|---|
| Boot to SSH | 16.30 s | 13.23 s |
| Host `phys_footprint` (idle after boot) | 1,238 MB (`Footprint`: 357 MB) | 1,215 MB (`Footprint`: 373 MB) |
| Host `phys_footprint` (after 512 MiB `/dev/urandom` alloc) | 2,593 MB (`Footprint`: 662 MB; included cargo build) | 1,678 MB (`Footprint`: 394 MB) |
| Host `phys_footprint` (after `rm` + `drop_caches` + 5 s) | **2,616 MB** (+23 MB, **0 reclaimed**) | **1,686 MB** (+8 MB, **0 reclaimed**) |
| Guest `free` RAM increase after `rm` + `drop_caches` | +1,049 MiB | +568 MiB |
| Host `memory_pressure` free % | 64% -> 57% -> 56% | 58% -> 57% -> 57% |
| `disk.qcow2` host allocated before/after `sudo fstrim -av` | 14.553 GiB (no discard) | **14.553 GiB -> 10.922 GiB (-3.631 GiB / -3,717.7 MiB)** |
| Clean SSH shutdown time | 1.16 s | Clean (`rc=0`) |

### Root-Cause Diagnosis

1. **Why QEMU HVF `phys_footprint` never shrinks on macOS (even with `virtio-balloon-pci,free-page-reporting=on`):** On Linux, QEMU handles `free-page-reporting` with `madvise(..., MADV_DONTNEED)`, which immediately frees anonymous pages. On Darwin/macOS, `MADV_DONTNEED` does not release physical pages, and guest RAM remains mapped in Hypervisor.framework (`hv_vm_map`). Combined with macOS host page-cache double buffering of large QCOW2 I/O during `mkosi` builds, an 8 GiB builder easily drives host memory pressure toward ~15 GiB.
2. **Why `disk.qcow2` grew to 14.55 GiB despite 12 GiB guest `df`:** `start-builder.sh` omitted `discard=unmap,detect-zeroes=unmap`, preventing Btrfs `discard` / `fstrim` from punching holes in the APFS-backed QCOW2 file. Enabling those flags and running `fstrim -av` reclaimed **3.63 GiB** live in place.
3. **Inside `disk.qcow2` (remaining 10.92 GiB):** `5.9 GiB` in `/home/builder/src/sbm-core/build/images` (copies already pulled to host, incl. `1.3 GiB` stale `sbm-core-arm64-software.raw`), `2.4 GiB` in `target/debug`, `1.4 GiB` in `build/mkosi-pkgcache` (reusable), `923 MiB` in `target/aarch64-unknown-linux-gnu` (reusable), `838 MiB` in `target/x86_64-unknown-linux-gnu`, `688 MiB` `.rustup`, `247 MiB` `.cargo`.

---

## 5. Straightforward Optimizations Implemented & Measured Impact

The following low-risk, non-architectural fixes were committed and pushed without introducing new infrastructure subsystems or modifying Agent 13's active working trees:

1. **`sbm-uefi` (`e397e34`):** Default `xtask build`/`qemu` to native host arch (avoiding redundant 3× builds and unnecessary cross-target work), and added zero-dependency Level 0 `sbm-gfx` host preview (`xtask preview`, 166.9 ms PNG, explicitly labeled non-qualification mock).
2. **`install` (`af37d98`):**
   - Fixed `scripts/check-reproducible.sh` to hash run A and prune run A's large `.raw`/`.raw.zst` before run B, then retain `repro-report.json` + sidecars/manifests while removing duplicate large images unless `--keep` is passed (halving peak disk usage and reducing retained disk usage from **6.70 GiB -> < 100 KiB** with identical byte-for-byte verification).
   - Added `--no-compress` / `SBM_COMPRESS=0` support to `scripts/build-media.sh` (still runs `media check-lock` and `media verify`).
   - Fixed `OnceLock<TempDir>` leak in `crates/sbm-exec/tests/e2e.rs` by reusing a single deterministic `shared-media` directory.
3. **`sbm-core` (`098921f`):**
   - Fixed `scripts/build-inside.sh` to compile and stage **only** the Rust target(s) in `SBM_ARCHES`, default `dev`/`integration` to native arch + `mkosi --incremental=yes` + `SBM_COMPRESS=0`, reserve dual-arch + `Incremental=no` + `zstd -19` for `SBM_BUILD_PROFILE=release`, and print stage timings.
   - Updated `scripts/finalize-image.py` to honor `SBM_COMPRESS=0` and reuse `.raw.zst` if `.raw` SHA-256 is unchanged.
   - Updated `scripts/sync-and-build.sh` to forward build profile/arch/compression env vars.
   - Updated `scripts/start-builder.sh` with `discard=unmap,detect-zeroes=unmap`, `virtio-balloon-pci,free-page-reporting=on,deflate-on-oom=on`, and `trim` subcommand.
   - Updated `scripts/boot-smoke.sh` to boot via a disposable QCOW2 overlay instead of attaching raw images RW directly.

### Measured Before vs. After Physical SSD Impact (Zero Data Loss)

| Action | Physical SSD Reclaimed | Verification |
|---|---:|---|
| `disk.qcow2` `discard=unmap` + `sudo fstrim -av` | **3.631 GiB** | `14.553 GiB -> 10.922 GiB` (`os.stat().st_blocks`) |
| Pruned `install/build/repro/{a,b}` large `.raw`/`.raw.zst` | **6.691 GiB** | Verified SHA-256 matched `install/build/out/release/*` first; sidecars/manifests retained |
| APFS `cp -c` CoW dedup of `install/build/inputs-0.0.1/sbm-core-{arm64,x86-64}.raw` | **2.560 GiB** | Verified SHA-256 identical before & after; both independent paths preserved |
| Removed 7 leaked `install/target/tmp/sbm-exec-it/shared-media-*` dirs | **0.104 GiB** (10.8 GiB apparent) | Fixed root leak in `af37d98` |
| Removed 4 clean, fully-merged `.integration-worktrees/*` | **0.250 GiB** | Verified empty `git status` and 0 unmerged commits vs `main` |
| **Total safely reclaimed** | **13.24 GiB** | Host `/` available rose from **154 GiB -> 166.80 GiB** (`df -h /`) |

### Additional Reclaimable Candidates (Deferred Pending Review)

- `sbm-core` builder guest `/home/builder/src/sbm-core/build/images` (**5.9 GiB** duplicate of host outputs) + `target/debug` (**2.4 GiB**) -> up to **~8.3 GiB** reclaimable inside `disk.qcow2` via `fstrim` (or **10.9 GiB** if replaced by Lima + `virtiofs`).
- `sbm-core/build/images/sbm-core-{arm64,x86-64}-software.raw` (**2.56 GiB**) — confirm unused by Agent 13 before pruning.
- `sbm-lab/var/runs/*` excluding Agent 13's 5 `int-*` dirs (**251 MiB**).
- Per-repo Cargo `target/` consolidation via Cargo 1.100 `build.build-dir` + `sccache` (currently **15.14 GiB** across 8 repos).

---

## 8. Track B Implementation Benchmarks (2026-11-26)

All measurements below were captured on the same 16 GiB Apple M5 MacBook Air (`aarch64`, 10 CPUs, macOS Darwin 27.0.0, `rustc`/`cargo` `1.100.0`) after user approval of the Track B roadmap.

### 8.1 Cargo Build-Dir Isolation, `sccache`, `cargo-nextest`, and `cargo-zigbuild`

1. **Cargo `1.100.0` `build.build-dir` isolation:**
   - Verified `CARGO_BUILD_BUILD_DIR="{cargo-cache-home}/build/{workspace-path-hash}"` on `sbm-common` (`~/.cargo/build/<hash>/...`). Eliminates per-repo `target/` tree clutter without the cross-worktree lock contention of an unhashed shared `CARGO_TARGET_DIR` (`cargo#17312`).
2. **`sccache 0.20.1` (`SCCACHE_ALLOW_INCREMENTAL=1`, scratch `CARGO_TARGET_DIR`, `cargo test --offline --no-run`):**

   | Run | Repo | Wall (s) | Cache Hit Rate | Non-Cacheable Calls | Cache Size |
   |---|---|---:|---:|---:|---:|
   | `server-plain` | `sbm-server` (422 crates) | `25.85` | — | — | — |
   | `server-sc-cold` | `sbm-server` (cold) | `32.89` (`+27%`) | `0.00%` (752 miss) | 76 (35 bin, 20 proc-macro, 11 incr) | 235 MiB |
   | `server-sc-warm` | `sbm-server` (fresh target) | `19.64` (`-24%`) | `85.51%` (643 hit / 109 miss) | 76 | 328 MiB |
   | `core-plain` | `sbm-core` (392 crates) | `14.86` | — | — | — |
   | `core-sc-xrepo` | `sbm-core` (after `sbm-server`) | `19.15` (`+29%`) | `13.84%` (22 hit / 137 miss) | 46 (22 bin, 9 proc-macro, 10 incr) | 519 MiB |

   - **Verdict:** Do **not** enable `sccache` globally on single-worktree macOS laptops (27–29% slower on cold/cross-repo builds, +519 MiB SSD). Recommend opt-in only for CI or repeated fresh worktrees of `sbm-server`.

3. **`cargo-nextest 0.9.152` vs `cargo test`:**
   - Fixed a cross-process race on `install/crates/sbm-exec/tests/e2e.rs` `shared_media()` (`install` commit `b11c427`).
   - `install` (61 tests across 3 heavy integration binaries):
     - Baseline `cargo test`: **`85.05 s`** wall (`231.5 s` user, `56.4 s` sys).
     - Warm `cargo test`: **`80.43 s`** wall (`241.3 s` user, `55.6 s` sys).
     - Warm `cargo nextest run`: **`46.33 s`** wall (`262.0 s` user, `24.8 s` sys) — **1.84× faster** (`-45.5%` wall time, `-56%` sys CPU).
   - `sbm-lab` (52 tests): `3.62 s` (`cargo test`) -> **`3.26 s`** (`cargo nextest`, `-10%`).
   - **Verdict:** Adopted in `sbm/justfile` (`just test install` uses `cargo nextest run` when installed).

4. **`cargo-zigbuild 0.24.1` (`zig 0.17.1`, `sbm-cored`, target `aarch64-unknown-linux-gnu` on macOS):**
   - Cold build: **`9.14 s`** wall (`262 MiB` target dir, `0 MiB` VM RAM).
   - Warm incremental rebuild (touch `sbm-cored.rs`): **`0.50 s`** wall (`0.18 s` Cargo compile, `0 MiB` VM RAM) — valid `ELF 64-bit LSB pie executable, ARM aarch64`.
   - **Verdict:** Adopted in `sbm/justfile` (`just core-zigbuild`) for fast zero-VM iteration on pure-Rust Linux binaries (`sbm-cored`, `sbm-runtime`).

### 8.2 Lima `2.3.0` + `krunkit 1.3.2` (`libkrun 1.19.8`) vs QEMU HVF Builder

Both builders ran sequentially under `sbm-lab resources` admission with identical guest specs: **Fedora 45 `aarch64` (`7.2.11-300.fc45.aarch64`), `4096 MiB` RAM, `2` vCPUs**. Allocation test: `dd if=/dev/urandom of=/tmp/fill.bin bs=1M count=1024 && sync`, followed by `rm -f /tmp/fill.bin && sync && echo 3 > /proc/sys/vm/drop_caches`.

| Metric | QEMU HVF (`start-builder.sh`) | Lima `2.3.0` + `krunkit 1.3.2` (`libkrun 1.19.8`) |
|---|---:|---:|
| Cold creation + first boot | — (existing disk) | `23.11 s` (from local Fedora 45 qcow2) |
| Warm boot-to-SSH | `16.32 s` | **`8.83 s`** (`1.85×` faster) |
| Clean stop | `1.14 s` | **`0.90 s`** |
| Host process mem / RSS (idle) | `1242 MiB` / `1257 MiB` | **`1151 MiB`** / `1159 MiB` |
| Host process mem / RSS (`alloc_1024m`) | `2140 MiB` / `2155 MiB` | `2010 MiB` / `2006 MiB` |
| Host process mem / RSS (`after_free_16s`) | `2150 MiB` / `2166 MiB` (**0 MiB reclaimed**) | `2018 MiB` / `2013 MiB` (**0 MiB reclaimed**) |
| Post-`dnf`+`cargo` mem (`post_cargo` -> `post_clean_6s`) | — | `3782 MiB` -> `3756 MiB` (`free_pct`: `54%` -> `55%`, `comp`: `4721` -> `4625 MiB`) |
| Source sync (noop) | `0.20 s` (`rsync`) | **`0.00 s`** (`virtiofs`) |
| Warm incremental `cargo build -p sbm-cored` | `0.31 s` (local disk) | `0.31 s` (`virtiofs` src, local target) |
| Guest `fstrim` & `unshare -r` (`mkosi` prereq) | Both work (`-3.63 GiB` trimmed) | Both work (`22.6 GiB` / `2.4 GiB` trimmed; `1.3 GiB` allocated) |
| `sbm-lab resources` integration | Native in `start-builder.sh` | Requires external wrapper |
| Read-only host mount enforcement | N/A (`rsync`) | Not enforced by `krunkit` `virtiofs` |

**Root Cause & Migration Decision:** Although Homebrew ships `libkrun 1.19.8` (which implements `MADV_FREE` + 2 MiB Replace-mode reclaim) and Fedora loads `virtio_balloon`, inspecting the live `krunkit` command line confirmed that **`lima 2.3.0` + `krunkit 1.3.2` do not attach a `virtio-balloon` device yet** (tracked upstream in `krunkit#121`). Consequently, `krunkit` reclaimed effectively **0 MiB** of freed guest RAM on macOS, matching QEMU HVF. Per the migration criterion (*"Only migrate if it demonstrates meaningful improvements"*), **full builder migration is deferred until `krunkit#121` wires balloon reclaim**. The hardened QEMU HVF builder (`start-builder.sh`) remains the default for `mkosi` builds, while host `cargo-zigbuild` handles zero-VM Rust daemon iteration and QEMU remains the firmware test backend.

### 8.3 `sbm-lab` GPT/FAT Consolidation & `os-autoinst` Linux CI Prototype

- **GPT/FAT consolidation (`sbm-lab` commit `0794b0c`):** Replaced `sbm-lab`'s duplicated `gpt.rs` and `fatimg.rs` with thin adapters delegating to `install/crates/sbm-media` (`b11c427`), removing direct `crc32fast` and `fatfs` dependencies (**net −269 LOC**) and eliminating `~1.5 GiB` of peak heap copying during `integration.rs` `inspect_overlay_esp`. Verified green in hosted Linux CI (`https://github.com/smart-boot-menu/sbm-lab/actions/runs/45946390386`, `1m 45s`).
- **`os-autoinst` Linux KVM CI prototype (`sbm-lab` commits `142a0bb`, `c3448d9`):** Added `prototypes/os-autoinst/` and `.github/workflows/os-autoinst-prototype.yml` running `registry.opensuse.org/devel/openqa/containers/isotovideo:qemu-kvm` on free `ubuntu-latest` KVM runners against the tiny `efi-probe` UEFI binary (`NICTYPE=user`, `NICTYPE_USER_OPTIONS=restrict=on`, offline). Verified green end-to-end (`https://github.com/smart-boot-menu/sbm-lab/actions/runs/45946390417`, **`1m 14s`**). Existing `sbm-lab` test tiers remain preserved.
