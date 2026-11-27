# Native SBM simulator, 2026-11-26

Measurements and checks for the native development environment described in
[../../ecosystem/native-ui-preview-tooling-2026-11-26.md](../../ecosystem/native-ui-preview-tooling-2026-11-26.md).
Everything here is SIMULATED. It is evidence about the desktop tool, not about
firmware, QEMU or a Linux image.

- Host: MacBook Air M5, 16 GiB, macOS 27.0 (Darwin 27.0.0 arm64, 10 logical CPUs)
- Toolchain: rustc 1.100.0 (2026-11-09); dev profile, which `cargo xtask sim` uses
- Script: `sbm-uefi/sim/measure.py` (cold builds use fresh `CARGO_TARGET_DIR`s;
  crates were already downloaded, so network time is excluded). No `RUSTC_WRAPPER`/sccache.
- The dev profile optimizes `sbm-gfx` and the PNG encoder crates (`opt-level = 3`);
  all figures below were taken with that setting.

## Builds

| Build | Median | Min | Max | N |
| --- | --- | --- | --- | --- |
| Cold `sbm-preview --features window` | 4.2 s | 4.1 s | 4.2 s | 3 |
| Cold `sbm-sim` + `sbm-ui --features native-sim` (Iced) | 15.4 s | 15.3 s | 15.5 s | 3 |
| Warm, touch `sbm-preview/src/session.rs` | 0.32 s | 0.32 s | 0.56 s | 10 |
| Warm, touch `sbm-gfx/src/paint.rs` (renderer) | 0.39 s | 0.39 s | 0.41 s | 10 |
| Warm, touch `sbm-ui/src/main.rs` | 0.46 s | 0.45 s | 0.72 s | 10 |
| Warm, touch `sbm-runtime/src/ipc.rs` | 0.62 s | 0.61 s | 0.66 s | 10 |

An earlier N=10 warm pass on the same machine, taken while it was busier, had
medians of 0.57, 0.72, 0.82 and 1.02 s. Expect that kind of spread.

## UEFI menu window (`sbm-preview`, minifb)

| Metric | Median | Min | Max | N |
| --- | --- | --- | --- | --- |
| Process start to first presented frame (in-process clock) | 117.3 ms | 92.7 ms | 138.4 ms | 10 |
| Spawn to first frame reported (includes exec and dyld) | 129.6 ms | 101.1 ms | 148.1 ms | 10 |
| RSS, idle menu, no countdown (`windows-only`) | 88.6 MiB | 84.3 MiB | 93.0 MiB | 5 |
| CPU, idle menu, 8 s window after 3 s settle | 0.62 % | 0.62 % | 1.37 % | 5 |
| RSS, countdown running (`dual-boot`) | 92.8 MiB | 90.8 MiB | 93.2 MiB | 5 |
| CPU, countdown running, 4 s window | 1.25 % | 1.00 % | 2.25 % | 5 |
| Headless render + PNG, all 10 scenarios x screens x themes, 1280x800 | 29.4 ms | 28.5 ms | 33.0 ms | 64 |

CPU is one core = 100 %. Most of the RSS is AppKit/Metal for the window; the
framebuffer itself is 4 MiB at 1280x800. Before the redraw and profile fixes,
the countdown case measured a 7.25 % CPU median (N=5) and headless images took
about 175 ms each.

## Native Core (real `sbm-ui` + `sbm-sim`, `multi-disk` scenario)

| Metric | Median | Min | Max | N |
| --- | --- | --- | --- | --- |
| `sbm-sim` start to `UiReady` (`boot-success.json` written) | 244 ms | 234 ms | 251 ms | 5 |
| `sbm-ui` RSS | 105.0 MiB | 104.4 MiB | 105.4 MiB | 5 |
| `sbm-ui` idle CPU (8 s) | 0.13 % | 0.00 % | 0.50 % | 5 |
| `sbm-sim` RSS | 2.9 MiB | 2.9 MiB | 3.0 MiB | 5 |
| `sbm-sim` idle CPU (8 s) | 0.12 % | 0.12 % | 0.12 % | 5 |

## Functional checks (Verified)

- `sbm-uefi`: `cargo test` covers 44 tests, including 11 scenario tests: expected cards for
  all 10 scenarios, countdown boots the default, `failed_boots` falls back to
  SBM Core, the Secure Boot rejection error, corrupt loader and slot, the BitLocker
  note, firmware power path, deterministic renders, and the Core round trip
  booting the requested target once.
- `sbm-core`: `cargo test --workspace` covers 31 tests. 9 of them are `sbm-sim` contract tests
  against fixtures written by `sbm-preview --materialize`: the handoff verifies,
  BitLocker is protected, UiReady writes a success report, launches are never
  executed, BootTarget writes a one-shot command, unknown targets are rejected,
  and shutdown is recorded.
- Live: the real `sbm-ui` connected to `sbm-sim` and confirmed the boot, and closing the Core
  window ended `sbm-sim` with `power=reboot` (screenshot below).
- fmt and `clippy -D warnings` pass on host for both workspaces and on
  `aarch64-unknown-uefi` / `x86_64-unknown-uefi` for `sbm-app`.
- After the `sbm-menu` extraction, `cargo xtask qemu native` (aarch64, HVF) passed:
  first frame 3989 us, menu ready 14454 us, chainload and handoff written.
- Image isolation: `cargo check --locked` in `sbm-core` (default members, as
  `scripts/build-inside.sh` builds) compiles no `sbm-sim`, and `cargo tree -e features`
  shows `native-sim` disabled.

## Not verified (Experimental)

- The `minifb` window on Linux desktops; CI builds `sbm-preview` without the
  `window` feature.
- `--watch`, because `watchexec` is not installed on this Mac. Only the message for a missing `watchexec` was exercised.

## Screenshots

In `screenshots/native-sim/`: `uefi-dual-boot-menu-dark.png`,
`uefi-dual-boot-menu-800x600.png`, `uefi-multi-disk-menu-light.png`,
`uefi-existing-sbm-menu-dark.png` (fallback to Slot A after three failed boots),
`uefi-secure-boot-reject-error-dark.png`, `uefi-corrupt-boot-menu-light.png`,
`uefi-linux-only-diagnostics-dark.png`, `uefi-empty-uefi-menu-dark.png`,
`uefi-bitlocker-booted-dark.png` (simulated boot outcome), and
`core-multi-disk-home.png` (real `sbm-ui`, SIMULATED title and data).
