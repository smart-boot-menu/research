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

### Hardware & Platform Studies (`hardware/`)

- [Apple Silicon Architecture and M1–M5 Support](hardware/apple-silicon-m1-to-m5.md) — feature tables fetched 2026-11-26, corrected boot chain, and per-generation limits. `T8142` is not an install target.
- [External Boot Feasibility](hardware/apple-silicon-external-boot.md) — six scenarios, split by generation, without treating QEMU or one SoC as evidence for another.
- [GPUs, Graphics & GUI Frameworks (UEFI vs Linux)](hardware/gpu-and-graphics-acceleration.md) — Integrated/discrete GPU reality vs GUI libraries, `igpu` / `uefi-wm` verification, Slint 1.19.1 `no_std`, egui, Iced, wgpu, Vello, `embedded-graphics`, custom framebuffer rendering, Wayland/XWayland for GTK3 GParted, HiDPI, and low-RAM older laptops.

### Architecture & Technology Deep Dives (`architecture/`)

- [Rust UEFI Environment (`sbm-uefi`)](architecture/uefi-rust-stack.md) — Rust 2024 edition, `uefi-rs` v0.41.0, `no_std`+`alloc`, GOP, `SimpleFileSystem`/efifs, input protocol firmware bugs, UEFI HTTP/TLS vs `smoltcp`+TLS over SNP, EFI chainloading, and NVRAM wear risks vs ESP config.
- [Minimal Linux Management OS (`sbm-core`)](architecture/minimal-linux-core-os.md) — Buildroot, Yocto, Alpine, Debian, Fedora, NixOS, and `mkosi` v27.1; SquashFS vs EROFS; UKI; BusyBox vs systemd; `linux-firmware` sizing; `dm-verity` & A/B updates; `systemd-sysext`; concrete Core OS comparison for x86/ARM64 PCs vs Apple Silicon (`< 1 GB`, well below 4 GB).
- [Boot Security, Server Stack & Supply Chain](architecture/security-and-supply-chain.md) — UEFI Secure Boot, `rhboot/shim` 16.1, SBAT, MOK, Microsoft UEFI CA 2011 expiration (June 2026) & 2023 CA rollout, TPM 2.0 Measured Boot & BitLocker PCR triggers, TUF v1.0.33 / Sigstore / SLSA, and Rust server auth stack (Axum, Tokio, SQLx, WebAuthn, mTLS).
- [Server crate pins re-checked 2026-11-26](architecture/sbm-server-stack-2026-11-26.md) — crates.io and npm versions used by the `sbm-server` 0.1 implementation.
- [Virtualization, Emulation & Testing (`sbm-lab`)](architecture/virtualization-and-testing.md) — QEMU, EDK2 `edk2-stable202611` (OVMF/AAVMF), U-Boot `v2026.10`, `swtpm` v0.10.2, Apple Silicon HVF vs x86 TCG, GitHub Actions CI, `proptest`, `cargo-fuzz`, fault injection, and reproducible builds.

### Validation Evidence (`validation/`)

- `validation/evidence/` — Retained empirical test results, benchmark measurements, and hardware probe logs.
- [U-Boot EFI probe under QEMU virt, 2026-11-26](validation/evidence/sbm-as-uboot-efi-2026-11-26.md) — protocol probe only. QEMU `virt` is not an Apple Silicon Mac.
- [Debian 13.8.0 arm64 netinst QEMU boot, 2026-11-26](validation/evidence/2026-11-26-debian-13.8.0-arm64-netinst-qemu.md) — one HVF run that reached the GRUB install menu. This is a boot log, not a benchmark.
- [sbm-server release timings, 2026-11-26](validation/evidence/sbm-server-2026-11-26.md) — one cold start and N=10 warm starts plus health and boot-policy requests on the development Mac. These are not UEFI frame times.

## License

Original documentation in this repository is dual-licensed under [MIT](LICENSE-MIT) OR [Apache-2.0](LICENSE-APACHE).
