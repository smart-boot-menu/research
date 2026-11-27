# SBM 0.0.1 Graphical Recovery Integration Evidence (2026-11-26)

> **Scope & Honesty Notice:** Every VM run below is an **`N = 1` functional integration run** on a 16 GiB Apple Silicon host under QEMU/OVMF (`hvf` `accelerated_virtualization` for `aarch64`, `tcg` cross-architecture `emulation` for `x86_64`), admitted sequentially through `sbm-lab resources`. **These are NOT performance benchmarks and NOT physical-hardware measurements.** Physical laptop testing and native `x86_64` KVM runs remain unverified.

## 1. Pinned Source Revisions & Reproducible Artifacts

| Component | Repository / Revision | Artifact | Size (bytes) | SHA-256 |
|---|---|---|---:|---|
| Shared types | `sbm-common` `b3c77141504be71bfad6602d8df628f84f704c7b` | — | — | — |
| UEFI menu (`x86_64`) | `sbm-uefi` `653012f4b3bf0351b23aede6001ca4ca1879a0af` | `sbm-uefi-x86_64.efi` | 230,912 | `40f55d0f1c864d6c813e428b049b68b0f1332f037413e80243a31d3c4c4c06cb` |
| UEFI menu (`aarch64`) | `sbm-uefi` `653012f4b3bf0351b23aede6001ca4ca1879a0af` | `sbm-uefi-aarch64.efi` | 206,336 | `4a371d288ba6f7490b163d6892b64e46edbf12479b0d5b93008eabae290991f7` |
| Linux Core (`aarch64`) | `sbm-core` `50316131251bf10fda467fddccc70931978bdc21` | `sbm-core-arm64.raw` | 1,363,530,240 | `dd85499142edf6ea3b0a6e238d720c7c67360963165d78b85c6672adf443606d` |
| Linux Core UKI (`aarch64`) | `sbm-core` `50316131251bf10fda467fddccc70931978bdc21` | `sbm-core-arm64.efi` | 74,513,752 | `5ba2c2a6dee393516b95b9ac36e5009ed8f797571d303f17cdc5cc7744dd210e` |
| Linux Core (`x86_64`) | `sbm-core` `50316131251bf10fda467fddccc70931978bdc21` | `sbm-core-x86-64.raw` | 1,387,459,072 | `fbfd9e5b9ad39d2fad7d853bd3634fc43eddb34ee4e48e6a06ffa2adff789a28` |
| Linux Core UKI (`x86_64`) | `sbm-core` `50316131251bf10fda467fddccc70931978bdc21` | `sbm-core-x86-64.efi` | 80,909,144 | `1d5f1011f43decd0f120525865e36d66f5c0ac63179f845e73b6510fa803d9d8` |
| Dual-arch raw media | `install/media/0.0.1.lock.toml` | `sbm-0.0.1-uefi-dualarch.raw` | 2,216,689,664 | `7be07b851f1e61be2544027c1caaf28febcc385bf01d8978b12738bef3be64a9` |
| Dual-arch zstd media | `install/media/0.0.1.lock.toml` | `sbm-0.0.1-uefi-dualarch.raw.zst` | 1,756,287,279 | `e18a41b08ac71c01883a73b8d4b94c1ad81be965f972568f93e3dbd8434b23a8` |

Reproducibility verified with `./scripts/check-reproducible.sh media/0.0.1.lock.toml`: all 6 outputs (`sbm-0.0.1-uefi-dualarch.raw`, `.sha256`, `.raw.zst`, `.raw.zst.sha256`, `release-manifest.json`, `build-receipt.json`) were byte-identical across two independent builds.

- `aarch64` roothash: `925e22ebd0dc9ce7fb72e24c1d3787aa648cab7606124570a0ca26d13190b412` (root `925E22EB-D0DC-9CE7-FB72-E24C1D3787AA`, verity `648CAB76-0612-4570-A0CA-26D13190B412`)
- `x86_64` roothash: `d4d79238973794157dba3ddb380d020262d80b228a1b672196c852043f41c5f7` (root `D4D79238-9737-9415-7DBA-3DDB380D0202`, verity `62D80B22-8A1B-6721-96C8-52043F41C5F7`)

## 2. Full End-to-End Integration Runs (`sbm-lab test --tier integration`)

Exact same unchanged `sbm-0.0.1-uefi-dualarch.raw` (`7be07b85...`) mounted behind disposable QCOW2 overlays as removable USB (`bootindex=0`) with decoy ESP (`SBMDECOY`, enumerated first as `/dev/vda`), blank 1 GiB (`SBMTEST1G`, `/dev/vdb`) and 2 GiB (`SBMTEST2G`, `/dev/vdc`) virtual disks, zero virtual NIC, and no `-no-reboot`.

### A. `aarch64` (`hvf`, `virtio-gpu-pci,xres=1280,yres=800`, 4096 MiB RAM, 2 vCPU)

- **Run ID:** `int-aarch64-18ebcad4c2d656f8-4847-1` (`success: true`, `duration_ms: 61299`, **9 / 9 gates passed**)
- **Stage timings:** `uefi_menu_1: 6844 ms`, `core_ui_ready: 14793 ms`, `gparted_launched: 18099 ms`, `uefi_menu_2: 40483 ms`
- **UEFI & Core markers:**
  - `SBM_MARK first_frame us=8650`, `SBM_MARK discovery_done us=1079487 count=2`, `SBM_MARK target Fedora Linux (Decoy Fixture)`, `SBM_MARK target SBM Core Slot A`, `SBM_MARK menu_ready us=1082836`, `SBM_MARK key c`, `SBM_MARK chainload SBM Core Slot A`
  - `SBM_PROTECT status=backing-protected backing=/dev/sda esp=/dev/sda1 ro_partitions=4`
  - `SBM_BOOT_METRICS mode=backing-protected mem_total_kib=3976632 mem_available_kib=3559704 kernel=7.3.4-300.fc45.aarch64 backing=/dev/sda handoff_verified=true`
  - `SBM_UI_READY handoff_verified=true esp=/dev/sda1 root_source=/dev/mapper/root`
  - `SBM_BOOT_SUCCESS_WRITTEN attempt_id=uefi-1727508 target_id=sbm-core:slot-a`
  - `SBM_GPARTED_LAUNCHED pid=568`
  - `SBM_POWER_ACTION action=reboot` -> QMP `RESET` (`guest-reset`)
  - Post-reboot: `SBM_MARK attempt confirmed uefi-1727508`, `SBM_MARK menu_ready us=1090082`
- **Post-run ESP & Integrity checks:** `last_attempt.json`: `status=confirmed`, `consecutive_failures=0`, `attempt_id=uefi-1727508`; `current_handoff_present=true`, `boot_success_consumed=true`, `decoy_esp_unchanged=true`, `inputs_unchanged=true`.
- **Screenshots:**
  - [`screenshots/2026-11-26-graphical-recovery/aarch64-01-uefi-menu.png`](screenshots/2026-11-26-graphical-recovery/aarch64-01-uefi-menu.png)
  - [`screenshots/2026-11-26-graphical-recovery/aarch64-02-core-ui.png`](screenshots/2026-11-26-graphical-recovery/aarch64-02-core-ui.png)
  - [`screenshots/2026-11-26-graphical-recovery/aarch64-03-gparted.png`](screenshots/2026-11-26-graphical-recovery/aarch64-03-gparted.png)
  - [`screenshots/2026-11-26-graphical-recovery/aarch64-04-gparted-devices.png`](screenshots/2026-11-26-graphical-recovery/aarch64-04-gparted-devices.png)
  - [`screenshots/2026-11-26-graphical-recovery/aarch64-05-post-reboot-menu.png`](screenshots/2026-11-26-graphical-recovery/aarch64-05-post-reboot-menu.png)

### B. `x86_64` (`tcg`, `virtio-vga,xres=1280,yres=800`, 4096 MiB RAM, 2 vCPU)

- **Run ID:** `int-x86_64-18ebcae5732ae478-490e-1` (`success: true`, `duration_ms: 159140`, **9 / 9 gates passed**)
- **Stage timings:** `uefi_menu_1: 18759 ms`, `core_ui_ready: 74439 ms`, `gparted_launched: 80901 ms`, `uefi_menu_2: 138909 ms`
- **Renderer fallback observed live:** `sbm-ui` detected Mesa `llvmpipe` missing `SHADER_FLOAT16_IN_FLOAT32` under `wgpu` and automatically restarted with `ICED_BACKEND=tiny-skia`, displaying `Renderer: software (wgpu fallback: tiny-skia)` and `desktop ready (confirmed uefi-16688591)`.
- **UEFI & Core markers:**
  - `SBM_MARK first_frame us=28922`, `SBM_MARK discovery_done us=15994161 count=2`, `SBM_MARK menu_ready us=16024663`, `SBM_MARK key c`, `SBM_MARK chainload SBM Core Slot A`
  - `SBM_PROTECT status=backing-protected backing=/dev/sda esp=/dev/sda1 ro_partitions=4`
  - `SBM_BOOT_METRICS mode=backing-protected mem_total_kib=3986928 mem_available_kib=3623448 kernel=7.3.4-300.fc45.x86_64 backing=/dev/sda handoff_verified=true`
  - `SBM_UI_READY handoff_verified=true esp=/dev/sda1 root_source=/dev/mapper/root`
  - `SBM_BOOT_SUCCESS_WRITTEN attempt_id=uefi-16688591 target_id=sbm-core:slot-a`
  - `SBM_GPARTED_LAUNCHED pid=619`
  - `SBM_POWER_ACTION action=reboot` -> QMP `RESET` (`guest-reset`)
  - Post-reboot: `SBM_MARK attempt confirmed uefi-16688591`, `SBM_MARK menu_ready us=16421689`
- **Post-run ESP & Integrity checks:** `last_attempt.json`: `status=confirmed`, `consecutive_failures=0`, `attempt_id=uefi-16688591`; `current_handoff_present=true`, `boot_success_consumed=true`, `decoy_esp_unchanged=true`, `inputs_unchanged=true`.
- **Screenshots:**
  - [`screenshots/2026-11-26-graphical-recovery/x86_64-01-uefi-menu.png`](screenshots/2026-11-26-graphical-recovery/x86_64-01-uefi-menu.png)
  - [`screenshots/2026-11-26-graphical-recovery/x86_64-02-core-ui.png`](screenshots/2026-11-26-graphical-recovery/x86_64-02-core-ui.png)
  - [`screenshots/2026-11-26-graphical-recovery/x86_64-03-gparted.png`](screenshots/2026-11-26-graphical-recovery/x86_64-03-gparted.png)
  - [`screenshots/2026-11-26-graphical-recovery/x86_64-04-gparted-devices.png`](screenshots/2026-11-26-graphical-recovery/x86_64-04-gparted-devices.png)
  - [`screenshots/2026-11-26-graphical-recovery/x86_64-05-post-reboot-menu.png`](screenshots/2026-11-26-graphical-recovery/x86_64-05-post-reboot-menu.png)

### C. Pure Software-Rendering Fallback (`ramfb` / `simpledrm`, `aarch64` HVF, 2048 MiB RAM)

- Booted the exact same `sbm-0.0.1-uefi-dualarch.raw` with `-device ramfb` (no `/dev/dri/renderD128`).
- `start-session` selected `WLR_RENDERER=pixman` + `ICED_BACKEND=tiny-skia`.
- Reached `SBM_UI_READY` (`6.20 s`) and `SBM_BOOT_SUCCESS_WRITTEN attempt_id=uefi-1666419`, displaying `Renderer: software (no DRM render node: pixman + tiny-skia)` and `desktop ready (confirmed uefi-1666419)`.
- **Screenshot:** [`screenshots/2026-11-26-graphical-recovery/aarch64-ramfb-software-ui.png`](screenshots/2026-11-26-graphical-recovery/aarch64-ramfb-software-ui.png)

## 3. Negative Gate Verifications (Disposable Copies, `aarch64` HVF)

1. **Tampered UKI (1 byte flipped in `EFI/sbm/recovery/slot-a/sbm-core-aarch64.efi`):**
   - `sbm-uefi` logged `SBM_MARK target SBM Core Slot A (unverified)`.
   - Pressing `c` refused launch and displayed *"Cannot start this operating system — No healthy recovery system was found — Enter or Esc returns to the menu"* ([`aarch64-negative-tampered-uki.png`](screenshots/2026-11-26-graphical-recovery/aarch64-negative-tampered-uki.png)). Zero chainload, zero Linux boot.
2. **Corrupted dm-verity (4 KiB flipped in `root-arm64-verity`):**
   - Kernel logged `device-mapper: verity: 8:4: metadata block 1 is corrupted` and `reached maximum errors`; never mounted EROFS, never emitted `SBM_UI_READY`, never wrote `boot-success.json`.
3. **Decoy ESP priority:** Enumerated before boot USB as `/dev/vda` in both positive integration runs; `decoy_esp_unchanged: true`, Core bound strictly to `/dev/sda1`.
4. **Invalid handoff:** Verified in `sbm-runtime::handoff` unit tests (mismatched digest, attempt ID, arch, decoy ESP rejection) and observed live on direct boot (`SBM_BOOT_SUCCESS_DECLINED reason=no EFI handoff file`).

## 4. Host Resource Measurements (16 GiB Apple Silicon)

- Shared resource manager (`sbm-lab resources`) admitted every VM sequentially (`heavy limit 1`, `guest budget 5120 MiB`).
- Peak active guest reservation: `4096 MiB`, `2 vCPU` (measured `qemu-system-x86_64` RSS ~630 MiB during TCG run).
- Host memory pressure remained `normal` across all runs (`swap 651–699 / 2048 MiB`).
- Post-run `sbm-lab resources doctor`: `running 0/2`, `reservations: none`, `qemu: none`, `pressure normal`.

## 5. Canonical Release Gate Scorecard (`sbm/RELEASE_GATES.md` Update)

| Gate | `aarch64` (AAVMF / HVF) | `x86_64` (OVMF / TCG) | Status & Remaining Non-Core Notes |
|---|---|---|---|
| **0. Immutable reproducible build** | PARTIAL | PARTIAL | Core built from clean committed `5031613`, unified `sbm-common` `b3c77141`, and `check-reproducible.sh` passed byte-identically on `0.0.1.lock.toml`. Remaining non-Core gap: `install` `release-manifest.json` still emits `"artifacts": []` (populates `"media"` only). |
| **1. Real UEFI menu & startup latency** | PARTIAL (1,083 ms) | PARTIAL (16.0 s TCG) | Graphical menu verified with QMP frames on both arches; remaining non-Core gap in `sbm-uefi` (`653012f`): discovery-time UKI hashing exceeds `< 250 ms` HVF target. |
| **2. Discovered targets** | **PASS** | **PASS** | Both `Fedora Linux (Decoy Fixture)` and `SBM Core Slot A` verified in serial and QMP frames. |
| **3. Manage Computer & Spec 03 handoff** | PARTIAL | PARTIAL | Fail-closed UKI verification, `current.json`/`last_attempt.json` on boot ESP only, and SHA-256 handoff verification pass; remaining non-Core gap in `sbm-uefi`: still uses cmdline load options instead of all 5 volatile EFI vars (`sbm-core` already supports both). |
| **4. Real Linux Core (dm-verity EROFS)** | **PASS** | **PASS** | Strict boot ESP binding (`/dev/sda1`, decoy `/dev/vda` untouched), dm-verity EROFS mount, backing-disk RO protection (`ro_partitions=4`, ESP writable). |
| **5. Management UI (`sbm-ui`)** | **PASS** | **PASS** | Live `labwc` + Iced `sbm-ui` with daemon-backed data, fsynced `boot-success.json`, verified `virtio-gpu` + `ramfb` (`pixman + tiny-skia`) + `virtio-vga` (`wgpu -> tiny-skia`) fallback. |
| **6. GParted & virtual disks** | **PASS** | **PASS** | Real GParted via XWayland with QMP frames showing open device dropdown listing `/dev/sda`, `/dev/vda`, `/dev/vdb` (1.00 GiB), `/dev/vdc` (2.00 GiB). |
| **7. Return to menu & confirmation** | **PASS** | **PASS** | Single clean guest reboot (`RESET` event, no `requested-command.json`), return to UEFI menu, `last_attempt.json` `status=confirmed`, `consecutive_failures=0`, `boot-success.json` consumed. |
| **8. Offline operation** | **PASS** | **PASS** | Zero `-netdev` across full Gates 1–7 loop on both arches. |
| **9. Integrity & resource hygiene** | **PASS (local)** | **PASS (local)** | `fmt`/`clippy`/`test` pass in `sbm-core` (22), `sbm-lab` (52), `install` (61); input `.raw` and firmware hashes unchanged; `sbm-core` CI pin updated; hosted CI re-run pending push. |
