# Native UI preview tooling: build vs reuse (2026-11-26)

Decision record for the native SBM development environment (`sbm-uefi/sim`,
`sbm-uefi/crates/sbm-preview`, `sbm-uefi/crates/sbm-menu`,
`sbm-core/crates/sbm-sim`). It follows the Level 0 recommendation in
[dev-infrastructure-and-testing-landscape-2026-11-26.md](dev-infrastructure-and-testing-landscape-2026-11-26.md).
Measurements are in
[../validation/evidence/sbm-native-sim-2026-11-26.md](../validation/evidence/sbm-native-sim-2026-11-26.md).

## What was built, and why it is not a duplicate

| Piece | Reused | Written |
| --- | --- | --- |
| UEFI menu logic | `sbm-boot` discovery, policy, attempts, handoff; `sbm-gfx` `MenuState`, `step`, painter | `sbm-menu` (about 330 lines), moved out of `sbm-app/src/main.rs` so firmware and preview share one copy of input mapping, card order, diagnostics and failure text |
| Firmware services | `sbm_boot::MemoryVolume` (existing test double), `Volume` trait | `sbm-preview::machine`: a scenario becomes in-memory ESPs with synthetic PE headers and scripted `StartImage` results |
| Window | `minifb` 0.29 | Event loop with dirty-flag redraw (`sbm-preview/src/window.rs`, about 330 lines) |
| PNG | `png` 0.18 | Replaced the custom stored-deflate encoder in the old `xtask preview` (deleted) |
| Scenarios | `toml` 0.9 + `serde` | Schema and 10 files (`sim/scenarios`) |
| Core UI | Unmodified `sbm-ui` (Iced 0.14) | `native-sim` feature: socket path from `SBM_SIM_CORED_SOCKET`, SIMULATED title (about 15 lines) |
| Core daemon | `sbm-runtime` parsers (`parse_lsblk`, `infer_installed_os`, `parse_ip_addr`, `parse_meminfo`), `verify_handoff`, `validate_plan`, IPC framing | `sbm-sim` (about 450 lines): serves a session directory and records launches and power actions instead of executing them |
| Task runner | Agent 16's `sbm/justfile` and existing `xtask` | `xtask sim`, `xtask core`, delegating `xtask preview`; the justfile `preview`/`preview-core` recipes call them |

No generic hardware, firmware or VM emulator was written. The simulator stops at
the boundaries SBM already abstracts: the `Volume` trait on the UEFI side and the
`ClientRequest`/`ClientResponse` socket on the Core side.

## Window library evaluation

| Option | Licence | Verdict |
| --- | --- | --- |
| `minifb` 0.29 | MIT OR Apache-2.0 | **Chosen.** It takes a `&[u32]` 0RGB buffer, the same layout as `sbm_gfx::Framebuffer`, so there is no conversion. It has resize, `AspectRatioStretch` scaling, keyboard, mouse, cursor hiding and `set_title`. A 4 s cold build. |
| `softbuffer` + `winit` | Apache-2.0 / MIT | Works, but needs an `ApplicationHandler` event loop and a `Rc<Window>`/surface lifetime dance for the same result. Larger dependency tree. |
| `pixels` | MIT | Brings in `wgpu` and GPU scaling. Unnecessary for a software renderer, and much slower to build. |
| Iced / egui shell around the framebuffer | MIT | Would put a second UI toolkit around the firmware renderer, so not a faithful preview. |

## Hot reload and file watching

| Option | Licence | Verdict |
| --- | --- | --- |
| `watchexec` | Apache-2.0 | **Used if installed** (`cargo xtask sim --watch`), otherwise it prints install instructions. It is not installed on this Mac. |
| `cargo-watch` | CC0 | The maintainers point users to `watchexec` and `bacon`, so it is not adopted. |
| `bacon` | AGPL-3.0 | Usable as a separate developer tool. It is never linked, but we don't depend on it. |
| Custom in-process reloader | n/a | Not written. A warm rebuild plus relaunch takes under a second (see evidence), so a reloader isn't needed. |

## Production isolation

- `sbm-app` depends on `sbm-menu` (no_std, no new crates). `sbm-preview` is a
  host-only binary crate. The UEFI release build (`-p sbm-app -p sbm-fixture`)
  never compiles it.
- `sbm-sim` is outside `sbm-core`'s `default-members`. `scripts/build-inside.sh`
  runs `cargo build --release --locked` without `-p`, so image builds never
  compile it, and `cargo tree -e features` shows `native-sim` off.
- `native-sim` changes no runtime behaviour except where the shell connects,
  and only when `SBM_SIM_CORED_SOCKET` is set.

## Discovered production issue (not fixed here)

Core's **Operating systems → Request this target on next boot** sends
`InstalledOs.id` = `os:<partuuid>`. The UEFI menu only knows
`efi:<partuuid>:<hash>` and `sbm-core:*` ids. If that id passes `sbm-cored`'s
`known_target` check, the UEFI resolver highlights an unknown id, falls back to
card 0, and boots it immediately because the countdown is zero. The simulator
reproduces this faithfully. Mapping an `InstalledOs` to the UEFI target id
needs an owner decision: either carry `discovered_targets` from the handoff in
the Core UI, or reject `os:` ids for `BootTarget`.

## Status

- Verified: everything listed in the evidence file (tests, screenshots, QEMU
  smoke after the `sbm-menu` extraction, measured timings) on macOS 27 / M5.
- Experimental: the Linux desktop window build (`minifb` needs X11 or Wayland
  development libraries) was not exercised; CI runs the headless build only.
