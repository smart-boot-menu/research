# SBM 0.0.1 Wave 2 integration audit evidence — 2026-11-26

Independent evidence collected during the second cross-repository integration audit across `~/gh/smart-boot-menu/`. No physical disks, firmware settings, or host boot policies were modified. No new VMs were launched; all observations come from source inspection, archived run logs, GitHub Actions artifacts, and offline host unit tests.

## 1. Audited revisions, CI runs, and local host tests

| Repository | Audited commit | Latest hosted CI run | Offline host `cargo test` |
|---|---|---|---|
| `sbm-common` | `b3c77141504be71bfad6602d8df628f84f704c7b` (unified `main`, merging `4b43222` + `57e9b3d`) | [45906797387](https://github.com/smart-boot-menu/sbm-common/actions/runs/45906797387) (pass) | 8 passed (3+3+2) + both UEFI `no_std` checks |
| `docs` | `157d34ca440dc98d5792c19b043c84ebb1f48327` (unified `main`) | [PR #1 Docs Check](https://github.com/smart-boot-menu/docs/pull/1) (pass) | N/A (docs) |
| `sbm-uefi` | `2be0f654f27ff9eecb39948a29595a8a0e80f364` | [45900855497](https://github.com/smart-boot-menu/sbm-uefi/actions/runs/45900855497) (**FAIL**: `cargo fmt` at `crates/xtask/src/qemu.rs:107-110`) | 28 passed (18+5+5) |
| `sbm-core` | `14efe2b` (`c7e9439` added `g sbm 1000` after candidate images were built) | [45900852498](https://github.com/smart-boot-menu/sbm-core/actions/runs/45900852498) (pass) | 19 passed (macOS host subset) |
| `sbm-lab` | `8b63fd30f0f1ed4e750fc5b2ce4a75a19a6a0156` | [45900849263](https://github.com/smart-boot-menu/sbm-lab/actions/runs/45900849263) (**FAIL**: `smoke`, `product`, `secure`; `check` pass) | 42 passed |
| `install` | `f9248b8b8480276c4298fc6c3ab5c344d8a813f8` | [45902812780](https://github.com/smart-boot-menu/install/actions/runs/45902812780) (pass) | 61 passed (39+9+8+4+1) |
| `sbm-server` | `1983995e7250e1ec4396cb75e8b8b430011e100a` | [45891616013](https://github.com/smart-boot-menu/sbm-server/actions/runs/45891616013) (pass) | 19 passed (3+11+5) |
| `sbm-catalog` | `091dfb2` | [45900858374](https://github.com/smart-boot-menu/sbm-catalog/actions/runs/45900858374) (pass) | 22 passed |
| `sbm-as` | `e9f0e53ed2b30d76f9309d68f6239f77ff46d6c0` | [45900860702](https://github.com/smart-boot-menu/sbm-as/actions/runs/45900860702) (pass) | 17 passed |
| `sbm-agent` / website | `9821b2b` / `3301e39` | Scaffolds | N/A |

## 2. Candidate 2.06 GiB dual-arch image provenance & evidence

- Raw image: `install/build/out/release/sbm-0.0.1-uefi-dualarch.raw`, `2,215,641,088` bytes, SHA-256 `c7ff8cc1551a107e4a1d7dbc58ef027468296f946ed22c26c3158ea9f2bee7f5` (verified live with `sbm-install media verify --quick`).
- Compressed image: `sbm-0.0.1-uefi-dualarch.raw.zst`, `1,754,487,243` bytes, SHA-256 `2906358e66e939e75386bcf1c66c342ef9565798c156fe9a744eb6b31f1e214d`.
- **Not reproducible from immutable committed source**:
  - `build-receipt.json` records `install` `git_commit` `3037bbceb93751d7014857ea0d081eb184b93d52` (2 commits behind `f9248b8`).
  - Core inputs (`sbm-core-arm64.raw` `a46f7c5a...`, `sbm-core-x86-64.raw` `9e7e3952...`) were built from `sbm-core` `0a49e45211` plus uncommitted local edits and predate `c7e9439` (`g sbm 1000`).
  - `sbm-core/mkosi.conf` pulls unpinned Fedora 45 packages, `scripts/build-inside.sh` generates ephemeral `mkosi genkey` keys, and `mkosi` generates random dm-verity salts.
- **Observed lab runs (`N = 1` functional runs, not benchmarks)**:
  - `aarch64` HVF (`6b08a7bd-64238-422656625575`, 4096 MiB, `ramfb`): booted `sbm-uefi` -> `SBM Core Slot A` -> Linux `7.3.4-300.fc45.aarch64` -> `systemd-veritysetup@root` -> `erofs (dm-0)` -> `sbm-cored` -> `sbm-core login:`.
  - `x86_64` TCG (`6b08a893` 4096 MiB, `6b08ab26` 1024 MiB, `VGA`): `serial-qemu.log` reached `sbm-core login:` on `7.3.4-300.fc45.x86_64`, but `install/build/lab-release/x86_64-run.json` is `0` bytes because the 600 s command timeout killed `sbm-lab` before its 720 s TCG timeout completed.
  - `install/screenshots/x86_64-0.0.1-handoff.png` (`3b26dda7...`) is byte-identical to `install/screenshots/x86_64-uefi-splash.png` because `sbm-lab/crates/sbm-lab/src/run.rs:659-668` takes only one early QMP screendump on `first_frame` / `menu_ready`.

## 3. UEFI startup latency regression breakdown

| Run / artifact | UKI payload read | `first_frame` | `discovery_done` | `menu_ready` | Discovery share |
|---|---|---:|---:|---:|---:|
| `sbm-uefi` `562b227` fixture, aarch64 HVF | 2 × 18.9 KiB | 5.3 ms | 14.3 ms | 21.8 ms | 41.1% |
| `install` fixture dual-arch, aarch64 HVF (`6b089d44`) | 2 × 18.4 KiB | 3.5 ms | 7.5 ms | 15.0 ms | 26.4% |
| **Real `0.0.1` dual-arch, aarch64 HVF (`6b08a7bd`, 4096 MiB)** | **74.5 MB + 80.9 MB = 155.4 MB** | **6.3 ms** | **1,744.7 ms** | **1,748.1 ms** | **99.4%** |
| `install` fixture dual-arch, x86_64 TCG (`6b089a4b`) | 2 × 18.4 KiB | 54.8 ms | 103.6 ms | 150.4 ms | 32.4% |
| **Real `0.0.1` dual-arch, x86_64 TCG (`6b08a893`, 4096 MiB)** | **155.4 MB** | **39.9 ms** | **16,939.1 ms** | **16,963.9 ms** | **99.6%** |
| **Real `0.0.1` dual-arch, x86_64 TCG (`6b08ab26`, 1024 MiB)** | **155.4 MB** | **54.6 ms** | **31,610.6 ms** | **31,687.6 ms** | **99.6%** |

Exact root causes verified in `sbm-uefi`:
1. Eager synchronous SHA-256 hashing inside `discover_volume()` -> `scan_recovery()` (`crates/sbm-boot/src/discover.rs:179`) before menu paint.
2. Dual-arch double-hashing: `scan_recovery()` hashes both `sbm-core-x86_64.efi` (80.9 MB) and `sbm-core-aarch64.efi` (74.5 MB) before `push_file()` (`discover.rs:570-575`) filters by `pe_machine()`.
3. Catastrophic $O(N^2)$ FAT32 reopen + seek: `UefiVolume::read_at()` (`crates/sbm-app/src/volume.rs:95-97,286-308`) calls `open_volume()`, `root.open()`, `file.set_position(offset)`, and allocates a fresh 64 KiB `Vec` on every chunk (`2,372` times), causing EDK2 `EnhancedFatDxe` to walk the 4 KiB FAT32 cluster chain from cluster 0 each time (~22.5 million cluster traversals vs ~19,000 sequential hops).
4. Pure-software `sha2` (`default-features = false`) compiled with `[profile.release] opt-level = "z"` prevents round unrolling over 2.43 million SHA-256 blocks.

## 4. Hosted CI failure root causes (`sbm-lab` run `45900849263`)

Downloaded GitHub Actions artifacts (`smoke-logs`, `product-logs`, `secure-logs`) confirm two distinct defects:
1. **`smoke` & `product` AArch64 timeout (`720 s` / `1,200 s`)**: `sbm-lab/crates/sbm-lab/src/accel.rs:196` passes `-cpu max` to `qemu-system-aarch64` under TCG on `ubuntu-latest` (`x86_64`) with `AAVMF_CODE.no-secboot.fd` (`2025.11-3ubuntu7.4`). Under TCG, `-cpu max` enables full software QARMA5 Pointer Authentication (`pauth-impdef=off`), stalling AAVMF immediately after the single banner line `UEFI firmware (version 2025.11-3ubuntu7.4 built at 12:58:05 on Oct  8 2026)`.
2. **`secure` job (`secure-unsigned-rejected`, `secure-tampered-rejected`)**: OVMF (`OVMF_CODE_4M.secboot.fd`) actually rejected both images in `3.7 s` (`BdsDxe: failed to load Boot0002 ...: Access Denied -- rejected probably by Secure Boot`), and `stop_on_serial` exited cleanly, but `sbm-lab/crates/sbm-lab/src/run.rs:495-505` unconditionally sets `failure = Some("guest did not emit efi-launched")` without checking `forbid_markers` or `stop_on_serial`.

## 5. Graphical recovery boot blockers verified in `sbm-core` logs

1. **`status=217/USER` (`sbm-core/build/evidence/arm64-journal/serial.log:1059,1265-1267`)**: `systemd-sysusers` failed (`Failed to create sbm: please create GID 1000`) because both candidate images predate `c7e9439`. `sbm-session.service` crashed in 3–80 ms before `labwc` ever started, even under `virtio-gpu-pci` (`arm64-software-4g`). All three display devices (`ramfb`/`simpledrm`, `VGA`/`bochs-drm`, `virtio-gpu-pci`/`virtio_gpu`) successfully registered `/dev/dri/card0`.
2. **ESP mounted at `/boot` instead of `/efi` (`arm64-journal/serial.log:1078,1115,1161,1255`)**: `systemd-gpt-auto-generator` auto-mounts the ESP at `/boot` because `/efi` does not exist in EROFS at generator time, causing `sbm-smoke.service` to skip, `start-session` to miss `/efi/EFI/sbm/state/renderer`, and `sbm-cored::write_esp()` (`crates/sbm-cored/src/lib.rs:547-598`) to ignore `mount` errors and write into volatile tmpfs.
3. **`resolve_disk("/dev/mapper/root")` always returns `None` (`crates/sbm-cored/src/lib.rs:639-669`)**: Missing `fs::canonicalize()` before stripping `/dev/` causes `sbm-protect.service` to print `unchanged` (`arm64-journal/serial.log:1099`), leaves `ProtectedSbmRecovery` unclassified, and leaves GParted with no RO boot-disk protection.
4. **Non-root `/dev/kmsg` `EACCES` & missing software-renderer auto-selection (`mkosi.extra/usr/lib/sbm/start-session:19-40`)**: `start-session` runs as UID 1000 so `SBM_DESKTOP_READY` and `SBM_LABWC_EXIT` silently fail with `Permission denied`, and it never checks `[ ! -e /dev/dri/renderD128 ]` to auto-enable `WLR_RENDERER=pixman` + `ICED_BACKEND=tiny-skia`.

## 6. Shared host resource manager audit (`sbm-lab` `8b63fd3` & integrations)

Critical defects verified in source and on disk:
1. **Shell subshell `owner_pid` mismatch (`resources.rs:822-850`, `sbm-resources.sh:31-43`, `sbm-core` `14efe2b`, `sbm-as` `e9f0e53`)**: `hold=$(sbm_reserve ...)` records the dead subshell PID, causing `sbm_attach` to fail deterministically and kill freshly spawned VMs.
2. **Self-detection in `resources run` (`resources.rs:983-1012`, `sbm-catalog` `091dfb2`)**: `read_qemu()` substring-matches `"qemu-system-"` in `ps` args without excluding self PID or checking `argv[0]` basename, causing `sbm-lab resources run -- qemu-system-...` to double-count itself and self-refuse heavy workloads.
3. **1-second `short_id()` collision (`util.rs:160-168`, `resources.rs:801-812`)**: Confirmed on disk in `sbm-lab/target/sbm-lab-tests/heavy-7/reservations/`, where an `Efi` reservation silently overwrote a `Build` reservation created in the same second.
4. **PID-reuse `SIGKILL` on drop (`resources.rs:241-253`, `boot-smoke.sh:212`)**: Unconditionally sends `SIGKILL` to already-reaped QEMU PIDs.
5. **Arbitrary `SIGKILL` in `clean --stop` (`cli.rs:939-956`)** and **unconditional `--forget` fallback in `sbm_release` (`sbm-resources.sh:45-55`)**.
6. **Additional concurrency, persistence, and accounting gaps**: `collect_host()` runs outside `admit.lock`; detached builders always fail `resources doctor`; persistent runtime dir survives reboot without `boot_time`/process start-time validation and treats `EPERM` as alive; `parse_memory_mib`/`parse_smp` undercharge `-m 4G,slots=...` and `-smp cores=4` to 1 MiB / 1 vCPU; `sbm-core` UKIs are clamped to `Workload::Efi` (1024 MiB / 1 vCPU); Linux 16 GiB CI runners hit the `total_mib < 16384` budget cliff plus an unneeded 2× macOS footprint tax; missing sibling `../sbm-lab` breaks standalone checkouts in `sbm-core`, `sbm-uefi`, `sbm-catalog`, and `sbm-as`; `--id` lacks path-traversal validation.
