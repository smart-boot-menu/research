# SBM Core baseline check, 2026-11-26

Checked on the Apple Silicon development host and inside a Fedora 45 aarch64 builder VM. Versions below were printed by the tools or by `dnf repoquery`. Image sizes and boot times are not in this note; those belong in `validation/evidence/` after a boot.

## Kept

| Choice | What was verified |
|---|---|
| mkosi 27.1 | `mkosi --version` inside the builder printed `mkosi 27.1` after installing the GitHub tag `v27.1`. Fedora 45's own `mkosi` package is 26, so the image build uses the venv copy. |
| Fedora 45 | Cloud image `Fedora-Cloud-Base-Generic-45-1.4.aarch64.qcow2`, SHA256 `1632acc8bc14d45b6c9596a0acc71b070d3748d1abdcf1a519be5c983bb0286c`. `labwc` 0.20.2-2.fc45, `gparted` 1.8.1-1.fc45, `wayvnc` 0.9.1-5.fc45. Newest `linux-firmware` subpackages queried: 20261019-1.fc45. |
| EROFS + zstd + dm-verity hash | `systemd-repart` on this systemd accepts `Format=erofs`, `Compression=zstd`, `Verity=data` and `Verity=hash`. No verity signature partition. |
| Signed UKI | mkosi `Bootloader=uki` and `SecureBoot=yes` with a local development key. QEMU's firmware in the boot tests does not have that key enrolled, so those tests boot with Secure Boot off. |
| Iced 0.14.0 | crates.io latest published release used by the shell. Default features include wgpu and tiny-skia. `ICED_BACKEND=tiny-skia` selects software rendering. |
| labwc, XWayland, GParted, wayvnc | Packages exist in Fedora 45. GParted's real binary is `/usr/libexec/gpartedbin`. wayvnc is installed and left disabled. |

## RAM-resident threshold

A machine sold as 4 GiB usually reports `MemTotal` below 4 GiB. The automatic copy therefore requires `MemTotal` of at least 3584 MiB and enough bytes for the EROFS image plus 1536 MiB of headroom. Otherwise the root stays on the backing disk and that disk is held read-only for destructive plans. `sbm.toram=never` and unknown values also stay on the backing disk. The same numbers are in `crates/sbm-runtime/src/toram.rs` and `image/initrd-extra/usr/lib/sbm/sbm-toram.sh`.

## Apple profile

`dnf repoquery kernel-16k` on the Fedora 45 repositories enabled in the generic cloud image returned no package. `asahi-scripts` 20250713 (noarch) is present. The `aarch64-apple` mkosi profile names `kernel-16k` and is not built. This does not claim that an M5 (`T8142`) can boot it. The generic ARM64 image uses the 4K `kernel-core` packages because QEMU `virt` uses 4K pages.

## Builder

The image build runs in a Fedora 45 aarch64 VM under QEMU `virt` with HVF, disk and seed image under `sbm-core/build/builder/` (gitignored). Homebrew QEMU on the Mac is 11.1.3. x86_64 guest boots on that Mac are TCG, not HVF. `sbm-lab` is still a Phase 0 repository, so the boot wrapper lives in `sbm-core/scripts/boot-smoke.sh` until Agent 03 replaces it.
