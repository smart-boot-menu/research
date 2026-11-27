# Smart Boot Menu — Research (`research`)

Primary-source ecosystem research, hardware enablement studies, technology evaluations, and empirical validation evidence for [Smart Boot Menu](https://github.com/smart-boot-menu).

Modeled after dedicated research repositories (`0SFS/research`), this repository separates **primary-source technical investigation and empirical evidence** from **normative specifications and ADRs** (which live in [`smart-boot-menu/docs`](https://github.com/smart-boot-menu/docs)) and **project coordination** (in [`smart-boot-menu/sbm`](https://github.com/smart-boot-menu/sbm)).

**Research baseline:** November 2026. Every release version, license identifier, and hardware support status is sourced from primary documentation, upstream repositories, or standards; unverified claims are explicitly marked as unverified or open questions.

## Index

### Ecosystem Surveys (`ecosystem/`)

- [Boot Management Ecosystem](ecosystem/boot-management.md) — rEFInd, RefindPlus, Meridian, Visor, GRUB 2, systemd-boot, Limine, Clover, OpenCore, iPXE, U-Boot, EDK2, Petitboot, LinuxBoot, u-root, and modern Rust UEFI projects (`uefi-rs`, `r-efi`, `crdyboot`, `oreboot`, Patina, Redox bootloader).
- [OS Installation & Bare-Metal Provisioning](ecosystem/os-provisioning.md) — PINN, NOOBS, Ventoy, iVentoy, netboot.xyz, MAAS, Tinkerbell, FOG Project, OpenStack Ironic, Foreman, Calamares, `bootc`/`ostree`, `systemd-repart`, `wimlib`, `osinfo-db`, Agama, and cross-OS (Windows/Linux/BSD) installation mechanics from Linux.
- [Installer mechanisms for the OS catalog](ecosystem/installer-mechanisms.md) — which of those mechanisms, plus Ubuntu autoinstall, Debian preseed, Kickstart, archinstall, NixOS, Windows Setup, WinPE, and FreeBSD, are stable enough to record in `sbm-catalog` without executing them.
- [Recovery & Storage Ecosystem](ecosystem/recovery-and-storage.md) — GParted / GParted Live, SystemRescue, Rescuezilla, Clonezilla, Partclone, GNU Parted, `util-linux`, TestDisk/PhotoRec, GNU `ddrescue`, ZFSBootMenu, Parted Magic, BootIt UEFI, `cryptsetup` (LUKS2 + BitLocker `bitlk`), LVM/RAID/Btrfs/ZFS, and Rust storage crates.
- [Remote Management & Out-of-Band Ecosystem](ecosystem/remote-management.md) — RustDesk, noVNC, MeshCentral, Tailscale, Headscale, WireGuard (`boringtun`/`gotatun`), Iroh, JetKVM, PiKVM, Intel AMT / Device Management Toolkit, DMTF Redfish, IPMI, and emerging Rust remote/KVM stacks.
- [Development Infrastructure, Testing & Tooling Landscape (Build vs. Reuse Audit)](ecosystem/dev-infrastructure-and-testing-landscape-2026-11-26.md) — primary-source evaluation of EDK II `EmulatorPkg`, Intel `EfiWrapper`, U-Boot `sandbox`, `openQA`/`os-autoinst`, `labgrid`, `test-foundry`, `virtme-ng`, `Lima`, `vfkit`, `libkrun`/`krunkit`, `mkosi`, `systemd-sysext`, `sccache`, `cargo-zigbuild`, Cargo 1.100 `build-dir`, and a module-by-module audit of `sbm-lab` (9,805 LOC).
- [SBM Manager & USB Flashing Tools Evaluation (Build vs. Reuse)](ecosystem/media-creator-evaluation.md) — primary-source evaluation of Raspberry Pi Imager (`--repo` custom repository JSON v4, `.zst` support), Fedora Media Writer, balenaEtcher, USBImager, GNOME Impression, KDE ISO Image Writer, Rufus, Caligula, and trade-offs for **SBM Manager**'s expanded profile/module/remote-recovery configuration scope.

### Hardware & Platform Studies (`hardware/`)

- [Apple Silicon Architecture and M1–M5 Support](hardware/apple-silicon-m1-to-m5.md) — feature tables fetched 2026-11-26, corrected boot chain, and per-generation limits. `T8142` is not an install target.
- [External Boot Feasibility](hardware/apple-silicon-external-boot.md) — six scenarios, split by generation, without treating QEMU or one SoC as evidence for another.
- [GPUs, Graphics & GUI Frameworks (UEFI vs Linux)](hardware/gpu-and-graphics-acceleration.md) — Integrated/discrete GPU reality vs GUI libraries, `igpu` / `uefi-wm` verification, Slint 1.19.1 `no_std`, egui, Iced, wgpu, Vello, `embedded-graphics`, custom framebuffer rendering, Wayland/XWayland for GTK3 GParted, HiDPI, and low-RAM older laptops.

### Architecture & Technology Deep Dives (`architecture/`)

- [Rust UEFI Environment (`sbm-uefi`)](architecture/uefi-rust-stack.md) — Rust 2024 edition, `uefi-rs` v0.41.0, `no_std`+`alloc`, GOP, `SimpleFileSystem`/efifs, input protocol firmware bugs, UEFI HTTP/TLS vs `smoltcp`+TLS over SNP, EFI chainloading, and NVRAM wear risks vs ESP config.
- [Minimal Linux Management OS (`sbm-core`)](architecture/minimal-linux-core-os.md) — Buildroot, Yocto, Alpine, Debian, Fedora, NixOS, and `mkosi` v27.1; SquashFS vs EROFS; UKI; BusyBox vs systemd; `linux-firmware` sizing; `dm-verity` & A/B updates; `systemd-sysext`; concrete Core OS comparison for x86/ARM64 PCs vs Apple Silicon (`< 1 GB`, well below 4 GB).
- [SBM Core baseline check, 2026-11-26](architecture/sbm-core-baseline-validation.md) — Fedora 45 package names, mkosi 27.1, the RAM-resident threshold, and the missing `kernel-16k` package.
- [Internal SSD Installation, Partitioning Tools & Calamares Evaluation](architecture/internal-install-and-calamares-evaluation.md) — critical evaluation of Calamares `3.4.3` (Qt6/KF6/KPMcore, `GPL-3.0-or-later`) vs preserving SBM's Rust planner and security boundaries while wrapping mature partitioning/filesystem/UEFI tools (`sfdisk`, `sgdisk`, `dosfstools`, `e2fsprogs`, `ntfsresize`, `btrfs-progs`, `efibootmgr`, `GParted`), safe shrinking rules, and Windows/BitLocker coexistence pitfalls.
- [Network Recovery Bootstrap & Maximum Modularity Evaluation](architecture/network-recovery-and-modularity-evaluation.md) — primary-source evaluation of UEFI HTTP(S) Boot, iPXE, signed UKIs, minimal Linux network-bootstrap UKI + `dm-verity` `switch-root` (Ethernet vs Wi-Fi, RAM-only zero-SSD-write invariant, HTTP Range resume, ephemeral revocable remote recovery), and `0SFS`-style modularity via `mkosi` (`Format=sysext`/`confext`), signed `dm-verity` DDIs, TPM2 PCR 13, and `systemd-sysupdate` without inventing a custom package manager.
- [Boot Security, Server Stack & Supply Chain](architecture/security-and-supply-chain.md) — UEFI Secure Boot, `rhboot/shim` 16.1, SBAT, MOK, Microsoft UEFI CA 2011 expiration (June 2026) & 2023 CA rollout, TPM 2.0 Measured Boot & BitLocker PCR triggers, TUF v1.0.33 / Sigstore / SLSA, and Rust server auth stack (Axum, Tokio, SQLx, WebAuthn, mTLS).
- [Server crate pins re-checked 2026-11-26](architecture/sbm-server-stack-2026-11-26.md) — crates.io and npm versions used by the `sbm-server` 0.1 implementation.
- [Virtualization, Emulation & Testing (`sbm-lab`)](architecture/virtualization-and-testing.md) — QEMU, EDK2 `edk2-stable202611` (OVMF/AAVMF), U-Boot `v2026.10`, `swtpm` v0.10.2, Apple Silicon HVF vs x86 TCG, GitHub Actions CI, `proptest`, `cargo-fuzz`, fault injection, and reproducible builds.

### Validation Evidence (`validation/`)

- `validation/evidence/` — Retained empirical test results, benchmark measurements, and hardware probe logs.
- [SBM Developer Resource Audit & QEMU HVF Measurements (2026-11-26)](validation/evidence/sbm-dev-resource-audit-2026-11-26.md) — Phase 1 host & workspace storage audit (54.53 GiB baseline, 10.05 GiB duplicates), build/test latencies, Phase 6 QEMU HVF `virtio-balloon` vs `discard=unmap` + `fstrim` benchmarks, and 13.24 GiB measured SSD reclamation.
- [SBM 0.0.1 integration baseline](validation/evidence/sbm-integration-baseline-2026-11-26.md) — Hosted CI failures, builder evidence, host test counts, and EFI artifact hashes.
- [SBM 0.0.1 Wave 2 integration audit evidence](validation/evidence/sbm-integration-wave2-2026-11-26.md) — Wave 2 commits, candidate image provenance, UEFI O(N²) FAT32 hashing breakdown, hosted CI failure root causes, Core graphical boot blockers, and shared resource manager audit.
- [U-Boot EFI probe under QEMU virt, 2026-11-26](validation/evidence/sbm-as-uboot-efi-2026-11-26.md) — protocol probe only. QEMU `virt` is not an Apple Silicon Mac.
- [Debian 13.8.0 arm64 netinst QEMU boot, 2026-11-26](validation/evidence/2026-11-26-debian-13.8.0-arm64-netinst-qemu.md) — one HVF run that reached the GRUB install menu. This is a boot log, not a benchmark.
- [openSUSE Leap 16.1 aarch64 online installer QEMU boot, 2026-11-26](validation/evidence/2026-11-26-opensuse-leap-16.1-aarch64-qemu.md) — one HVF run. GRUB showed the Leap installer, then the guest kernel started unpacking an initramfs. No disk was attached.
- [Fedora 45 Server aarch64 netinst QEMU boot, 2026-11-26](validation/evidence/2026-11-26-fedora-45-aarch64-netinst-qemu.md) — one HVF run. The highlighted GRUB entry started the installer kernel with `rd.live.check`. No disk was attached.
- [sbm-server release timings, 2026-11-26](validation/evidence/sbm-server-2026-11-26.md) — one cold start and N=10 warm starts plus health and boot-policy requests on the development Mac. These are not UEFI frame times.

## License

Original documentation in this repository is dual-licensed under [MIT](LICENSE-MIT) OR [Apache-2.0](LICENSE-APACHE).
