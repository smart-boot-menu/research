# SBM 0.0.1 Qualification Evidence — `x86_64` (2026-11-27)

- **Release:** `0.0.1` (`x86_64` single-architecture image; `dualarch` remains optional/experimental and non-gating)
- **Trust Tier:** **Development / Preview** (`production_secure_boot_claimed: false`, `production_tuf_signed: false`; explicitly distinguished from future SBM 0.2+ consumer Secure Boot releases)
- **Qualification Verdict:** **FUNCTIONAL & NEGATIVE GATES PASS / GATE 1 NATIVE-KVM REAL-MEDIA `N>=10` LATENCY PENDING** — Gate 0 (immutable compressed reproducibility), Gates 2–9 (both `virtio-vga` and `VGA` software fallback), and all 6 Tier 2 negative/request scenarios pass on the final immutable `sbm-0.0.1-uefi-x86_64.raw` (under Apple Silicon TCG emulation, functional-only). Hosted native Linux KVM CI (`sbm-uefi@4bfb3ba`, run `45980442284`) verified the exact frozen `sbm-x86_64.efi` binary at `67.67 ms` cold / `51.47 ms` warm (`69.96 ms` under enrolled Secure Boot) on synthetic fixture media, plus enrolled Secure Boot refusal (`ACCESS_DENIED`). Per the Honest Qualification Rule in `sbm/RELEASE_GATES.md`, TCG latency (`189.6 ms`) is NOT cited as `< 250 ms` compliance; `N >= 10` native-KVM timing on the assembled `sbm-0.0.1-uefi-x86_64.raw` remains pending a native `x86_64` KVM host (or physical Tier 3 run on `EXP-HW-HP01`).
- **Machine-Readable Bundle:** [`sbm-0.0.1-qual-x86_64-2026-11-27.json`](sbm-0.0.1-qual-x86_64-2026-11-27.json)

## 1. Frozen Immutable Source Revisions & Hosted CI

| Repository | Frozen Commit | Hosted CI Run | Status |
|---|---|---|---|
| `sbm-common` | `abe3029c831f05d77d10cfb214a0c236903d00d9` | `45958218582` | `success` |
| `sbm-uefi` | `4bfb3ba6457a4f6625d13c6a1435f69f15b8d011` | `45980442284` | `success` (includes native x86-64 KVM OVMF + Secure Boot denial) |
| `sbm-core` | `960f11a5e748e30f105cd79cc648f0633af63138` | `45977817124` | `success` |
| `install` | `870069dccd2b7fc680a35955c27a6481bb46a0e4` | `46014761860` | `success` |
| `sbm-lab` | `5ab4f8c0485a226fc639778a65a8a1467c352bde` | `46059331789` | `success` |
| `sbm-as` | `759be4eabed0cb760eabe5d8befcbba43fe067c6` | `45977832267` | `success` |
| `sbm-server` | `ae18f7832faae6a8b2ad7de709c0ebc733a73531` | `45940770054` | `success` |

## 2. Final Artifact Inventory, Sizes & Compressed Reproducibility (Gate 0)

Built cleanly with `SBM_BUILD_PROFILE=release` (`Incremental=no`, zero `sbm-smoke` files/presets verified inside mounted read-only EROFS root), locked via `install/media/0.0.1.lock.toml` (SHA-256 `a88928d218afc2785dfecf0765b4cf7f3b8920948de0d1accb13be3a92eed213`), verified by `sbm-install media verify`, and proven byte-identical across two independent clean builds with zstd compression enabled (`check-reproducible.sh`: `reproducible: true`, `compressed: true`, 6/6 output files identical).

| Artifact | Size | SHA-256 |
|---|---:|---|
| `sbm-x86_64.efi` | `465,920 B` (`455.0 KiB`) | `729ef70a39d016c00cf70a9a750632dfde5c00617af444aa41db9ccee48948a3` |
| `sbm-core-x86-64.efi` (UKI) | `80,909,144 B` (`77.16 MiB`) | `5b470091531e2196cb35de3fcc86f040ef56c8d774fecaafc152126a0593672c` |
| `sbm-core-x86-64.raw` (Core input) | `1,387,549,184 B` (`1,323.3 MiB`) | `05cd737e76fa9cf9862e7bb679669c15e69a282da538bda772c4f3406a73297d` |
| `sbm-0.0.1-uefi-x86_64.raw` | `1,120,927,744 B` (`1,069.0 MiB`) | `9bebf042be7c79feea29d2080c192ea50133d74fb29612545356b1e23756a378` |
| `sbm-0.0.1-uefi-x86_64.raw.zst` | `896,237,084 B` (`854.7 MiB`) | `4f825fc1eaedebe6cf2436a3ea33613c4be9879e7e950de313d11e115a23381e` |
| `release-manifest.v1.json` | `1,299 B` | `af26a50775ee6c0a6ea79099683cc1a2e79773734c3c73ae413eb82946dec70d` |
| `build-receipt.json` | `8,147 B` | `be7d18e715e8f1127106bd1a5eb829f0177c6d80d9a5e97227713c82be8d57ef` |
| `dm-verity` `roothash` | — | `081f2d35cbf028fd6769e565782101556303447414aed0e20ed34b26996f366e` |

## 3. Gate Scorecard (`x86_64` — Hosted KVM CI + Local TCG OVMF)

| Gate | Result | Measured Evidence |
|---|---|---|
| **Gate 0 — Immutable reproducible build** | **PASS** | `repro-report.json`: `reproducible: true`, `compressed: true` (6/6 files identical); `sbm-install media verify` green; `production_secure_boot_claimed: false`, `production_tuf_signed: false`, `experimental: false`. |
| **Gate 1 — Real UEFI menu & startup speed (`N=10` KVM)** | **PARTIAL (KVM real-media `N>=10` pending)** | Hosted GitHub Actions native `ubuntu-latest` KVM (`sbm-uefi@4bfb3ba`, run `45980442284`, synthetic fixture, `1280×800`, 5 targets): `menu_ready` **`67.67 ms` cold / `51.47 ms` warm** (`69.96 ms` under enrolled Secure Boot), glyph cache `13,076 B`. Real-media OVMF graphical picker visually verified locally under TCG (`189.60 ms` virtio / `189.35 ms` VGA; `latency_budget_applicable: false` — NOT cited as `< 250 ms` compliance). |
| **Gate 2 — Discovered targets** | **PASS** | Picker rendered both `Fedora Linux (Decoy Fixture)` and `SBM Core Slot A` (visually verified). |
| **Gate 3 — UKI verification & Spec 03 handoff** | **PASS** | Streaming 1 MiB SHA-256 `verify_done`: `result=ok` (`1,341.57 ms` virtio / `1,359.85 ms` VGA under TCG for 80.9 MiB UKI; `968 µs` on native KVM fixture), all 5 volatile EFI vars + exact boot ESP UUID + UUIDv4 `attempt_id` + empty load options. |
| **Gate 4 — Real Linux Core (`dm-verity` EROFS, zero smoke)** | **PASS** | Booted Fedora 45 kernel (`7.3.4-300.fc45.x86_64`), resolved boot ESP via BLI despite decoy ESP first, mounted `/dev/mapper/root`, `SBM_PROTECT backing-protected` on boot & GParted exit, zero `SBM_SMOKE` units/markers. |
| **Gate 5 — Live Management UI & software fallback** | **PASS** | Passed 9/9 on both `virtio-vga` (`1280×800`, 118.4 s) AND `-device VGA` (`1024×768`, 113.1 s; `Renderer: software (no DRM render node: pixman + tiny-skia)`). |
| **Gate 6 — GParted & virtual disks** | **PASS** | Real GParted via XWayland (`gparted_visual_delta`: `0.4381` virtio, `0.4386` VGA); device dropdown displayed `/dev/sda`, `/dev/vda`, `/dev/vdb (1.00 GiB)`, `/dev/vdc (2.00 GiB)`; backing RO refcounted safely; `fixture_disks_unchanged: true`. |
| **Gate 7 — Return to menu & confirmation** | **PASS** | Plain `Reboot` -> `RESET` -> picker -> UUIDv4 attempt confirmed (`d3c665a6-0002-4525-8472-95bf10ff880d` virtio / `7d4d4cac-f6b5-43ab-8973-14c02ef0016c` VGA; `status: confirmed`, `consecutive_failures: 0`), `boot_success_consumed: true`. |
| **Gate 8 — Offline operation** | **PASS** | Zero `-netdev` across all positive and negative runs. |
| **Gate 9 — CI, integrity & resource hygiene** | **PASS** | All 7 product/tooling repos + `research` green on hosted CI; `inputs_unchanged: true`, `decoy_esp_unchanged: true`, `fixture_disks_unchanged: true`; all VMs admitted via `sbm-lab resources` with zero leaks. |
| **Neg. 1 — Tampered UKI** | **PASS** | 1-byte UKI flip -> `verify_done result=failed`, `linux_never_started`, accessible refusal screen captured (`07-tampered-uki-refusal.png`). *(Hosted KVM Secure Boot run `45980442284` also verified unsigned-recovery denial `load_failed status=ACCESS_DENIED`.)* |
| **Neg. 2 — Corrupted `dm-verity`** | **PASS** | Corrupted root block -> `uki_verified_before_mount`, `linux_started`, `verity_rejected_mount`, `ui_never_ready`, `boot_success_never_written`. |
| **Neg. 3 — Invalid handoff & stale confirmation** | **PASS** | `invalid-handoff`: `handoff_unverified`, `SBM_BOOT_SUCCESS_DECLINED`, `boot-success.json` absent. `stale-boot-success`: `attempt_confirm_rejected`, `attempt_not_confirmed_in_json`, stale report removed. |
| **Neg. 4 — Decoy ESP immutability** | **PASS** | Strict byte-for-byte SHA-256 unchanged across all positive runs and all 5 non-chainloading negative scenarios. |
| **Rec. 5 — Bound one-shot `LocalBootRequest`** | **PASS** | `local-boot-request-negative`: rejected, deleted before act, target not launched. `local-boot-request-positive`: `policy reason=core-request`, deleted before act, fixture launched (`fixture_started` + `chainload-ok.txt`), zero SBM state on decoy ESP. |
| **Tier 3 — Physical hardware (`EXP-HW-HP01`)** | **OPT-IN / NOT RUN** | Awaiting human-authorized physical USB boot test on the dedicated HP laptop (`EXP-HW-HP01`). |

## 4. Visually Verified Screenshots

All screenshots under [`screenshots/2026-11-27-sbm-0.0.1-qual/x86_64/`](screenshots/2026-11-27-sbm-0.0.1-qual/x86_64/) were converted from raw QMP PPM dumps and visually inspected:

- [`01-uefi-menu.png`](screenshots/2026-11-27-sbm-0.0.1-qual/x86_64/01-uefi-menu.png) — UEFI graphical picker (`virtio-vga`, `1280×800`, Atkinson Hyperlegible typography)
- [`02-core-ui.png`](screenshots/2026-11-27-sbm-0.0.1-qual/x86_64/02-core-ui.png) — Live `sbm-ui` (`backing-protected`, verified handoff, 4 detected disks)
- [`03-gparted.png`](screenshots/2026-11-27-sbm-0.0.1-qual/x86_64/03-gparted.png) — GParted launched over XWayland
- [`04-gparted-devices.png`](screenshots/2026-11-27-sbm-0.0.1-qual/x86_64/04-gparted-devices.png) — GParted device dropdown showing `/dev/sda`, `/dev/vda`, `/dev/vdb (1.00 GiB)`, `/dev/vdc (2.00 GiB)`
- [`05-post-reboot-menu.png`](screenshots/2026-11-27-sbm-0.0.1-qual/x86_64/05-post-reboot-menu.png) — Post-reboot UEFI picker after attempt confirmation
- [`06-vga-software-ui.png`](screenshots/2026-11-27-sbm-0.0.1-qual/x86_64/06-vga-software-ui.png) — `-device VGA` (`1024×768`) software fallback UI (`pixman + tiny-skia`)
- [`07-tampered-uki-refusal.png`](screenshots/2026-11-27-sbm-0.0.1-qual/x86_64/07-tampered-uki-refusal.png) — Accessible refusal screen on 1-byte tampered UKI
