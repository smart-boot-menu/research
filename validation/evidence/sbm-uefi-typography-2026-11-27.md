# UEFI Accessible Vector Typography & Unified Rasterizer — Measured Evidence (2026-11-27)

**Scope:** Before/after measurements for Agent 20's resolution-independent vector typography (`ttf-parser` 0.25.1 + `ab_glyph_rasterizer` 0.1.10 + `fontTools`-optimized Atkinson Hyperlegible Next & Mono) on `sbm-uefi` (`branch agent20/accessible-typography`, base `e631fbd`) and `sbm-core` (`sbm-ui`).

**Host:** Apple Silicon MacBook Air M5, macOS, `rustc 1.100.0`, QEMU 11.1.3 (`aarch64` HVF with `edk2-stable202408-prebuilt.qemu.org`).

> **Labeling:** Microbenchmarks (`bench_typography`) and `sbm-preview` numbers are native macOS host measurements. The QEMU `aarch64` table is real guest-timer UEFI firmware execution under HVF (`cntpct_el0` / `cntfrq_el0` at 24 MHz), not bare-metal PC hardware.

## 1. Release UEFI Binary Size & Bundled Font Budget

Release profile: `opt-level = "z"`, `lto = true`, `codegen-units = 1`, `panic = "abort"`, `strip = true`, verified with SBAT (`sbat,1,`) and `IMAGE_DLLCHARACTERISTICS_NX_COMPAT`.

| Metric | Baseline (`e631fbd`, 8×8 bitmap) | After (Vector + Atkinson Next & Mono) | Delta | % of 1 MiB Target |
| --- | ---: | ---: | ---: | ---: |
| `aarch64-unknown-uefi` `sbm.efi` | 207,360 B (202.5 KiB) | **392,704 B (383.5 KiB)** | +185,344 B (+181.0 KiB) | **37.5%** |
| `x86_64-unknown-uefi` `sbm.efi` | 231,936 B (226.5 KiB) | **439,808 B (429.5 KiB)** | +207,872 B (+203.0 KiB) | **41.9%** |
| Bundled vector font data (3 TTFs) | 0 B (2,048 B 8×8 bitmap) | **101,836 B (99.4 KiB)** | +101,836 B (44.6% smaller than 183,768 B raw upstream) | 9.7% |
| Vector + typography engine code (`ttf-parser`, `ab_glyph_rasterizer`, `libm`, `vector.rs`, `typography.rs`) | 0 B | ~83.5 KiB (`aarch64`) / ~106.0 KiB (`x86_64`) | — | ~8–10% |

Both architectures remain well under half of the aspirational 1 MiB EFI target.

## 2. Real UEFI Firmware Smoke Test (QEMU OVMF `aarch64` HVF)

Command: `SBM_LAB_BIN=.../sbm-lab cargo run -p xtask -- qemu native` (1024×768 Bgr GOP, 3,145,728 B backbuffer, 6 boot targets, full chainload + handoff acceptance).

| Guest Metric (from `SBM_MARK uefi_entry`) | Baseline (`docs/MEASUREMENTS.md`) | After (Vector + Atkinson Next & Mono) | Delta |
| --- | ---: | ---: | ---: |
| `first_frame` (splash rendered to GOP) | 5,347 µs | **5,788 µs** | +441 µs (+0.44 ms) |
| `discovery_done` (6 targets) | 14,286 µs | **13,504 µs** | −782 µs |
| `menu_ready` (full 6-card menu painted) | 21,765 µs | **22,793 µs** | +1,028 µs (+1.03 ms; **11× under 250 ms goal**) |
| `EfiConventionalMemory` drop (`delta` pages × 4 KiB) | 32 pages (128 KiB) | **31 pages (124 KiB)** | −1 page (no net increase) |
| Temporary glyph cache after splash + menu | N/A | **110 entries, 12,861 B (12.6 KiB)** | 237 hits / 110 misses (68.3% first-frame hit rate, 0 evictions) |
| Acceptance (`fixture_started`, `chainload_returned`, `handoff`, `chainload_ok`) | Pass (2/2) | **Pass (2/2)** | Full parity |

## 3. Host Microbenchmarks (`cargo run -p sbm-gfx --release --example bench_typography`)

### 3a. Cold First-Frame Full-Menu Render (Median of 11 Cold Runs, No Pre-Warming)

| Resolution | Legacy `Bitmap8x8` | `AtkinsonNext` | `AtkinsonNextMono` | First-Frame Cache |
| --- | ---: | ---: | ---: | --- |
| `800×600` | 1,761.9 µs | 2,206.5 µs | **2,181.9 µs** | 79 entries / 6,218 B (6.1 KiB) |
| `1280×800` | 2,381.7 µs | 2,728.2 µs | **2,713.6 µs** | 79 entries / 7,727 B (7.5 KiB) |
| `1920×1080` | 3,976.2 µs | 3,992.5 µs | **3,979.2 µs** | 79 entries / 10,492 B (10.2 KiB) |
| `3840×2160` (4K) | 11,756.5 µs | 10,179.5 µs | **10,142.9 µs** | 79 entries / 24,713 B (24.1 KiB) |

At 4K (`3840×2160`), the unified vector + Atkinson renderer is **13.7% faster** than the legacy 8×8 pixel renderer while producing crisp antialiased typography and emblems.

### 3b. Cold Splash First-Frame Render (Median of 11 Cold Runs)

| Resolution | Legacy `Bitmap8x8` | `AtkinsonNextMono` | Delta |
| --- | ---: | ---: | ---: |
| `1024×768` | 380.1 µs | **505.1 µs** | +125.0 µs |
| `1280×800` | 486.8 µs | **610.9 µs** | +124.1 µs |
| `1920×1080` | 985.1 µs | **1,101.5 µs** | +116.4 µs |

### 3c. Repeated Full-Menu Render (50 Interactive Selection Redraws): Uncached vs. Bounded LRU Cache

Evaluating whether the small, temporary, in-memory glyph cache (cap: 192 entries / 64 KiB, zero disk persistence) improves efficiency:

| Resolution | Uncached (`set_cache_enabled(false)`) | Bounded LRU Cache (`set_cache_enabled(true)`) | Speedup | Peak Cache Footprint |
| --- | ---: | ---: | ---: | --- |
| `1024×768` | 3,010.0 µs/frame | **2,195.6 µs/frame** | **1.37×** | 79 entries / 7,727 B (0 evictions) |
| `1280×800` | 3,301.0 µs/frame | **2,505.2 µs/frame** | **1.32×** | 79 entries / 7,727 B (0 evictions) |
| `1920×1080` | 4,623.1 µs/frame | **3,763.1 µs/frame** | **1.23×** | 79 entries / 10,492 B (0 evictions) |

### 3d. Isolated Text-Only Rendering (100 Passes × 6 Mixed-Role Lines = 243 Glyphs/Pass at 1280×800)

| Renderer Configuration | Time / Pass | Speedup vs. Uncached | Cache Footprint |
| --- | ---: | ---: | --- |
| Legacy `Bitmap8x8` | 158.4 µs | — | 0 B |
| `AtkinsonNextMono` (uncached) | 972.9 µs | 1.00× | 0 B |
| `AtkinsonNextMono` (bounded LRU cache) | **321.1 µs** | **3.03×** | 114 entries / 13,464 B (13.1 KiB) |

The temporary bounded LRU cache cuts pure text-rendering time by **67% (3.03× faster)** using only 13.1 KiB of RAM (20% of its 64 KiB cap), clearly justifying its retention.

## 4. Native Simulator Startup & Incremental Rebuild (`sbm-preview --features window`)

| Metric | Measured Result |
| --- | --- |
| Cold window launch to first frame (`--scenario multi-disk --font next-mono --exit-after-ms 1`) | 645.3 ms first frame (668.5 ms total) |
| Warm window launch to first frame (runs 2 & 3) | **151.2–153.2 ms** first frame (173.5 ms median total) |
| Incremental rebuild after editing `crates/sbm-gfx/src/typography.rs` | **0.76 s** wall time (`cargo` 0.72 s) |

## 5. Automated Test & Visual Comparison Matrix

- `sbm-uefi`: **61 tests passed, 0 failed** (`sbm-boot` 18, `sbm-gfx` 19 incl. GPOS kerning, `.notdef` honesty, language-pack fallback, bounded cache cap, wrapped `ortho_polygon` AA, dark/light/high-contrast contrast ratios, `sbm-menu` 3, `sbm-preview` 16, `xtask` 5); `cargo clippy --all-targets -- -D warnings` clean; `aarch64-unknown-uefi` & `x86_64-unknown-uefi` release builds clean.
- `sbm-core`: **37 tests passed, 0 failed** (`sbm-runtime` 23, `sbm-sim` 9, `sbm-ui` 5 verifying embedded Atkinson TTFs, 5-role monotonicity across scales 85%–200%, and all 4 appearance themes); `cargo clippy --workspace --all-targets -- -D warnings` clean; `sbm-ui --features native-sim` builds cleanly.
- Identical-scene comparison screenshots generated and visually verified in `sbm-uefi`:
  - `multi-disk` menu across `bitmap`, `next`, and `next-mono` in Dark & Light at `800×600`, `1280×800`, `1920×1080`, and `3840×2160` (24 PNGs).
  - `diagnostics` screen (`bitmap`, `next`, `next-mono` dark/light), `dual-boot` high-contrast dark/light, `secure-boot-reject` error, and `splash` (10 PNGs).
  - Real UEFI OVMF `aarch64` screenshots (`screenshots/aarch64-menu-dark.png`, `screenshots/aarch64-menu-light.png`).
