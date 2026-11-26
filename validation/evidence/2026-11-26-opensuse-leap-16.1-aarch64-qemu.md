# openSUSE Leap 16.1 aarch64 online installer, one QEMU boot

Date: 2026-11-26. One run. This is a boot log, not a benchmark.

Image: `Leap-16.1-online-installer-aarch64.install.iso`, 768606208 bytes, SHA-256 `71d9f43a0e920010526b5e1853d8add4cec1dbfdd7e5f30d132bc99da924d2f3`. `tools/boot-installer.sh` checked that digest before starting QEMU. The signed upstream document is the SHA-512 file, verified with openSUSE Project Signing Key `AD485664E901B867051AB15F35A2F86E29B700A4`. SHA-512 `e324cc6b418b2586d4def9fb8eae14e354a8aa07ed58d5c56a1f2c3d894ac477603a8c73bc2cc38ced0306064de5cfe596aa74e29c56eec0ed938ba578075282`.

Host firmware for the guest was `/opt/homebrew/share/qemu/edk2-aarch64-code.fd` and a copy of `edk2-arm-vars.fd`. The machine type was `virt` with HVF. No host disk was attached. The guest is generic AArch64 UEFI, not Apple Silicon.

The wrapper sent SIGTERM after 75 seconds. QEMU reported termination on signal 15. The serial file was 31633 bytes.

Firmware and bootloader text in that file:

- `UEFI firmware (version edk2-stable202408-prebuilt.qemu.org built at 16:28:50 on Sep 12 2024)`
- `Image type X64 can't be loaded on AARCH64 UEFI system.`
- `BdsDxe: loading Boot0001 "UEFI Misc Device"`
- `GNU GRUB version 2.14`
- menu text `Install Leap 16.1 (aarch64)`, `Failsafe -- Install Leap 16.1 (aarch64)`, `Check Installation Medium`, and `Rescue System`
- `The highlighted entry will be executed automatically in 60s.`

The 75 second wait outlasted that menu timeout. The serial log then shows the guest Linux kernel, including `Trying to unpack rootfs image as initramfs...`. No disk was attached, and the process was stopped during that early boot. No operating system was installed.
