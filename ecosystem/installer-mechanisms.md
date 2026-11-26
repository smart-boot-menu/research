# Installer mechanisms for the OS catalog

Date: 2026-11-26. This note decides what `sbm-catalog` 0.1 records. The broader survey is [os-provisioning.md](os-provisioning.md). Booting an installer and installing an operating system are different steps. The catalog milestone does the first and only names the second.

## Stable enough to name, not to run

These projects publish an unattended interface that a later `sbm-core` recipe can target. The catalog stores the name and `stable`. It does not generate the answer file or execute it.

| Mechanism | What it automates | Why it is not run yet |
| --- | --- | --- |
| Ubuntu autoinstall | Subiquity answer file on the live server ISO | Needs a bound disk and cloud-init media |
| Debian preseed | debian-installer answers | The interface is old and still shipped; the catalog has no preseed runner |
| Fedora Kickstart | Anaconda | Same: data only |
| openSUSE AutoYaST | YaST | Same: data only |

## Experimental or incomplete

| Mechanism | Catalog mark | Reason |
| --- | --- | --- |
| `archinstall` | experimental | Guided Python installer, not a frozen answer-file contract |
| NixOS declarative install | experimental | The ISO boots; the install closure is a separate evaluation |
| FreeBSD `installerconfig` | experimental | `bsdinstall` is the supported installer; the scripted config is not executed |
| Windows `autounattend` and WIM apply | experimental | `wimlib-imagex apply` can copy a Windows image. Creating the boot configuration that Windows Setup would write is still open, so the catalog does not apply a WIM |
| Windows Setup / WinPE | not executed | Consumer ISOs are user-supplied. No Windows image is mirrored |

## Boot an image, do not adopt the project

| Project | Decision |
| --- | --- |
| PINN, netboot.xyz, iPXE | Useful references for menu and network boot. Not embedded. |
| NOOBS | Superseded. Not integrated. |
| Ventoy and iVentoy | Not embedded. Catalog images stay individual verified ISOs. |
| MAAS, Tinkerbell, Foreman, Ironic | Server provisioning systems. Not a catalog dependency. |
| Calamares | GPL/Qt installer used by some desktops. Not linked into the catalog. |
| The Update Framework | Implemented for catalog metadata with `tough` 0.24.0. See `sbm-catalog/docs/TUF.md`. Image authenticity still depends on each vendor's signature, which is only checked where a pinned key and a signed file were actually verified. |

## Architectures

Generic `aarch64` UEFI is not Apple Silicon. Apple Silicon needs an explicit `m1n1` plus U-Boot artifact and the platform `apple-silicon-m1n1-uboot`. No current catalog image has that artifact. Arch Linux ARM, Windows on Arm, and the extra Ubuntu and openSUSE architectures that were visible in checksum files are not pinned.

## Lab result

Debian 13.8.0 arm64 netinst was the installer used for the boot demonstration because its hash matched retrieved signed-metadata sums and the host can run `qemu-system-aarch64` with HVF. The log is [the 2026-11-26 evidence note](../validation/evidence/2026-11-26-debian-13.8.0-arm64-netinst-qemu.md). That virtual machine is generic UEFI. It does not test Apple Silicon firmware.
