# External boot on Apple Silicon

**Last verified:** 2026-11-26

Levels describe the platform and the upstream software that already exists. They do not mean SBM has performed that boot. The encoded table is `sbm_as_platform::external::assessments`. SBM does not bypass Apple boot security. A pure external boot with no internal change is the case Asahi's FAQ calls unavailable.

Sources: the M1–M5 study, Asahi's [quirks](https://asahilinux.org/docs/platform/quirks/), [boot guide](https://asahilinux.org/docs/alt/boot-process-guide/), [U-Boot page](https://asahilinux.org/docs/sw/u-boot/), [FAQ](https://asahilinux.org/docs/project/faq/), and the [OpenBSD/arm64](https://www.openbsd.org/arm64.html) platform page.

## What "external" can mean

m1n1 stage 1 loads stage 2 from the internal NVMe ESP only. U-Boot's default payload is `\EFI\BOOT\BOOTAA64.EFI` on the paired ESP. The quirks page separates that System ESP (always internal, holding m1n1 stage 2, U-Boot, device trees, and `vendorfw.cpio`) from the OS ESP, which can be on USB when U-Boot was told to boot USB. The System ESP PARTUUID is the device-tree property `asahi,efi-system-partition`. One OS owns `/m1n1/boot.bin`. Other installs set `M1N1_UPDATE_DISABLED=1`.

The pinned Asahi defconfig does not run `usb start` before `bootflow scan -b`. Asahi documents interrupting U-Boot and running `bootcmd_usb0` for a USB disk. An SBM menu that is itself `BOOTAA64.EFI` on the internal ESP can see USB block devices only after U-Boot has started USB. `config/apple-uboot.fragment` is that one-line preboot change. It has not been built or booted on Apple hardware.

USB rescue images are a documented exception: their own ESP, their own `BOOTAA64.EFI`, and they do not manage the internal `boot.bin`.

## Scenarios

| Scenario | M1 | M2 | M3 | M3 Ultra | M4 | M5 |
|---|---|---|---|---|---|---|
| Apple external macOS startup disk | Supported today | Supported today | Supported today | Supported today | Supported today | Supported today |
| External ARM64 Linux disk | Demonstrated experimentally | Demonstrated experimentally | Demonstrated experimentally | Requires upstream | Requires upstream | Requires upstream |
| External Linux root, internal boot chain | Supported today | Supported today | Supported today | Requires upstream | Requires upstream | Requires upstream |
| Generic ARM64 UEFI image | Theoretically possible | Theoretically possible | Theoretically possible | Requires upstream | Requires upstream | Requires upstream |
| Other ARM64 OS (OpenBSD's list) | Supported today | Supported today | Unsupported | Unsupported | Unsupported | Unsupported |
| x86 through a hypervisor | Supported today | Supported today | Supported today | Supported today | Supported today | Supported today |

### 1. Apple external macOS

Supported today on every generation as an Apple startup disk. The Mac still starts from internal storage, and the external volume group's LocalPolicy lives with the rest of Apple's boot policy. SBM can show the volume as a reboot target. Selecting it returns to the Apple startup picker. SBM does not write LocalPolicy, does not call `bless`, and does not `StartImage` `boot.efi`. There is no verified UEFI call that sets the Apple startup disk.

### 2. External ARM64 Linux

Asahi documents booting a USB disk from U-Boot on machines where that stack exists. SBM has not done it on hardware, so the level is "demonstrated experimentally" for installer-yes M1, M2, and M3 (not Ultra). M3 Ultra, M4, and M5 still need upstream installer, DART, and USB support before this is available. The menu's contribution, once USB is published to UEFI, is to list FAT ESPs on those disks.

### 3. External root, internal boot chain

Asahi's quirks page says one external USB install may own the internal System ESP and update m1n1, U-Boot, and the device tree there, while its own bootloader lives on the USB ESP. That is the supported layout for machines the installer already supports. It is not a second boot policy. M3 Ultra, M4, and M5 are not installer targets, so they stay at "requires upstream."

### 4. Generic ARM64 UEFI images

U-Boot can `StartImage` an aarch64 PE/COFF binary. A generic image that expects ACPI, 4 KiB pages, or non-Apple drivers does not become a running OS. The QEMU probe shows `StartImage` against a second EFI application built for this test. It does not show a generic distribution booting on Apple hardware. On M3 Ultra, M4, and M5 the bridge itself is not available yet.

### 5. Other ARM64 operating systems

OpenBSD 8.0's arm64 page lists specific M1 and M2 Macs. It says Apple systems need the Asahi installer first, after which the OpenBSD installer can boot from USB. The page does not list M3, M4, or M5. SBM has not booted OpenBSD. Those machines are "supported today" as OpenBSD's port, and "unsupported" otherwise. This pass did not survey other BSD ports.

### 6. x86

x86 operating systems run as guests of a host that has already booted (macOS or Linux, with that host's hypervisor). SBM rejects them as native EFI chainload targets, including a Windows aarch64 EFI binary presented as a native Apple boot. SBM does not ship the hypervisor.

## What SBM can improve

On a machine where the bridge is up, SBM can list the Apple reboot target, the one ESP-owning OS, and USB ESPs that U-Boot has already published, and it can store that choice in `\EFI\sbm\boot-policy.json` because UEFI variables do not persist. It cannot make an M4 or M5 Mac enumerate USB, and it cannot make a generic ARM64 image into an Apple-compatible OS.
