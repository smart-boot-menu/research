# U-Boot EFI probe, QEMU virt, 2026-11-26

This is a protocol log. QEMU `virt` is not an Apple Silicon Mac. The run does not show Apple NVMe, DART, xHCI, Boot ROM, or LocalPolicy. It does not measure boot time or image size.

## How it was run

`sbm-as/scripts/uboot-efi-poc.sh` on the development Mac, which was not modified.

- QEMU emulator version 11.1.3 (`/opt/homebrew/bin/qemu-system-aarch64`)
- U-Boot `v2026.10`, tarball sha256 `f943a88389c8fd70a6b0d061a88267465e143c7ca6ea2b75837ab20f5b60c739`, commit `a58e2b7af42c179ace2c6210de0a219fa3f3cf3c`
- Config: `qemu_arm64_defconfig` plus `sbm-as/config/uboot-qemu-efi.fragment` (`BOOTDELAY=0`, Bochs video 1024x768, `PREBOOT="usb start"`, `EFI_LOADER`)
- Probe crate `sbm-as-efi-probe` with `uefi` 0.41.0, target `aarch64-unknown-uefi`
- Disks: a 32 MiB virtio FAT ESP and a second 32 MiB FAT image on USB EHCI
- Host tools used to compile U-Boot: GNU make 4.4.1, Homebrew bash 5.3.20, Homebrew `openssl@3`

The script tries HVF first. That process aborted before U-Boot printed a banner:

```
Assertion failed: (isv), function hvf_handle_exception, file hvf.c, line 2447.
```

The passing command was:

```
qemu-system-aarch64 -machine virt -cpu cortex-a57 -m 1024 -serial stdio -display none -device VGA -bios u-boot.bin
```

with the virtio ESP and the EHCI USB disk. Full lines are in the script's `build/poc/serial.log`, which is gitignored.

## What U-Boot printed

```
U-Boot 2026.10 (Nov 26 2026 - 05:21:35 -0600)
Cannot persist EFI variables without system partition
Missing TPMv2 device for EFI_TCG_PROTOCOL
Missing RNG device for EFI_RNG_PROTOCOL
Loading Boot0000 'usb 0' failed
Booting: Label: virtio 0
```

USB EHCI reported 2 USB devices. The automatic USB boot entry failed. U-Boot then started the probe from the virtio disk's default EFI path. `usb start` had already run from `PREBOOT`, which is why the probe could see the second disk afterward.

## What the probe printed

```
SBM-AS-PROBE: start
SBM-AS-PROBE: qemu-virt-is-not-apple-silicon
SBM-AS-PROBE: gop=1024x768 filled=true
SBM-AS-PROBE: keyboard=true
SBM-AS-PROBE: pointer=false
SBM-AS-PROBE: block-io=4
SBM-AS-PROBE: fdt=true
SBM-AS-PROBE: variable=nonvolatile:true,volatile:false,read:true
SBM-AS-PROBE: policy-written
SBM-AS-PROBE: discovered=\EFI\sbm\chain-target.efi
SBM-AS-CHAIN: ok
SBM-AS-PROBE: chainload=ok
SBM-AS-PROBE: discovered=\EFI\sbm\external-payload.efi
SBM-AS-PROBE: done
```

`filled=true` means `GraphicsOutput.Blt` of a 32 by 32 rectangle returned success. The probe does not switch GOP mode. The Bochs VGA device is the QEMU display; Asahi U-Boot's GOP is `CONFIG_VIDEO_SIMPLE`.

`fdt=true` means the standard EFI configuration table was present. That table on `virt` is QEMU's device tree, not an Apple one.

`variable=nonvolatile:true` means `SetVariable` with `NON_VOLATILE` returned success and `GetVariable` read it back in the same boot. `volatile:false` means the probe skipped the volatile attempt after that success. U-Boot had already printed that it cannot persist EFI variables without a system partition. The probe does not reboot, so durability was not measured. Asahi's quirks page says the Apple build has no variable store at all. `nvram_writable` stays false for Apple Silicon. This QEMU result does not change that.

`policy-written` is `\EFI\sbm\boot-policy.json` on the virtio ESP. The chain target printed `SBM-AS-CHAIN: ok` and returned. The external marker was a file on the EHCI disk, not a Linux installation.

## What this does not show

- No Apple Silicon generation booted SBM.
- No macOS, Linux, OpenBSD, or Windows image was chainloaded.
- No internal disk, LocalPolicy, or firmware on the host was written.
- The untested `config/apple-uboot.fragment` was not used.
