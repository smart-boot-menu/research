# Fedora 45 Server aarch64 netinst, one QEMU boot

Date: 2026-11-26. One run. This is a boot log, not a benchmark.

Image: `Fedora-Server-netinst-aarch64-45-1.4.iso`, 1436913664 bytes. After download, `shasum -a 256` printed `311cb9fb35aa39b7df5768c0ccfef571a515d471e824af0d8cbe96de645ddfe8`, the same digest as the cleartext-signed Fedora 45 aarch64 CHECKSUM. `tools/boot-installer.sh` checked that digest again before starting QEMU.

Host firmware for the guest was `/opt/homebrew/share/qemu/edk2-aarch64-code.fd` and a copy of `edk2-arm-vars.fd`. The machine type was `virt` with HVF. No host disk was attached. The guest is generic AArch64 UEFI, not Apple Silicon.

The wrapper sent SIGTERM after 75 seconds. QEMU reported termination on signal 15. The serial file was 33022 bytes.

Firmware and bootloader text in that file:

- `UEFI firmware (version edk2-stable202408-prebuilt.qemu.org built at 16:28:50 on Sep 12 2024)`
- `Image type X64 can't be loaded on AARCH64 UEFI system.`
- `BdsDxe: loading Boot0001 "UEFI Misc Device"`
- `GRUB version 2.12`
- menu text `Install Fedora 45`, `Test this media & install Fedora 45`, and `Troubleshooting`
- `The highlighted entry will be executed automatically in 60s.`

The highlighted entry started. The kernel command line in the serial log is `BOOT_IMAGE=/images/pxeboot/vmlinuz inst.stage2=hd:LABEL=Fedora-S-dvd-aarch64-45 rd.live.check`. The log also contains `Loaded X.509 cert 'Fedora kernel signing key: b1525a3668c2a70c9b9733c5a9f40d35d7a13f85'`. No disk was attached, and the process was stopped during early kernel startup. No operating system was installed.
