# Development Infrastructure, Testing & Tooling Landscape (Build vs. Reuse Audit)

**Date:** 2026-11-26  
**Scope:** Open-source alternatives for UEFI/bootloader testing, OS/GUI integration testing, fast incremental Linux development, Apple Silicon virtualization backends, Rust build/cache optimization, and an honest build-versus-reuse audit of `sbm-lab` (9,805 LOC).  
**Companion measurements:** [SBM Developer Resource Audit (2026-11-26)](../validation/evidence/sbm-dev-resource-audit-2026-11-26.md).

---

## 1. Decision Hierarchy

Smart Boot Menu exists to deliver a reliable, fast, secure boot manager and recovery environment — **not** to build a bespoke virtualization orchestrator, image manager, or build system. Every infrastructure capability is evaluated in strict preference order:

1. **Reuse** an existing maintained open-source implementation directly.
2. **Configure and compose** existing tools declaratively (config files, profiles, `just`/`mkosi`/`lima.yaml`).
3. **Write minimal integration adapters** only where no tool bridges an SBM-specific gap.
4. **Contribute improvements upstream** when a good tool is missing a small feature.
5. **Fork** only when upstream is incompatible or stalled and the delta is justified.
6. **Build custom** only when we can make it meaningfully better than existing tools for a genuine SBM requirement.

Rust-first applies to SBM's shipped product components (`sbm-uefi`, `sbm-core`, `sbm-server`, `install`), **not** to rewriting mature Python/Go/Perl/shell developer infrastructure in Rust. Deleting duplicated custom code is a net win.

---

## 2. Open-Source Landscape Survey

### 2.1 UEFI & Bootloader Host Execution / Testing

| Project | Primary Source | License | Host OS / Arch Support | Maintenance Status | Assessment for SBM |
|---|---|---|---|---|---|
| **EDK II `EmulatorPkg`** | [`tianocore/edk2`](https://github.com/tianocore/edk2/tree/master/EmulatorPkg) (`EmulatorPkg.dsc`, `build.sh`, `Readme.md`) | `BSD-2-Clause-Patent` | `IA32` / `X64` only (Windows, Linux, macOS x86_64 via X11/XQuartz). **No native ARM64 host support.** An Oct 2021 AArch64 Linux RFC (`edk2-devel`) was never merged. | Active upstream, x86-only for `EmulatorPkg` | **Do not adopt.** Cannot run natively on Apple Silicon M5 (`arm64`) without Rosetta/x86 containerization. Upstream TianoCore explicitly recommends `ArmVirtPkg` + `qemu-system-aarch64` for ARM64 UEFI testing. |
| **Intel `EfiWrapper`** | [`intel/efiwrapper`](https://github.com/intel/efiwrapper) | `BSD-2-Clause` | Linux host (`efiwrapper_host`), Android Kernelflinger subset, depends on `gnu-efi` + Coreboot `libpayload`. | **Archived / unmaintained** — Intel ceased all development, fixes, and PRs. | **Reject.** Unmaintained, tied to Kernelflinger/Coreboot `libpayload`, lacks GOP/modern UEFI protocols needed by `sbm-uefi`. |
| **U-Boot `sandbox` (`CONFIG_EFI_LOADER`)** | [`u-boot/u-boot`](https://docs.u-boot.org/en/latest/arch/sandbox/sandbox.html), [UEFI on U-Boot](https://docs.u-boot.org/en/latest/develop/uefi/uefi.html) | `GPL-2.0-or-later` | **Linux host ELF only** (x86_64 and aarch64 Linux; NOT native macOS). Optional SDL2 graphics (`NO_SDL=1` headless). | Very active upstream (`v2026.10` already used by `sbm-as`) | **Adopt on Linux CI / Linux dev VM** for `sbm-as` / U-Boot UEFI compatibility! Loads standalone `.efi` binaries directly via `host load hostfs` + `bootefi` and runs `efi_selftest` + `test/py` (`test_efi_secboot`) without booting QEMU. Cannot replace native macOS preview or EDK II OVMF/AAVMF firmware testing. |
| **QEMU + OVMF / AAVMF (`ArmVirtPkg`)** | [`qemu/qemu`](https://www.qemu.org/), [`tianocore/edk2`](https://github.com/tianocore/edk2) | `GPL-2.0-only` / `BSD-2-Clause-Patent` | macOS Apple Silicon (**HVF** for `aarch64`, TCG for `x86_64`), Linux KVM/TCG. | Very active (QEMU 11.1.3 installed) | **Keep as authoritative firmware harness.** Real EDK II UEFI services, GOP framebuffer, NVRAM vars, `swtpm` TPM 2.0, SMBIOS, multi-disk GPT, and `x86_64` verification. |
| **Pure-Rust host crates (`sbm-boot`, `sbm-gfx`)** | `sbm-uefi` workspace (`default-members`) | `MIT OR Apache-2.0` | Native macOS, Linux, Windows (0.14 s for 31 tests; 167 ms for 1280×800 PNG preview). | Owned by SBM | **Keep (Level 0 fast loop).** Zero VM overhead for layout/theme/policy logic; strictly labeled non-qualification mock preview. |

### 2.2 Automated OS, Graphical & Hardware Testing Frameworks

| Project | Primary Source | License | Host OS / Arch Support | Maintenance Status | Assessment for SBM |
|---|---|---|---|---|---|
| **`os-autoinst` (`isotovideo`) & `openQA`** | [`os-autoinst/os-autoinst`](https://github.com/os-autoinst/os-autoinst), [`os-autoinst/openQA`](https://github.com/os-autoinst/openQA) | `GPL-2.0-or-later` | **Linux-native** (Perl + C++ OpenCV; standalone `isotovideo:qemu-kvm` Podman container on Linux CI; software-emulated `qemu-x86` container on non-KVM hosts, **no native macOS HVF**). | Industry standard (openSUSE, Fedora, Debian, AlmaLinux, Qubes OS, GNOME, KDE) | **Strong candidate to replace `sbm-lab`'s graphical integration harness in Linux CI.** Provides mature QMP input, screenshot/video recording, visual **needles** (match areas, exclude regions, OCR, click points), and serial assertions — far stronger than custom PPM pixel-variance checks (`ppm_has_variance` in `sbm-lab/src/proxy.rs`) and 924 LOC of ad-hoc QMP polling in `integration.rs`. **Constraint:** Not a native macOS HVF inner-loop tool; use in Linux CI (or Linux builder/CI runners) for Level 3/4 graphical & recovery qualification. |
| **`labgrid`** | [`labgrid-project/labgrid`](https://github.com/labgrid-project/labgrid), [docs](https://labgrid.readthedocs.io) | `LGPL-2.1-or-later` | **Linux-primary** (Python 3.10+, pytest plugin, gRPC coordinator since v25.0). Client library works on Darwin; hardware exporters (`udev`, raw net) require Linux. | Active (Pengutronix; v25.x) | **Adopt when testing physical hardware & Apple Silicon boards** (and optional for Linux CI QEMU). Models `Target`/`Resource`/`Driver`/`Strategy` state machines (`off -> U-Boot/UEFI -> Linux shell`) uniformly across `QEMUDriver` and physical power switches, serial consoles, SD-muxes, and USB loaders. Eliminates any need to build hardware-lab control into `sbm-lab`. |
| **`jc-lab/test-foundry`** | [`jc-lab/test-foundry`](https://github.com/jc-lab/test-foundry) | `GPL-2.0-only` | Linux and Windows hosts (Go); `swtpm` Linux-only. Focused on **Windows kernel/driver & UEFI guests**; Linux guest support noted as incomplete (April 2026). | Young / single-author | **Do not adopt.** Narrowly focused on Windows WinRM/BSOD driver testing with incomplete Linux guest support and no macOS HVF focus. |

### 2.3 Fast Incremental Linux Development & Immutable Root Overlays

| Project | Primary Source | License | Host OS / Arch Support | Maintenance Status | Assessment for SBM |
|---|---|---|---|---|---|
| **`mkosi`** | [`systemd/mkosi`](https://github.com/systemd/mkosi) (`mkosi.1.md`, `docs/sysext.md`) | `LGPL-2.1-or-later` | Linux host (`mkosi v27.1` is **already installed** in `sbm-core`'s Fedora builder). | Very active (systemd upstream) | **Reconfigure and reuse built-in features instead of writing custom scripts!** `mkosi` *already* implements: (1) `Incremental=yes` (caches base RPM tree before `--extra-tree`), (2) `Format=sysext` + `Overlay=yes` + `BaseTrees=` (native `systemd-sysext` builder), (3) `mkosi vm` + `RuntimeTrees=<host>:<guest>` (VirtioFS live mount; note `systemd/mkosi#4278`: mount under `/var/` not `/run/`), and (4) `Ephemeral=yes` (reflink snapshot on Btrfs/XFS). |
| **`systemd-sysext` & `systemd-confext`** | [`systemd-sysext(8)`](https://www.freedesktop.org/software/systemd/man/latest/systemd-sysext.html), [`systemd.io`](https://systemd.io/BUILDING_IMAGES/) | `LGPL-2.1-or-later` | Linux (`systemd >= 256` already in Fedora 45 `sbm-core`). | Very active (core systemd) | **Adopt for Level 2 incremental `sbm-core` dev!** Overlays dev binaries in `/usr/` onto an immutable `dm-verity` root via `overlayfs` without modifying or rebuilding the base image. Supports `--mutable=ephemeral` (v256+) for live tinkering that vanishes on unmerge, while production images keep unsigned extensions disabled and measure signed extensions into TPM2 PCR 13. |
| **`virtme-ng` (`vng`)** | [`arighi/virtme-ng`](https://github.com/arighi/virtme-ng) | `GPL-2.0-only` | **Linux host only** (exports host live rootfs over `virtiofsd` + `overlayfs` + `microvm` + Rust `virtme-ng-init`, boots in ~2 s). Cannot export a macOS rootfs natively. | Very active (`v1.42`) | **Useful inside Linux CI / Linux dev hosts** for fast kernel/initrd/systemd unit testing (~2 s boot, `--build-host` remote offload, `--root` external rootfs), **not** a native macOS tool. On Linux, `mkosi vm --ephemeral --runtime-tree` covers most `sbm-core` userspace needs without adding another dependency. |

### 2.4 macOS Apple Silicon Virtualization & Linux Builder Backends

This category yielded the most critical primary-source finding of the entire investigation — directly contradicting the common assumption that switching from QEMU HVF to Apple's `Virtualization.framework` (`vz` / `vfkit`) automatically solves macOS memory reclamation.

| Backend | Primary Source | License | Shared FS | macOS Memory Reclamation Status (Verified Nov 2026) | Assessment for SBM |
|---|---|---|---|---|---|
| **QEMU 11.1.3 (`accel=hvf`)** | [`qemu/qemu`](https://www.qemu.org/), Phase 6 benchmark on this M5 Mac | `GPL-2.0-only` | `virtio-9p` (slow) or SSH/`rsync` (current) | **Does NOT reclaim `phys_footprint` on macOS.** Measured on this host: after 512 MiB `/dev/urandom` allocation -> `rm` + `drop_caches` + 5 s with `virtio-balloon-pci,free-page-reporting=on`, `phys_footprint` stayed at **1,686 MB** (vs 1,678 MB peak) because Darwin `MADV_DONTNEED` does not release anonymous pages. Stopping the VM is the only way to release RAM (1.16 s stop / 13.23 s start). **Disk trim DOES work:** `discard=unmap,detect-zeroes=unmap` + `fstrim -av` immediately reclaimed **3.63 GiB** from `disk.qcow2`. | **Keep strictly for UEFI/firmware/hardware testing** (OVMF/AAVMF, U-Boot, `swtpm`, QMP, `x86_64` TCG). **Replace** for the daily Linux builder VM if/when Lima+`krunkit` is authorized. |
| **Apple `Virtualization.framework` (`vz`) via `vfkit` or Lima `vmType: vz`** | [`crc-org/vfkit`](https://github.com/crc-org/vfkit), [`lima-vm/lima#4220`](https://github.com/lima-vm/lima/issues/4220), [`#2789`](https://github.com/lima-vm/lima/issues/2789) | `Apache-2.0` | Native Apple `virtiofs` (fast, zero `rsync`, eliminates QCOW2 host/guest double buffering for shared source trees) + Rosetta 2 (`--rosetta`) | **Does NOT reclaim host RAM on macOS (`FB22614752`).** Primary-source guest inspection (`lima#4220`) confirms Apple's `VZVirtioTraditionalMemoryBalloonDevice` negotiates only `MUST_TELL_HOST` (bit 0) and `DEFLATE_ON_OOM` (bit 2) — it **omits `VIRTIO_BALLOON_F_REPORTING` (bit 5)**, and even manually inflating the balloon does not return physical host pages to macOS (unchanged through macOS 26 / SDK 26.5 / WWDC 2026). Also note `vz`+`virtiofs` edge cases (`lima#5039` `O_CREAT\|O_EXCL` mode `0000`, `lima#4053` transient uid race). | **Partial win only:** Eliminates custom cloud-init/SSH/`rsync` builder scripts and enables Rosetta 2 for faster x86_64 binaries inside ARM64 guests, **but does NOT solve idle VM memory retention.** |
| **Lima (`vmType: krunkit`) / `libkrun >= v1.19.8`** | [`lima-vm/lima#4220`](https://github.com/lima-vm/lima/issues/4220), [`libkrun/libkrun#703`](https://github.com/libkrun/libkrun/pull/703), [`libkrun/libkrun#931`](https://github.com/libkrun/libkrun/pull/931) (v1.19.8, Nov 6, 2026) | `Apache-2.0` | Native `virtiofs` | **ONLY macOS Hypervisor.framework backend with working guest-to-host memory reclamation.** Negotiates `STATS_VQ` (bit 1), `FREE_PAGE_HINT` (bit 3), and `VIRTIO_BALLOON_F_REPORTING` (bit 5). Uses `MADV_FREE` (`#703`, harvestable under macOS pressure — note Activity Monitor `phys_footprint` stays high even while host free/compressor pressure recovers) and **2 MiB block Replace mode (`#931`, v1.19.8)** to hand freed 2 MiB blocks directly back to macOS. Caveat: `krunkit` `virtiofs` cannot enforce read-only mounts (`lima-vm.io` docs) and `libkrun/krunkit#121` tracks wiring Replace mode into `krunkit`. | **Top candidate to replace `sbm-core`'s custom QEMU Fedora builder (`start-builder.sh` + `provision-builder.sh` + `sync-and-build.sh` + `sbm-lab/src/builder.rs`).** Declarative `lima.yaml`, zero `rsync`, working memory reclaim, Apache-2.0. Evaluate via a single Homebrew trial (`brew install lima krunkit`) upon user approval. |
| **Remote native Linux builders / CI (`ubuntu-latest`, `ubuntu-24.04-arm`)** | GitHub Actions / existing SSH builders | N/A | N/A | Zero local Mac RAM/thermal/SSD cost. | **Default home for dual-arch and `x86_64` release image builds.** Avoid local `x86_64` TCG emulation and cross-arch `qemu-user-static-x86` `mkosi` (measured 215 s + 139 s zstd on M5) except when explicitly needed. |

### 2.5 Build, Compilation, Testing & Cache Tooling

| Tool | Primary Source | License | Compatibility with Host (`cargo 1.100.0`, Nov 2026) | Assessment for SBM |
|---|---|---|---|---|
| **Cargo 1.100 built-in `build.build-dir` (v2 layout) + Auto-GC** | [Cargo `build-dir` docs](https://doc.rust-lang.org/cargo/reference/config.html), Rust 1.91 & 1.100 release notes, `rust-lang/cargo#17312` | `MIT OR Apache-2.0` | **Already installed (`cargo 1.100.0`).** `$CARGO_HOME` download auto-GC is built in (since 1.88). `cargo clean --dry-run` is built in. **Important caveat (`cargo#17312`):** Do NOT share one un-hashed `build-dir` across concurrent worktrees/agents (lock contention + stale-mtime cache poisoning). Use `build-dir = "{cargo-cache-home}/build/{workspace-path-hash}"` to centralize intermediates safely per workspace. | **Reuse directly.** Eliminates any need for a custom Cargo GC subsystem. |
| **`mozilla/sccache`** | [`mozilla/sccache`](https://github.com/mozilla/sccache) (`docs/Rust.md`) | `Apache-2.0` | Native macOS ARM64 & Linux (`brew install sccache`). Requires `SCCACHE_ALLOW_INCREMENTAL=1` so dependencies hit `sccache` while local workspace crates retain fast `rustc` incremental builds. **Limitations:** Does not cache `bin`, `dylib`, `cdylib`, `proc-macro`, or incremental workspace crates. | **Adopt for cross-repo/worktree & CI dependency caching** (replaces redundant recompilation of `serde`, `clap`, `tokio`, `iced`, `fatfs`, etc. across our 8 Rust repos!). |
| **`cargo-zigbuild`** | [`rust-cross/cargo-zigbuild`](https://github.com/rust-cross/cargo-zigbuild) | `MIT` | Native macOS ARM64 (`brew install zig cargo-zigbuild`). Cross-compiles pure-Rust Linux `aarch64-unknown-linux-gnu` / `x86_64-unknown-linux-gnu` directly on macOS without a Linux VM. **Limitation:** Does not cross-link Linux C desktop libraries (`wayland`, `libxkbcommon`, etc. needed by `sbm-ui`). | **Adopt selectively** for pure-Rust Linux binaries (`sbm-cored`, `sbm-ctl`, `sbm-toram`) if we want zero-VM Linux binary builds; keep `sbm-ui` building in Linux VM/CI and previewing natively on macOS via Iced. |
| **`cargo-nextest`** | [`nextest-rs/nextest`](https://nexte.st/) | `MIT OR Apache-2.0` | Updated for Cargo 1.100 v2 `build-dir`, per-test process isolation, JUnit output, build archiving. | **Optional adoption** (especially in CI and `install`'s 83.7 s integration suite for per-test timing visibility and filtering). |
| **`cargo-sweep`** | [`holmgr/cargo-sweep`](https://github.com/holmgr/cargo-sweep) (`#150`) | `MIT` | **Unmaintained** and broken by Rust 1.100's new `build-dir` v2 layout. | **Reject.** Use `cargo clean --dry-run` / centralized `build-dir` instead. |
| **`cargo-cache`** | [`matthiaskrgr/cargo-cache`](https://github.com/matthiaskrgr/cargo-cache) | `MIT OR Apache-2.0` | Reports/prunes `$CARGO_HOME` only (not `target/`). Partially overlaps Cargo 1.88 auto-GC. | **Optional** for manual `$CARGO_HOME` inspection; not essential. |
| **OS Storage Primitives (`cp -c`, `qemu-img`, `fstrim`)** | macOS APFS `clonefile(2)`, QEMU `qemu-img`, Linux `util-linux` | OS built-in | Already verified working on this M5 MacBook Air (`cp -c` rc=0; `discard=unmap` + `fstrim` reclaimed 3.63 GiB; APFS clone-dedup reclaimed 2.56 GiB). | **Reuse directly.** Do NOT build a custom content-addressed storage daemon! |
| **`just` (task runner)** | [`casey/just`](https://github.com/casey/just) | `CC0-1.0` | Single static binary on macOS & Linux (`brew install just`). | **Strong alternative to writing a custom `sbm-dev` CLI framework!** A ~100-line root `justfile` composing `cargo`, `mkosi`, `lima`, `sbm-lab`, and `qemu-img` is declarative, zero-maintenance, and self-documenting. |

---

## 3. Honest Audit of `sbm-lab` (9,805 LOC)

As of 2026-11-26 (including Agent 13's `integration.rs`), `sbm-lab` contains **9,213 LOC of custom Rust** plus 592 LOC of shell scripts and TOML profiles/scenarios (**9,805 LOC total**).

### Module-by-Module Build-vs-Reuse Breakdown

| Module (`sbm-lab/crates/sbm-lab/src/`) | LOC | What It Currently Does | Existing Open-Source / Org Alternative | Recommendation |
|---|---:|---|---|---|
| `resources.rs` | 1,887 | Cross-repo QEMU RAM/vCPU/pressure admission lock (`~/Library/Application Support/sbm-lab`) + unit tests. | **None for macOS multi-agent HVF.** Linux has `systemd-run`/cgroups; macOS has no user-space VM RAM budget guard across concurrent AI agents on a 16 GiB MacBook Air. | **Keep (and do NOT bloat).** Genuine gap on macOS. Keep minimal; do not build a general-purpose job scheduler. |
| `run.rs` | 1,330 | QEMU lifecycle runner, serial/marker watch, QMP screendump, fixture setup, fault injection, report generation. | **`os-autoinst` (`isotovideo`)** on Linux CI; **`labgrid`** for physical + Linux QEMU targets. | **Split & shrink (~70% deletable over time).** Move Linux CI graphical/end-to-end integration runs to `os-autoinst`; retain only a minimal macOS HVF EFI smoke runner (~400 LOC). |
| `cli.rs` | 1,181 | Clap CLI plumbing over all subcommands below. | Shrinks proportionally as modules below are deleted/replaced; top-level workflow ergonomics can use `justfile`. | **Shrink** alongside deleted modules. |
| `integration.rs` | 924 | Graphical recovery integration test (`sbm-uefi` -> `sbm-core` -> Wayland -> GParted) via raw QMP keystrokes & PPM variance. | **`os-autoinst` (`isotovideo`)** with visual needles, OCR, exclude areas, and video capture (Linux CI). | **Replace in Linux CI with `os-autoinst` (`isotovideo`)** once Linux KVM CI/runner integration is adopted; keep working until replaced so Agent 13's passing 9-gate check is never lost. |
| `fixtures.rs` | 615 | Synthetic multi-OS GPT disk fixtures (Windows/BitLocker, Linux BLS, corrupt GPT, multi-ESP, LUKS1 header). | Partially overlaps `install/crates/sbm-media/src/fixture.rs` (`CoreFixtureSpec`, `synthetic_efi_app`). | **Consolidate** into one shared fixture builder (with `sbm-media`) rather than maintaining two disk-image builders across two repos. |
| `product.rs` | 342 | Assembles SBM UEFI menu test disk and discovers `sbm-uefi`/`sbm-core` build artifacts. | Thin SBM-specific glue (depends on consolidated `sbm-media` ESP builder). | **Keep thin / consolidate** disk writing with `sbm-media`. |
| `launch.rs` | 335 | Builds QEMU argv for HVF/TCG, disks, virtio-gpu, USB, networking, `swtpm`. | `os-autoinst` (Linux CI) / `labgrid` `QEMUDriver` / small macOS HVF wrapper. | **Shrink** to minimal macOS HVF UEFI launcher once Linux CI uses `os-autoinst`/`labgrid`. |
| `gpt.rs` | 324 | Custom GPT partition table writer. | **Exact org duplicate** of `install/crates/sbm-media/src/gpt.rs`! | **Delete & reuse** `sbm-media`'s GPT implementation. |
| `accel.rs` | 266 | Host arch + QEMU HVF/KVM/TCG detection. | Trivial (`uname -m` + `qemu-system-* -accel help`); ~50 LOC needed. | **Shrink.** |
| `profile.rs` | 257 | TOML profile/scenario schema loader. | Shrinks if `os-autoinst` / `labgrid` own test scenarios. | **Shrink.** |
| `firmware.rs` | 247 | Discovers Homebrew/distro OVMF, AAVMF, and U-Boot `qemu_arm64` firmware files. | Genuine cross-platform (macOS Homebrew vs Linux distro paths) glue. | **Keep** (small, useful). |
| `qmp.rs` | 219 | Hand-rolled QMP JSON-over-Unix-socket client. | Built into `os-autoinst`, `qemu.qmp` (official upstream Python package). | **Delete** if Python/pytest or `os-autoinst` handles QEMU control; otherwise keep minimal. |
| `util.rs` | 208 | `which`, PE machine check, flock wrapper, tool version probing. | Standard helpers (`sbm-media::pe` already checks PE headers). | **Shrink.** |
| `host.rs` | 190 | `doctor` tool/host capability check. | Useful lightweight diagnostic. | **Keep / simplify.** |
| `secureboot.rs` | 175 | Dev Secure Boot key enrollment & image tampering wrapper (`virt-fw-vars`, `sbsign`). | `virt-fw-vars` / `sbctl` / `sbsigntool` directly in shell or pytest. | **Keep thin or replace with 30-line script.** |
| `markers.rs` | 173 | ANSI stripping & serial marker regex parser. | Built into `os-autoinst` (`wait_serial`) & `labgrid` (`ConsoleProtocol`). | **Delete** when tests move to `os-autoinst`/`labgrid`. |
| `builder.rs` | 127 | Cloud-image & cloud-init seed ISO builder for Linux VM. | **Lima (`lima.yaml`)** (Apache-2.0) completely replaces this + `sbm-core/scripts/{start-builder.sh,provision-builder.sh,sync-and-build.sh}`. | **Delete & replace with Lima** (after trial). |
| `proxy.rs` | 112 | Local TCP delay proxy + `ppm_has_variance` pixel heuristic. | `os-autoinst` needles replace `ppm_has_variance`; `tc`/`toxiproxy` or 20-line Python replaces delay proxy. | **Delete / replace.** |
| `artifacts.rs` | 95 | Scans/fetches release artifacts. | `gh release download` / `install/scripts/fetch-inputs.sh`. | **Delete duplicate.** |
| `fatimg.rs` | 80 | Custom FAT32 ESP image writer. | **Exact org duplicate** of `install/crates/sbm-media/src/esp.rs`! | **Delete & reuse** `sbm-media`'s FAT32 writer. |
| `bench.rs` | 75 | Min/p50/p95/max latency stats. | `hyperfine` (MIT/Apache-2.0) or 20-line Python script. | **Optional delete / replace.** |
| `lib.rs`, `main.rs`, `error.rs` | 58 | Entrypoint & error type. | — | **Keep.** |

### Summary of `sbm-lab` Audit

- **Immediately redundant (~530 LOC in `sbm-lab` + ~250 LOC bash in `sbm-core`):**
  - `gpt.rs` (324 LOC) + `fatimg.rs` (80 LOC) duplicate `install/crates/sbm-media`.
  - `builder.rs` (127 LOC) + `sbm-core`'s cloud-init/SSH/`rsync` builder scripts (~250 LOC) duplicate **Lima**.
- **Replaceable in Linux CI & hardware labs (~3,100+ LOC):**
  - `integration.rs` (924 LOC) + `run.rs` (1,330 LOC) + `qmp.rs` (219 LOC) + `markers.rs` (173 LOC) + `proxy.rs` (112 LOC) reimplement core features of **`os-autoinst` (`isotovideo`)** (visual needles, QMP, serial, video) and **`labgrid`** (board/QEMU control, pytest strategies).
- **What is genuinely justified to keep (~1,500–2,000 LOC total, a ~75–80% reduction):**
  1. **macOS host memory-admission guard** (trimmed `resources.rs`) — protects a 16 GiB fanless Apple Silicon Mac from concurrent AI agents launching overlapping VMs (no OSS equivalent exists on macOS).
  2. **Minimal macOS HVF UEFI smoke launcher** — fast local Level 1 `sbm-uefi` + AAVMF/OVMF/U-Boot boot check on Apple Silicon (where `os-autoinst` and U-Boot `sandbox` cannot run natively with HVF).
  3. **Single shared SBM fixture generator** — consolidated once in `sbm-media`, never duplicated.

---

## 4. Recommended Composed Architecture (Pending User Review)

Instead of inventing new custom subsystems (`sbm-dev` framework, custom storage GC daemon, custom overlay packager, custom job scheduler), compose existing maintained tools across five validation tiers:

| Tier | Goal | Recommended Composed Stack (Reuse First) | Typical Latency |
|---|---|---|---|
| **Level 0** | Native host logic, storage planning, UI layout | `cargo test` (with `build.build-dir` v2 + `sccache`), `sbm-uefi` `xtask preview` (already committed), optional `--mock` mode in `sbm-ui` (tiny ~100 LOC adapter, no new framework) | **0.1–6 s** |
| **Level 1** | Minimal UEFI firmware smoke (native arch) | Thin QEMU HVF + AAVMF smoke on macOS; **U-Boot `sandbox`** (`bootefi` + `test/py`) on Linux CI / Linux dev VM | **2–8 s** |
| **Level 2** | Incremental `sbm-core` Linux app iteration | **Lima (`krunkit` or `vz` + `virtiofs`)** for zero-`rsync` ARM64 dev/build (or `cargo-zigbuild` for pure-Rust `sbm-cored`/`sbm-ctl`), **`mkosi --incremental=yes`** + **`mkosi Format=sysext`** + **`systemd-sysext --mutable=ephemeral`** (zero custom overlay code) | **5–25 s** |
| **Level 3** | Full-system graphical & recovery integration | Native ARM64 QEMU HVF locally (explicit); **`os-autoinst` (`isotovideo`)** with visual needles in Linux CI; **`labgrid`** for physical / Apple Silicon boards | **60 s (HVF) / CI** |
| **Level 4** | Release qualification (signed, compressed, reproducible) | Linux CI / native remote builders for x86_64 + aarch64 single-arch (recommended) and optional experimental dual-arch; `mkosi --incremental=no` + `zstd -19` + `check-reproducible.sh` | **CI / explicit** |

### Concrete Decisions Requested From Project Lead Before Writing New Infrastructure

1. **Linux Builder Backend (`Lima` + `krunkit` trial):** Approve installing `lima` + `krunkit` via Homebrew to run an apples-to-apples benchmark against QEMU HVF (`sbm-core/scripts/start-builder.sh`) on `virtiofs` incremental builds and `libkrun >= 1.19.8` memory reclamation — with the goal of deleting `builder.rs` + custom cloud-init/`rsync` scripts if it wins.
2. **`sbm-lab` Scope Reduction & External Harnesses:** Approve (a) consolidating `sbm-lab`'s duplicate `gpt.rs`/`fatimg.rs` onto `sbm-media`, (b) evaluating `os-autoinst` (`isotovideo`) for Linux CI graphical integration tests (`integration.rs`), and (c) adopting `labgrid` when physical hardware / Apple Silicon board testing begins (without renaming `sbm-lab` yet).
3. **Developer Entrypoint (`justfile` vs custom CLI):** Approve using a thin root **`justfile`** (via `just`) composing existing commands (`cargo`, `mkosi`, `lima`, `sbm-lab`, `sbm-uefi` `xtask`) instead of writing and maintaining a bespoke `sbm-dev` CLI.
4. **Compiler & Test Tooling (`sccache`, `cargo-nextest`, `cargo-zigbuild`):** Approve configuring Cargo 1.100 `build.build-dir = "{cargo-cache-home}/build/{workspace-path-hash}"` + optional `sccache` (`SCCACHE_ALLOW_INCREMENTAL=1`) and `cargo-nextest`, and explicitly skipping unmaintained `cargo-sweep` and custom storage GC daemons.
5. **Distribution Policy Clarification (Phase 9):** Approve updating `ADR-009`, `docs/specifications/07-releases-compatibility-and-diagnostics.md`, and `sbm/RELEASE_GATES.md` (with Agent 11 coordination) so single-arch `x86_64` and generic `aarch64` images are recommended defaults and combined dual-arch remains preserved as an optional experimental feature.

---

## 5. Primary Sources

- TianoCore EDK II `EmulatorPkg` (`EmulatorPkg.dsc`, `build.sh`, `Readme.md`): https://github.com/tianocore/edk2/tree/master/EmulatorPkg
- TianoCore EDK II AArch64 `EmulatorPkg` RFC (Oct 2021, unmerged): https://edk2.groups.io/g/devel/message/82182
- TianoCore `edk2-test` (UEFI-SCT): https://github.com/tianocore/edk2-test
- Intel `EfiWrapper` (archived/unmaintained): https://github.com/intel/efiwrapper
- Das U-Boot Sandbox & UEFI documentation: https://docs.u-boot.org/en/latest/arch/sandbox/sandbox.html and https://docs.u-boot.org/en/latest/develop/uefi/uefi.html
- `os-autoinst` & `openQA`: https://github.com/os-autoinst/os-autoinst and https://github.com/os-autoinst/openQA
- `labgrid`: https://github.com/labgrid-project/labgrid and https://labgrid.readthedocs.io/
- `jc-lab/test-foundry`: https://github.com/jc-lab/test-foundry
- `virtme-ng`: https://github.com/arighi/virtme-ng
- `systemd/mkosi` (`mkosi.1.md`, `docs/sysext.md`, `#4278`): https://github.com/systemd/mkosi
- `systemd-sysext(8)` & `systemd.io` image/extension docs: https://www.freedesktop.org/software/systemd/man/latest/systemd-sysext.html and https://systemd.io/BUILDING_IMAGES/
- `lima-vm/lima` (`vz`, `krunkit`, memory ballooning `#4220`, `#2789`, `FB22614752`): https://github.com/lima-vm/lima/issues/4220
- `libkrun/libkrun` macOS free-page reporting (`#703`, `#931` v1.19.8) & `libkrun/krunkit#121`: https://github.com/libkrun/libkrun/pull/931
- `crc-org/vfkit`: https://github.com/crc-org/vfkit
- `mozilla/sccache` Rust documentation: https://github.com/mozilla/sccache/blob/main/docs/Rust.md
- `rust-cross/cargo-zigbuild`: https://github.com/rust-cross/cargo-zigbuild
- Cargo `build.build-dir` (Rust 1.91 / 1.100) & shared-dir caveat (`rust-lang/cargo#17312`): https://doc.rust-lang.org/cargo/reference/config.html
- `nextest-rs/nextest`: https://nexte.st/
- `holmgr/cargo-sweep` Rust 1.100 compatibility issue (`#150`): https://github.com/holmgr/cargo-sweep/issues/150
