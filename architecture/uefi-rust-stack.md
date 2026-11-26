# Rust UEFI Environment Research (`sbm-uefi`)

**Last verified:** 2026-11-26  
**Primary Sources:** `rust-osdev/uefi-rs`, UEFI Specification 2.10/2.11, `google/crdyboot`, EDK II / OVMF docs.

---

## 1. Compiler, Edition & Targets (November 2026)

- **Rust Toolchain:** Rust **1.100.0** (released Nov 2026) with **Rust 2024 Edition** (`edition = "2024"`).
- **Compilation Targets:**
  - **`x86_64-unknown-uefi`** — Tier 2 target (precompiled `core` & `alloc` via `rustup target add x86_64-unknown-uefi`). Primary PC/laptop target.
  - **`aarch64-unknown-uefi`** — Tier 2 target (precompiled `core` & `alloc` via `rustup target add aarch64-unknown-uefi`). Primary ARM64 UEFI & Apple Silicon (via Asahi U-Boot) target.
  - **`i686-unknown-uefi`** — Tier 2 target for legacy 32-bit IA32 UEFI tablets/Bay Trail/older laptops where practical (best-effort CI check, secondary priority to x86-64 and ARM64).
- **Runtime Model:** `#![no_std]` + `extern crate alloc;` backed by `uefi::allocator::Allocator` (`BootServices` `AllocatePool`/`FreePool`).
- **Workspace Split Requirement:** All crates in `sbm-common` consumed by `sbm-uefi` (`sbm-types`, `sbm-protocol`) MUST support `default-features = false` (`#![no_std]` + `alloc`), verified continuously in CI against both `x86_64-unknown-uefi` and `aarch64-unknown-uefi`.

---

## 2. `uefi-rs` (`uefi` v0.41.0 / `uefi-raw` v0.17.0)

- **Latest Verified Release:** `uefi` **`0.41.0`**, `uefi-raw` **`0.17.0`** (September 2026; MSRV Rust 1.91.0; `MPL-2.0`).
- **Modern Global API (Post-`0.32`):** Legacy `SystemTable` / `BootServices` / `RuntimeServices` struct passing was deprecated in `0.32` and removed in subsequent versions. Modern `sbm-uefi` code MUST use:
  - `#[entry] fn main() -> Status`
  - `uefi::helpers::init()` (allocator, logger, panic hooks)
  - `uefi::boot::*` (`locate_handle_buffer`, `open_protocol_exclusive`, `load_image`, `start_image`, `stall`, `create_event`)
  - `uefi::system::*` and `uefi::runtime::*`
- **Testability Pattern (from `crdyboot`):** Abstract UEFI calls behind narrow internal Rust traits (e.g., `FirmwareStorage`, `Framebuffer`, `InputSource`, `NetworkTransport`) so 90% of `sbm-uefi` logic (discovery heuristics, policy resolution, layout, config parsing) runs as fast host `cargo test` unit tests without QEMU!

---

## 3. UEFI Protocols & Real-World Firmware Quirks

### Graphics (`EFI_GRAPHICS_OUTPUT_PROTOCOL`)
- Always query all supported modes, prefer the firmware's native/current mode (avoids black screens on laptop panels and Apple `VIDEO_SIMPLE`), and handle both `PixelFormat::Rgb` and `PixelFormat::Bgr` (and `Bitmask` fallback via `Blt`).
- Never assume `stride == horizontal_resolution`; always compute offsets using `info.stride()`.

### Filesystems (`EFI_SIMPLE_FILE_SYSTEM_PROTOCOL`)
- Standard UEFI firmware guarantees **FAT12/FAT16/FAT32** (the EFI System Partition).
- Optional read-only UEFI filesystem drivers (`efifs` — GPLv3/BSD drivers for `ext4`, `btrfs`, `xfs`, `ntfs`) can be loaded optionally by users via `ConnectController`, but **`sbm-uefi` MUST NOT depend on third-party EFI filesystem drivers for core operation**: all SBM config, BLS entries, UKIs, and `sbm-core` boot artifacts live on standard FAT32 ESP / XBOOTLDR partitions.

### Input (`SimpleTextInputEx`, `SimplePointer`, `AbsolutePointer`)
Real-world AMI, Insyde, Phoenix, and U-Boot firmwares exhibit notorious input bugs:
- **`WaitForKeyEx` hangs** on several laptops and composite USB HID devices -> **Mitigation:** Always drive the main loop with a bounded timer tick (`TimerRelative`, e.g., 16–33 ms) + non-blocking key/pointer polling rather than blocking indefinitely on `WaitForKeyEx`.
- **Unresponsive I2C trackpads:** Many laptops do not expose I2C-HID precision touchpads to UEFI (`SimplePointer` returns nothing or stays absent) -> **Mitigation:** Keyboard navigation MUST be a first-class, complete experience; mouse/touch is an additive enhancement.
- **Unscaled/Inverted `SimplePointer` deltas & buggy `AbsolutePointer`:** Clamp deltas, normalize using `mode.resolution`, and ignore spurious zero-resolution devices.

### Networking (`EFI_HTTP_PROTOCOL` / `EFI_TLS_PROTOCOL` vs `smoltcp` + TLS over SNP)

| Dimension | Native Firmware (`EFI_HTTP_PROTOCOL` + `EFI_TLS_PROTOCOL`) | Rust Stack (`smoltcp` + `embedded-tls`/`rustls` over SNP) |
|---|---|---|
| **Binary Size** | Tiny (~10–20 KB wrapper) | Moderate (~200–450 KB) |
| **Firmware Support** | Present on modern EDK2/OVMF & newer business PCs; **often missing** on older laptops & U-Boot! | **Works anywhere SNP exists** (EDK2, older laptops, U-Boot, PXE NICs) |
| **TLS Root CA & Security** | Relies on opaque firmware CA store (often outdated or empty) | **Full control:** pinned `sbm-server` CA / Ed25519 verification, TLS 1.3 |
| **Wi-Fi Reality** | Almost no consumer UEFI firmware has Wi-Fi drivers | Same — UEFI SNP rarely covers Wi-Fi |

**Recommendation for `sbm-uefi`:**
1. Phase 1 (SBM 0.1): Implement **Rust `smoltcp` + `embedded-tls` (or `rustls` `no_std`) over UEFI SNP** (with optional native `EFI_HTTP_PROTOCOL` fallback), enforcing strict bounded timeouts (DHCP ≤ 2.5 s, TCP/TLS/HTTP fetch ≤ 2.0 s, total network budget ≤ 4.0 s, immediately skippable via keypress).
2. Always treat UEFI networking as **opportunistic** (wired Ethernet / virtio-net / USB Ethernet). Full Wi-Fi, WPA3, WireGuard, and large OS downloads belong in **`sbm-core` (Linux)**.

### EFI Chainloading & Linux EFI Stub
- Standard EFI targets (Windows `\EFI\Microsoft\Boot\bootmgfw.efi`, GRUB, systemd-boot, ZFSBootMenu, UKIs): construct full `DevicePath`, call `uefi::boot::load_image(ImageSource::FromDevicePath { ... })`, set UTF-16 `load_options` via `LoadedImage::set_load_options`, and call `uefi::boot::start_image`.
- Direct Linux `CONFIG_EFI_STUB=y` kernels with separate initrd (such as `sbm-core` if not packaged as a single UKI): expose initrd via virtual handle implementing `EFI_LOAD_FILE2_PROTOCOL` with `LINUX_EFI_INITRD_MEDIA_GUID` (`5568e427-68fc-4f3d-ac74-ca555231cc68`), avoiding deprecated handoff structs.
- **Preferred `sbm-core` format:** Single **Unified Kernel Image (UKI)** `.efi` binary so `sbm-uefi` boots `sbm-core` with a single `LoadImage`/`StartImage` call, full Secure Boot signature verification, and TPM PCR measurement!

### Persistent Configuration: ESP Files vs UEFI NVRAM
- Writing frequently to **UEFI NVRAM** causes SPI NOR flash wear, garbage-collection fragmentation, and **documented bricking on buggy consumer laptop firmwares** (especially older Samsung/Lenovo/HP Insyde/AMI firmwares). Furthermore, **Asahi U-Boot on Apple Silicon does not persist arbitrary UEFI NVRAM variables back to Apple flash**.
- **SBM Rule:** Store all SBM configuration, boot policy cache, boot-attempt counters, and one-shot boot commands as **atomic files on the EFI System Partition (`\EFI\sbm\...`)** using write-temp + sync + rename. Touch UEFI NVRAM **only** during SBM installation/uninstallation (`BootOrder`/`BootXXXX`) and read-only volatile handoff variables (`LoaderEntrySelected` / SBM vendor GUID) where supported.
