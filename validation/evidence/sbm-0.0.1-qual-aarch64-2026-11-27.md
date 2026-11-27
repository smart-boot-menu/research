# SBM 0.0.1 Qualification Evidence — `aarch64` (2026-11-27)

- **Release:** `0.0.1` (`aarch64` single-architecture image; `dualarch` remains optional/experimental and non-gating)
- **Trust Tier:** **Development / Preview** (`production_secure_boot_claimed: false`, `production_tuf_signed: false`; explicitly distinguished from future SBM 0.2+ consumer Secure Boot releases)
- **Qualification Verdict:** **TIER 2 QUALIFIED (Development Preview)** — All mandatory Gates 0–9, all 4 mandatory negative security gates, and both bound `LocalBootRequest` scenarios pass on the final immutable, reproducible media. Tier 3 physical hardware remains opt-in / unverified.
- **Machine-Readable Bundle:** [`sbm-0.0.1-qual-aarch64-2026-11-27.json`](sbm-0.0.1-qual-aarch64-2026-11-27.json)

## 1. Frozen Immutable Source Revisions & Hosted CI

| Repository | Frozen Commit | Hosted CI Run | Status |
|---|---|---|---|
| `sbm-common` | `abe3029c831f05d77d10cfb214a0c236903d00d9` | `45958218582` | `success` |
| `sbm-uefi` | `4bfb3ba6457a4f6625d13c6a1435f69f15b8d011` | `45980442284` | `success` |
| `sbm-core` | `960f11a5e748e30f105cd79cc648f0633af63138` | `45977817124` | `success` |
| `install` | `870069dccd2b7fc680a35955c27a6481bb46a0e4` | `46014761860` | `success` |
| `sbm-lab` | `5ab4f8c0485a226fc639778a65a8a1467c352bde` | `46059331789` | `success` |
| `sbm-as` | `759be4eabed0cb760eabe5d8befcbba43fe067c6` | `45977832267` | `success` |
| `sbm-server` | `ae18f7832faae6a8b2ad7de709c0ebc733a73531` | `45940770054` | `success` |

## 2. Final Artifact Inventory, Sizes & Compressed Reproducibility (Gate 0)

Built cleanly with `SBM_BUILD_PROFILE=release` (`Incremental=no`, zero `sbm-smoke` files/presets verified inside mounted read-only EROFS root), locked via `install/media/0.0.1.lock.toml` (SHA-256 `a88928d218afc2785dfecf0765b4cf7f3b8920948de0d1accb13be3a92eed213`), verified by `sbm-install media verify`, and proven byte-identical across two independent clean builds with zstd compression enabled (`check-reproducible.sh`: `reproducible: true`, `compressed: true`, 6/6 output files identical).

| Artifact | Size | SHA-256 |
|---|---:|---|
| `sbm-aarch64.efi` | `410,112 B` (`400.5 KiB`) | `77fac1c2d3ad8e39f1f4932c99790b68e92cd0fed8ce28ee7efbeaeada244261` |
| `sbm-core-arm64.efi` (UKI) | `74,513,752 B` (`71.06 MiB`) | `646918d13ef62b994fd2ca7970295e709840a9cd1c87776a84e2e592bbfe07f0` |
| `sbm-core-arm64.raw` (Core input) | `1,363,620,352 B` (`1,300.4 MiB`) | `4f4aa59380561a8d47c3536b8f3fd03d39508dada69f29268312c02bb500f82b` |
| `sbm-0.0.1-uefi-aarch64.raw` | `1,097,859,072 B` (`1,047.0 MiB`) | `592d4ffec86fe8d4891f3ff3b09397dad5a358cd438baf8b8f852f627b45fdfc` |
| `sbm-0.0.1-uefi-aarch64.raw.zst` | `860,581,891 B` (`820.7 MiB`) | `56639af5a3e8d03d067b566cfd36495c424a8f352c26ca099ab1c0271203be24` |
| `release-manifest.v1.json` | `1,307 B` | `75d04bd5898aa15a970069e4cbfb3fc7440abbc2e88cc208613bfe762217efc8` |
| `build-receipt.json` | `8,158 B` | `935e0b4c9a27fc50d814a2faa8d783ed4d43a52a36efe5d7359838f02fa9aa26` |
| `dm-verity` `roothash` | — | `144daac8dcae00067920498dd302321cf4c681699563617b0e655b25eb1517bb` |

## 3. Gate Scorecard (`aarch64` — Native HVF / AAVMF)

| Gate | Result | Measured Evidence |
|---|---|---|
| **Gate 0 — Immutable reproducible build** | **PASS** | `repro-report.json`: `reproducible: true`, `compressed: true` (6/6 files identical); `sbm-install media verify` green; `production_secure_boot_claimed: false`, `production_tuf_signed: false`, `experimental: false`. |
| **Gate 1 — Real UEFI menu & startup speed (`N=10`)** | **PASS** | `N=10` cold boots on real `sbm-0.0.1-uefi-aarch64.raw` (HVF, `1280×800`, fresh QCOW2 + `vars.fd`, offline): `menu_ready` min `8.36 ms`, **median `15.83 ms`**, p95/max `17.69 ms` (`<< 250 ms`), `first_frame` median `7.19 ms`, glyph cache `10,066 B`. |
| **Gate 2 — Discovered targets** | **PASS** | Picker rendered both `Fedora Linux (Decoy Fixture)` and `SBM Core Slot A` (visually verified). |
| **Gate 3 — UKI verification & Spec 03 handoff** | **PASS** | Streaming 1 MiB SHA-256 `verify_done`: `368.06 ms` (`virtio`) / `194.90 ms` (`ramfb`), `result=ok`, all 5 volatile EFI vars + exact boot ESP UUID + UUIDv4 `attempt_id` + empty load options. |
| **Gate 4 — Real Linux Core (`dm-verity` EROFS, zero smoke)** | **PASS** | Booted Fedora 45 kernel (`7.3.4-300.fc45.aarch64`), resolved boot ESP via BLI despite decoy ESP first, mounted `/dev/mapper/root`, `SBM_PROTECT backing-protected` on boot & GParted exit, zero `SBM_SMOKE` units/markers. |
| **Gate 5 — Live Management UI & software fallback** | **PASS** | Passed 9/9 on both `virtio-gpu-pci` (`1280×800`, 59.4 s) AND `ramfb` (`1024×768`, 49.0 s; `Renderer: software (no DRM render node: pixman + tiny-skia)`). |
| **Gate 6 — GParted & virtual disks** | **PASS** | Real GParted via XWayland (`gparted_visual_delta`: `0.4393` virtio, `0.5769` ramfb); device dropdown displayed `/dev/sda`, `/dev/vda`, `/dev/vdb (1.00 GiB)`, `/dev/vdc (2.00 GiB)`; backing RO refcounted safely; `fixture_disks_unchanged: true`. |
| **Gate 7 — Return to menu & confirmation** | **PASS** | Plain `Reboot` -> `RESET` -> picker (`25.98 ms` virtio / `19.99 ms` ramfb) -> UUIDv4 attempt confirmed (`status: confirmed`, `consecutive_failures: 0`), `boot_success_consumed: true`. |
| **Gate 8 — Offline operation** | **PASS** | Zero `-netdev` across all positive, benchmark, and negative runs. |
| **Gate 9 — CI, integrity & resource hygiene** | **PASS** | All 7 product/tooling repos + `research` green on hosted CI; `inputs_unchanged: true`, `decoy_esp_unchanged: true`, `fixture_disks_unchanged: true`; all VMs admitted via `sbm-lab resources` with zero leaks. |
| **Neg. 1 — Tampered UKI** | **PASS** | 1-byte UKI flip -> `verify_done result=failed`, `linux_never_started`, accessible refusal screen captured (`07-tampered-uki-refusal.png`). |
| **Neg. 2 — Corrupted `dm-verity`** | **PASS** | Corrupted root block -> `uki_verified_before_mount`, `linux_started`, `verity_rejected_mount`, `ui_never_ready`, `boot_success_never_written`. |
| **Neg. 3 — Invalid handoff & stale confirmation** | **PASS** | `invalid-handoff`: `handoff_unverified`, `SBM_BOOT_SUCCESS_DECLINED`, `boot-success.json` absent. `stale-boot-success`: `attempt_confirm_rejected`, `attempt_not_confirmed_in_json`, stale report removed. |
| **Neg. 4 — Decoy ESP immutability** | **PASS** | Strict byte-for-byte SHA-256 unchanged across all positive runs and all 5 non-chainloading negative scenarios. |
| **Rec. 5 — Bound one-shot `LocalBootRequest`** | **PASS** | `local-boot-request-negative`: rejected, deleted before act, target not launched. `local-boot-request-positive`: `policy reason=core-request`, deleted before act, fixture launched (`fixture_started` + `chainload-ok.txt`), zero SBM state on decoy ESP. |
| **Tier 3 — Physical hardware** | **OPT-IN / NOT RUN** | Not yet executed on physical ARM64 hardware; not required for Tier 2 development-preview qualification. |

## 4. Visually Verified Screenshots

All screenshots under [`screenshots/2026-11-27-sbm-0.0.1-qual/aarch64/`](screenshots/2026-11-27-sbm-0.0.1-qual/aarch64/) were converted from raw QMP PPM dumps and visually inspected:

- [`01-uefi-menu.png`](screenshots/2026-11-27-sbm-0.0.1-qual/aarch64/01-uefi-menu.png) — UEFI graphical picker (`virtio-gpu`, `1280×800`, Atkinson Hyperlegible typography)
- [`02-core-ui.png`](screenshots/2026-11-27-sbm-0.0.1-qual/aarch64/02-core-ui.png) — Live `sbm-ui` (`backing-protected`, verified handoff, 4 detected disks)
- [`03-gparted.png`](screenshots/2026-11-27-sbm-0.0.1-qual/aarch64/03-gparted.png) — GParted launched over XWayland
- [`04-gparted-devices.png`](screenshots/2026-11-27-sbm-0.0.1-qual/aarch64/04-gparted-devices.png) — GParted device dropdown showing `/dev/sda`, `/dev/vda`, `/dev/vdb (1.00 GiB)`, `/dev/vdc (2.00 GiB)`
- [`05-post-reboot-menu.png`](screenshots/2026-11-27-sbm-0.0.1-qual/aarch64/05-post-reboot-menu.png) — Post-reboot UEFI picker after attempt confirmation
- [`06-ramfb-software-ui.png`](screenshots/2026-11-27-sbm-0.0.1-qual/aarch64/06-ramfb-software-ui.png) — `ramfb` (`1024×768`) software fallback UI (`pixman + tiny-skia`)
- [`07-tampered-uki-refusal.png`](screenshots/2026-11-27-sbm-0.0.1-qual/aarch64/07-tampered-uki-refusal.png) — Accessible refusal screen on 1-byte tampered UKI
