# Apple Silicon Architecture, Boot Chain, and M1–M5 Support

**Last verified:** 2026-11-26

This note corrects the 2026-11-26 draft against the pages and commits fetched that day. Support for one M-series generation is not evidence for another. `linux-asahi (7.5)` is Asahi's label for a feature that is available in the `linux-asahi` tree and expected to land upstream by 7.5. It is not a kernel tag. `git ls-remote --tags` on `AsahiLinux/linux` the same day showed `v7.3.4` (`7595a62d45ee1a2297eafa5f2ef5d0a45142d84d`) as the newest `v7.*` tag, and no `v7.4` or `v7.5` tag.

## Sources

- Asahi feature tables, fetched 2026-11-26: [M1](https://asahilinux.org/docs/platform/feature-support/m1/), [M2](https://asahilinux.org/docs/platform/feature-support/m2/), [M3](https://asahilinux.org/docs/platform/feature-support/m3/), [M4](https://asahilinux.org/docs/platform/feature-support/m4/), [M5](https://asahilinux.org/docs/platform/feature-support/m5/). Each page defines `linux-asahi`, `linux-asahi (release)`, `WIP`, and `TBA`.
- Asahi device list: <https://asahilinux.org/docs/hw/devices/device-list/>
- [Platform quirks](https://asahilinux.org/docs/platform/quirks/), [boot process](https://asahilinux.org/docs/alt/boot-process-guide/), [U-Boot](https://asahilinux.org/docs/sw/u-boot/), [FAQ](https://asahilinux.org/docs/project/faq/)
- Pinned Asahi U-Boot `asahi-releng` `ec49c9d70e6ab003813d6f475fec62dc1c0f4bfe`, `configs/apple_m1_defconfig` and `lib/efi_loader/Kconfig`
- Pinned asahi-installer `0259acb983e77b1499629c62f96cc690fb8aa9cd`, `src/step2/step2.sh`
- Pinned tags in `sbm-as/config/pins.env`: m1n1 `v2.0.1` (`5e3a756398bfe50189a1018ff2b876d61d395c08`), linux-asahi `v7.3.4`
- Apple Platform Security guide (boot process, LocalPolicy, paired recoveryOS, startup-disk policy): <https://support.apple.com/guide/security/>
- OpenBSD/arm64 platform page: <https://www.openbsd.org/arm64.html>

Third-party M5 GPU blog posts are not support evidence. Apple feedback `FB23916610` was not fetched from developer.apple.com. It is not used here, and it is not a finding about base M5 `T8142`. The development Mac is an M5 MacBook Air on macOS 27.0 (Darwin 27.0.0, build `26A428`).

## Corrections to the previous draft

1. The custom boot object is `boot.bin`, installed from paired one-true recovery. The pinned `step2.sh` checks that the current recovery is paired to the new volume group and is one-true recovery, then runs `bputil -nc` for that volume group and `kmutil configure-boot --raw` on `boot.bin`. The script tells the user that the macOS installation keeps its own security level. The earlier "unsigned Mach-O via `bputil -k`" description does not match this script. The LocalPolicy record is the SEP-signed, anti-replay object Apple documents. SBM does not run these commands.
2. Asahi's quirks page says there is no UEFI variable storage, so U-Boot always starts `\EFI\BOOT\BOOTAA64.EFI`. `nvram_writable` stays false on Apple Silicon. A QEMU flash variable that accepts `SetVariable` is a different machine. See the U-Boot EFI evidence note.
3. m1n1 stage 1 chainloads stage 2 only from the internal NVMe ESP (`/m1n1/boot.bin`). The pinned defconfig's boot command is `bootflow scan -b`. `CONFIG_USE_PREBOOT` is set and no `CONFIG_PREBOOT` string is set, so USB is not started before that scan. Asahi's U-Boot page still documents a manual `run bootcmd_usb0` after interrupting autoboot.
4. `CONFIG_EFI_LOADER` is not named in the pinned defconfig. Its Kconfig default is `y` for `armv8`, which this Apple config is.
5. M4 Wi-Fi and Bluetooth are `linux-asahi` on M4 and M4 Pro/Max device rows. They are `TBA` on A18 Pro (`T8140`). DART is `TBA` on every M4-class SoC. The installer column is `no` on every M4-class device.
6. M3 is not one row. Mac Studio M3 Ultra (`T6032`) installer is `no`. GPU is `WIP` on `T8122` and `TBA` on Pro, Max, and Ultra. USB3 and DCP are `linux-asahi` except Ultra, where both are `WIP`.
7. Fedora OpenGL and Vulkan version claims are not copied onto M3, M4, or M5. The GPU cell is `linux-asahi` for M1 and M2, `WIP` for M3 base, and `TBA` after that.
8. The 450–850 MiB and 1.5–2 GiB figures are design targets. This study did not measure an image.
9. A claim that U-Boot gained `t8122` and `t6032` support in a specific May–September 2026 merge was not checked against the pinned commit and is withdrawn.
10. M6 appears on Asahi's overview and is outside this matrix.

## Boot chain

Apple Silicon does not boot a UEFI firmware image from SPI flash. Apple's guide describes Boot ROM, LLB, a per-OS LocalPolicy, and iBoot. Full, Reduced, and Permissive security are per installed OS. Permissive security is available from paired one-true recovery.

Asahi's boot guide, for an installed system:

1. Apple's chain loads m1n1 stage 1. Stage 1 has the ESP PARTUUID from install time and loads `/m1n1/boot.bin` from that ESP on the internal NVMe. External media is not a stage-2 source.
2. Stage 2 is m1n1 plus the device tree plus U-Boot. It hands the flattened device tree onward.
3. U-Boot provides UEFI and, in the release configuration, loads `\EFI\BOOT\BOOTAA64.EFI`.

The pinned `apple_m1_defconfig` (the name is historical; `CONFIG_ARCH_APPLE=y` and `CONFIG_OF_UPSTREAM_VENDOR="apple"`) enables `CONFIG_NVME_APPLE`, `CONFIG_USB_XHCI_HCD`, `CONFIG_USB_XHCI_DWC3`, `CONFIG_USB_XHCI_PCI`, `CONFIG_USB_DWC3`, `CONFIG_USB_KEYBOARD`, `CONFIG_APPLE_SPI_KEYB`, `CONFIG_APPLE_MTP_KEYB`, `CONFIG_VIDEO_SIMPLE`, `CONFIG_NO_FB_CLEAR`, and `CONFIG_NO_NET`. It does not enable a USB mouse or trackpad. `CONFIG_NO_NET` means this U-Boot build has no UEFI network protocol.

Asahi's FAQ states that a pure USB boot with no change to internal storage is not available. The internal, Apple-authorized bootstrap is the path that exists. SBM does not bypass that policy.

## How to read the matrix

Cells are transcribed from the feature tables above. A number such as `7.0` means Asahi lists that upstream kernel. `linux-asahi` with no number means Asahi's "available in linux-asahi" cell. `(7.4)` or `(7.5)` is the upstream target in parentheses on that page, not a tag that exists. `Installer: no` means the machine is not an install target, even where a few blocks are marked `linux-asahi`.

`external_usb_boot_supported` in `sbm-as` is true only when the installer cell is `yes`, USB3 is published (`Upstream` or `linux-asahi`), and DART is not `WIP` or `TBA`. A lone USB cell is not enough. M4 base is the example: USB3 is `linux-asahi` and DART is `TBA`, so the bit is false. The M4 page's footnote says many peripherals depend on DART and PCIe.

`bridge_reachable` is true only when the installer cell is `yes` and NVMe is published. `can_chainload_efi` on the shared capability struct means "if U-Boot is already running." Callers also check `bridge_reachable`.

## M1 (`T8103`, `T6000`/`T6001`/`T6002`)

Installer `yes` on the listed devices. USB3 and DART are upstream (`7.0`, and DART `5.15` on the base SoC / `6.1` on Pro/Max/Ultra). NVMe is upstream `5.19`. GPU, display, keyboard, and touchpad are published on the laptop rows. Thunderbolt PCIe and DisplayPort adaptors are `WIP`. Wi-Fi is upstream `6.1` on the base device rows. The capability bits for external USB and the boot bridge are true.

## M2 (`T8112`, `T6020`/`T6021`/`T6022`)

Installer `yes`, except Mac Pro (2023), whose installer cell is `WIP`. USB3 is upstream `7.0`. DART is upstream `6.3` on `T8112` and `6.18` on `T602x`. NVMe is upstream `5.19`. GPU is `linux-asahi`. Thunderbolt PCIe adaptor is `WIP`. Wi-Fi is upstream `6.1` on the base laptop rows and `linux-asahi` on the Pro/Max/Ultra rows. The adapter's Mac Pro (2023) exception sets the installer to `WIP` and clears `bridge_reachable`. Other M2 devices keep both capability bits.

## M3 (`T8122`, `T6030`, `T6031`/`T6034`, `T6032`)

| | M3 `T8122` | M3 Pro `T6030` | M3 Max `T6031`/`T6034` | M3 Ultra `T6032` |
|---|---|---|---|---|
| Installer | yes (iMac, MacBook Pro, MacBook Air rows) | yes (MacBook Pro row) | yes (same Pro/Max row) | no (Mac Studio 2025) |
| USB3 | linux-asahi | linux-asahi | linux-asahi | WIP |
| DART | linux-asahi | linux-asahi | linux-asahi | linux-asahi |
| NVMe | linux-asahi (7.5) | linux-asahi (7.5) | linux-asahi (7.5) | linux-asahi (7.5) |
| GPU | WIP | TBA | TBA | TBA |
| DCP | linux-asahi | linux-asahi | linux-asahi | WIP |
| Thunderbolt PCIe adaptor | WIP | WIP | WIP | WIP |

External USB and the boot bridge are true for base, Pro, and Max. Both are false for Ultra. The adapter marks the Ultra display cell unpublished because DCP is `WIP` and the Studio main-display cell is empty.

## M4, including A18 Pro

| | A18 Pro `T8140` | M4 `T8132` | M4 Pro/Max `T6040`/`T6041` |
|---|---|---|---|
| Installer | no | no | no |
| USB3 | TBA | linux-asahi | TBA |
| DART | TBA | TBA | TBA |
| PCIe | TBA | linux-asahi | linux-asahi |
| NVMe | linux-asahi | linux-asahi | linux-asahi |
| GPU / DCP / touchpad | TBA | TBA | TBA |
| Boot framebuffer | linux-asahi (7.4) | linux-asahi (7.4) | linux-asahi (7.5) on the laptop row |
| Wi-Fi and Bluetooth | TBA | linux-asahi | linux-asahi |

No M4-class device is an install target. `external_usb_boot_supported` and `bridge_reachable` are false, including `T8132`.

## M5

| | M5 `T8142` (MacBook Air 2026, MacBook Pro 14-inch 2025) | M5 Pro/Max/Ultra column (`T6050`) |
|---|---|---|
| Installer | no | no |
| Devicetree | linux-asahi (7.5) | TBA |
| Boot framebuffer | linux-asahi (7.5); full display TBA | TBA (laptop rows); desktops have no main-display cell |
| NVMe | linux-asahi | TBA |
| Keyboard | linux-asahi | TBA |
| Keyboard backlight | linux-asahi (7.5) | TBA |
| USB2, USB3, Thunderbolt, PCIe, DART, GPU, Wi-Fi, Bluetooth, touchpad | TBA | TBA |
| PMU | WIP | TBA |

`T8142` is not an install target. Both capability bits are false. The device list also uses `T6051` for some M5 Max boards (14-inch and 16-inch MacBook Pro, Mac Studio). The feature table does not give `T6051` its own column; those products sit in the Pro/Max/Ultra device table, which is still `TBA`. The adapter maps `T6051` to that same unpublished profile.

## What SBM reuses

m1n1 and Asahi U-Boot stay upstream. SBM adds a capability adapter, partition protection, and an ESP file for boot policy because UEFI variables are absent. It does not add a second boot manager. The shared `aarch64` UEFI application, once `sbm-uefi` builds one, is the `\EFI\BOOT\BOOTAA64.EFI` payload. The probe in `sbm-as` only checks the protocols that payload needs. See `docs/hardware/apple-silicon-boot-architecture.md` and `apple-silicon-external-boot.md`.
