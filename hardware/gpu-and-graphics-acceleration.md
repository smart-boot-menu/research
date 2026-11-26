# GPUs, Hardware Acceleration & GUI Frameworks (UEFI vs Linux)

**Last verified:** 2026-11-26  
**Scope:** Evaluating graphics hardware realities across UEFI and Linux, disambiguating `iGPU` hardware from GUI libraries, verifying `uefi-wm`, and recommending a concrete UI/rendering strategy for `sbm-uefi` and `sbm-core`.

---

## 1. Fundamental Disambiguation: Integrated GPUs (`iGPU`) vs GUI Frameworks

1. **No Rust GUI framework named `iGPU` / `igpu` exists** as of November 2026. Searches across `crates.io` and GitHub show references to `iGPU` refer to **integrated GPU hardware** (e.g., Intel UHD/Arc Xe, AMD Radeon Vega/RDNA iGPUs, Apple Silicon AGX unified GPUs) or hardware-enumeration utilities (like `scry-gpu`).
2. **No exact crate/repo named `uefi-wm` exists** in indexed public repositories as of November 2026. The closest active Rust UEFI UI projects are:
   - `sloev/uefi-ui` (`no_std` + `alloc` immediate-mode UI prototype),
   - **`slint-ui/slint` official `examples/uefi-demo`** (`x86_64-unknown-uefi` software renderer example).
3. **Critical Architectural Fact — UEFI Does NOT Expose GPU Hardware Acceleration:**
   - Even if a computer has an Intel Arc iGPU, AMD RDNA iGPU, NVIDIA RTX discrete GPU, or Apple M5 GPU, **UEFI firmware ONLY exposes `EFI_GRAPHICS_OUTPUT_PROTOCOL` (GOP)** — a flat, linear CPU-writable RAM framebuffer (`Blt` / direct pixel writes).
   - There is **no** Vulkan, OpenGL, Metal, DirectX, DRM/KMS, or shader compiler in UEFI.
   - Therefore, **all rendering in `sbm-uefi` MUST be fast, clean CPU software rendering** to a double-buffered RAM backbuffer copied via a single `Blt` or linear framebuffer copy per dirty region.

---

## 2. Rust GUI & Rendering Library Evaluation (November 2026)

| Framework / Library | Latest Verified Version (Nov 2026) | License (SPDX) | `no_std` + `alloc` (UEFI) | Linux GPU Accel | Binary Size / Complexity | Verdict |
|---|---|---|---|---|---|---|
| **Slint** | `1.19.1` (Nov 2026; MSRV 1.95, Rust 2024) | **`GPL-3.0-only` OR Commercial/Royalty-Free** | **Yes!** Official `examples/uefi-demo` + `renderer-software` + `software-renderer-path` | **Yes** (Skia/FemtoVG/wgpu + Wayland/KMS) | Medium (~1–3 MB in UEFI) | **Caution / Licensing Blocker** (GPL-3.0 conflicts with SBM's `MIT OR Apache-2.0` default!) |
| **Custom Software Renderer (`tiny-skia` / `embedded-graphics` / `fontdue` or `ab_glyph`)** | `tiny-skia` `0.11+`, `embedded-graphics` `0.8+`, `ab_glyph` `0.2+` | **`MIT OR Apache-2.0` / `BSD-3-Clause`** | **Yes!** Clean `#![no_std]` + `alloc` | N/A (CPU only) | **Tiny (~150–400 KB)** | **Recommended for `sbm-uefi`** |
| **`vello_cpu` / `vello_gpu`** (Linebender) | `vello_cpu 0.4.0`, `vello_gpu 0.4.0` (Nov 2026; renamed from `vello_hybrid`) | `MIT OR Apache-2.0` | `vello_cpu` tested `no_std` + `alloc` | `vello_gpu` via `wgpu` | Medium-High (fast-moving `0.4` API) | **Watch / Optional Future Upgrade** |
| **Iced** | `0.13+` / `0.14` | `MIT` | **No** (requires `std`, `winit`/`wgpu` or `tiny-skia`) | **Yes** (`wgpu` Vulkan/GL + `tiny-skia` CPU fallback!) | Medium | **Recommended for `sbm-core` Native Rust GUI** |
| **egui** | `0.31+` / `0.33` | `MIT OR Apache-2.0` | Core layout is `no_std`, full renderer expects `std`/GL/`wgpu` | **Yes** (Glow/wgpu) | Low-Medium | **Viable alternative for `sbm-core`** (more developer-tool aesthetic) |
| **wgpu** | `24+` / `28+` | `MIT OR Apache-2.0` | **No** (requires OS Vulkan/GL/Metal) | **Yes** (Vulkan, OpenGL ES, software `llvmpipe`) | Heavy | **Used indirectly via Iced in `sbm-core`** |

### Why Slint Is NOT Selected as a Forced Common Framework
Although **Slint `1.19.1`** technically supports both `no_std` UEFI (`examples/uefi-demo`) and Linux GPU rendering:
1. **License incompatibility:** Slint's open-source license is **`GPL-3.0-only`** (its royalty-free commercial license requires attribution/royalty-free agreement terms incompatible with pure `MIT OR Apache-2.0` permissive open-source redistribution). Using Slint in `sbm-uefi` would force `sbm-uefi` distributions under GPLv3 obligations.
2. **UEFI binary size & boot latency:** A purpose-built `#![no_std]` software renderer in `sbm-uefi` (damage-tracked double buffer + vector/rounded-rect primitives + pre-rasterized or `ab_glyph`/`fontdue` bitmap fonts + crisp SVG-baked PNG/QOI icons) produces a **< 1 MB EFI binary** with **< 50 ms first-frame paint** and zero external build-time UI DSL complexity.
3. **Different UX complexity:** `sbm-uefi` needs an ultra-fast, keyboard/mouse/touch-friendly, deterministic boot picker (10–20 UI primitives). `sbm-core` needs a full multi-window desktop recovery workspace capable of launching **GTK3 GParted**, terminals, disk topology trees, OS catalog browsers, and remote VNC streams.

---

## 3. Recommended Strategy: Shared Design System, Separate Renderers (ADR-002)

- **Shared across UEFI & Linux (`sbm-common`):** Theme tokens (colors, spacing, typography scales, iconography metadata), boot-entry view models, status badges, and localized UI state schemas.
- **`sbm-uefi` Renderer (Pre-OS Fast Path):**
  - Pure-Rust `#![no_std]` + `alloc` retained/immediate lightweight 2D software renderer directly on UEFI GOP (`uefi::proto::console::gop::GraphicsOutput`).
  - Uses integer-scaled HiDPI detection (e.g., auto 2× scaling when GOP horizontal resolution ≥ 2200 px, such as MacBook Retina or 4K displays).
  - Double-buffered dirty-rectangle compositing to prevent flicker on slow GOP linear framebuffers (including older x86 HP laptops and U-Boot `VIDEO_SIMPLE` on Apple Silicon).
  - Graceful fallback to UEFI text console if GOP is unavailable (headless serial/QEMU `-nographic` testing!).
- **`sbm-core` Graphics Stack (Linux Management Path):**
  - **Kernel & Drivers:** Linux **DRM/KMS** (`i915`/`xe` for Intel iGPUs, `amdgpu`/`radeon` for AMD iGPUs/dGPUs, `nouveau`/`nova` + `simpledrm` for NVIDIA, `asahi`/`apple` DRM + `simpledrm` for Apple Silicon).
  - **Userspace Acceleration + Guaranteed Software Fallback:** **Mesa** (Vulkan/OpenGL for Intel/AMD/Apple M1–M3) with automatic **`llvmpipe` / `pixman`** CPU rendering fallback whenever GPU drivers are unavailable (e.g., Apple M4/M5 `simpledrm`, new NVIDIA GPUs, virtual `virtio-gpu`/`bochs-drm`, or older low-RAM HP laptops).
  - **Compositor:** **`labwc`** (lightweight `wlroots`-based Wayland stacking compositor, GPL-2.0 bundled in Linux image, tiny RAM footprint ~15–25 MB) with **XWayland enabled** (specifically required so **GTK3 GParted 1.8.1** runs cleanly!) and **`wayvnc`** for browser-native remote desktop (`noVNC`).
  - **SBM Management UI (`sbm-core`):** Built in **Iced** (`MIT`, Rust) using `wgpu` when hardware acceleration is present and `tiny-skia` software rendering automatically on `simpledrm`/older hardware — OR a local web UI served by `sbm-core` if browser reuse is preferred; **Iced** avoids shipping a 300 MB Chromium/WebKit engine inside a `< 700 MB` recovery image!
