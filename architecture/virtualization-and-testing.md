# Virtualization, Emulation & Automated Testing Research (`sbm-lab`)

**Last verified:** 2026-11-26  
**Scope:** End-to-end virtual hardware simulation, firmware stacks, Apple Silicon M5 workstation workflow, CI runner constraints, and Rust verification tooling.

---

## 1. Core Simulation Stack (November 2026 Versions)

| Tool | Latest Verified Version (Nov 2026) | License | Role in `sbm-lab` |
|---|---|---|---|
| **QEMU** (`qemu-system-x86_64`, `qemu-system-aarch64`) | QEMU `9.x` / `10.x` (via Homebrew on macOS, distro pkg on Linux CI) | `GPL-2.0-only` | Full-system UEFI VM harness for `sbm-uefi`, `sbm-core`, multi-disk GPT fixtures, and `sbm-server` integration tests |
| **EDK II — OVMF & AAVMF** | **`edk2-stable202611`** (Nov 20, 2026) | `BSD-2-Clause-Patent` | Standard x86-64 (`OVMF_CODE.fd`) and ARM64 (`AAVMF_CODE.fd` / `QEMU_EFI.fd`) UEFI firmware, including Secure Boot variants (`OVMF_CODE.secboot.fd`) |
| **Das U-Boot (`qemu_arm64`)** | **`v2026.10`** (Oct 5, 2026) | `GPL-2.0-or-later` | Emulates U-Boot's `CONFIG_EFI_LOADER` UEFI environment (closely mirroring Asahi U-Boot's UEFI subset without requiring physical Apple hardware!) |
| **`swtpm`** | **`v0.10.2`** (Aug 2026) | `BSD-3-Clause` | Software TPM 2.0 emulator attached to QEMU via `chardev`/`tpm-tis`/`tpm-crb` for testing Measured Boot, PCRs, and TPM-backed device identity |
| **QCOW2 Backing Images + Snapshots** | Built into `qemu-img` | `GPL-2.0-only` | Instant copy-on-write disk fixtures (clean Windows-like ESP, multi-Linux BLS ESP, corrupt GPT, BitLocker-flagged partition) without mutating golden fixtures |

---

## 2. Development Workstation (Apple Silicon M5) vs CI Runners

### On the M5 MacBook Air (`aarch64-apple-darwin`, macOS 27.0)
- **ARM64 VMs (`qemu-system-aarch64 -M virt -accel hvf -cpu host`):**
  - Runs at **near-native hardware speed** via Apple's Hypervisor.framework (`hvf`)!
  - Ideal inner-loop test target for `sbm-uefi` (`aarch64-unknown-uefi`), `sbm-core` (`aarch64`), and U-Boot `qemu_arm64` UEFI compatibility tests.
- **x86-64 VMs (`qemu-system-x86_64 -M q35 -accel tcg`):**
  - Runs via QEMU TCG software emulation on Apple Silicon (~5–15× slower than KVM/HVF, completely fine for booting `sbm-uefi` in 2–4 seconds; slower for full GUI `sbm-core` boot, so use `-nographic` / serial assertions or lightweight scenarios locally and full KVM on x86 Linux CI).
- **Local Prerequisites Not Yet Installed on Dev Mac:** Verified during environment inspection (`2026-11-26`) that `qemu-system-x86_64`, `qemu-system-aarch64`, and `swtpm` are not yet installed via Homebrew (`brew install qemu swtpm` will provision them cleanly for Agent 3 / `sbm-lab`).

### On GitHub Actions CI
- **`ubuntu-latest` (x86-64):** Supports nested KVM on supported runner classes (and fast TCG fallback for UEFI boot smoke tests). Runs `x86_64` OVMF + `sbm-uefi` + `sbm-core` integration tests.
- **`ubuntu-24.04-arm` (ARM64):** Note that standard GitHub-hosted ARM64 runners typically do **not** expose `/dev/kvm` (`qemu-system-aarch64` falls back to TCG), which is still fast enough for `sbm-uefi` AAVMF boot + serial/QMP screenshot assertions, while heavier `sbm-core` ARM64 tests can run on self-hosted or KVM-enabled runners.

---

## 3. Rust Testing, Fuzzing & Fault-Injection Strategy

1. **Unit & Contract Tests (`cargo test`):**
   - Pure-logic unit tests across `sbm-common`, `sbm-uefi` (mocked UEFI traits), `sbm-server`, `sbm-agent`, `install`.
   - Golden JSON Schema & protocol test vectors in `sbm-common/test-vectors/` verified by every producer and consumer.
2. **Property-Based Testing (`proptest`):**
   - Fuzz-like randomized testing of GPT partition plan validators, boot policy precedence resolvers, protocol version negotiation, and non-overlapping partition bounds.
3. **Coverage-Guided Fuzzing (`cargo-fuzz` / `libFuzzer`):**
   - Mandatory fuzz targets for all untrusted input parsers:
     - `sbm-uefi`: BLS `.conf` parser, PE/COFF header inspector, HTTP/JSON policy response parser, FAT path sanitizer.
     - `sbm-common`: `CommandEnvelope` signature & JSON decoder, `CatalogManifest` validator.
     - `sbm-core`: `lsblk`/`sfdisk` output parsers, storage plan validator.
4. **Fault Injection (Testing Failure Modes Deliberately):**
   - **QEMU QMP power-cut injection:** Kill VM mid-boot or mid-update to verify `sbm-uefi` boot-attempt fallback and atomic ESP config recovery.
   - **Network fault injection:** Drop packets, blackhole TCP, inject TLS cert mismatch, inject expired/replayed `CommandEnvelope` -> verify bounded timeout (< 4 s) and clean offline fallback.
   - **Disk fault injection:** Linux `dm-flakey` / read-only block devices in `sbm-core` tests -> verify graceful error handling with zero partition table corruption.
5. **Visual & Latency Verification:**
   - Capture UEFI GOP framebuffer via QEMU QMP `screendump` to verify graphical menu rendering automatically.
   - Measure real boot timestamps via UEFI monotonic clock / QEMU serial markers (`LoaderTimeInitUSec` -> first frame -> user-interactive ready) — benchmark methodology defined in `docs/testing/benchmarks.md` before publishing any performance claims.
