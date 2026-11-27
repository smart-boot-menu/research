# Agent 15 UEFI launch and handoff evidence

**Verified locally on 2026-11-27.** See the [machine-readable samples and provenance](sbm-uefi-agent15-2026-11-27.json).

## ARM64 real recovery media

The measurements used QEMU 11.1.3 with AAVMF and HVF on an Apple M5 Mac with 16 GiB RAM and macOS 27.0. The benchmark guest had one vCPU, 1,024 MiB RAM, 1280×800 virtio graphics, USB recovery media and no NIC. Each cold boot used a fresh copy of the firmware variables and a disposable disk overlay. The dual-architecture comparison used identical media contents apart from the picker EFI. The final single-architecture runs used freshly assembled media from the committed picker and a clean Core build.

| Image and EFI build | Boot samples | First frame, median ms | Discovery complete, median ms | Menu ready, median ms | Menu ready, min–max ms | Selected-image verification, median ms |
|---|---:|---:|---:|---:|---:|---:|
| Existing dual-arch medium, baseline `653012f` | 10 | 9.0 | 2,067.0 | 2,073.8 | 2,054.3–2,990.0 | — |
| Existing dual-arch medium, candidate `fd857b9` with SHA-256 at opt 3 | 10 | 8.9 | 9.8 | 16.2 | 14.3–26.7 | 393.5 |
| Existing dual-arch medium, same candidate with SHA-256 at opt z | 10 | 9.3 | 10.3 | 16.8 | 14.3–19.1 | 615.5 |
| Fresh ARM64 medium, picker `664a0b3` and Core `c2020c7` | 10 | 8.0 | 8.9 | 15.7 | 14.4–17.0 | 362.0 |

The first three timestamps are elapsed from UEFI entry. The verification interval measures the sequential read and SHA-256 of 74,513,752 bytes in 72 reads through one 1 MiB buffer after choosing Manage Computer. The baseline computed the digests during startup. The matched comparison showed a 222.0 ms median launch improvement at a cost of 5,120 EFI bytes for the SHA-256 setting. The final EFI sizes are 230,912 bytes for ARM64 and 261,632 bytes for x86-64.

On a separate media copy with a single-byte UKI change, the picker reached menu readiness and refused recovery with a digest mismatch. The log showed no handoff publication or chainload. The original artifacts stayed unchanged. [Accepted boot menu frame](screenshots/2026-11-27-agent15/aarch64-uefi-menu.png)

## Firmware and graphical integration

- The ARM64 GPT smoke fixture checked the exact partition UUID, all five volatile variables, equality of the handoff and attempt IDs, the handoff SHA-256, and empty recovery load options inside the launched image.
- The rebuilt ARM64 image passed all nine local graphical integration checks in an offline HVF guest with two vCPUs and 4,096 MiB RAM. Core reported a verified integrated handoff, opened GParted, rebooted, and confirmed the exact attempt. The decoy ESP and original inputs stayed unchanged. The first and second menu-ready markers were 16.115 ms and 23.324 ms.
- Native x86-64 KVM passed the OVMF smoke with the synthetic recovery payload. The fixture verified the volatile variables and empty load options. The first menu-ready marker in this run was 64.767 ms. [Native x86-64 OVMF CI](https://github.com/smart-boot-menu/sbm-uefi/actions/runs/45967541247)
- Under enrolled Secure Boot, the signed picker verified the unsigned recovery fixture against its manifest, then OVMF denied the image load with `ACCESS_DENIED`. The harness asserted that the fixture never started. [Secure Boot CI and JSON artifacts](https://github.com/smart-boot-menu/sbm-uefi/actions/runs/45967541247)
- The native scenario suites exercise bound one-shot selection, consumption, local override, absent and ambiguous targets, multiple loaders, invalid handoff, stale success, corrupted recovery and simulated firmware rejection. [Core CI](https://github.com/smart-boot-menu/sbm-core/actions/runs/45966525283)

Firmware rereads the EFI image from its exact device path when loading it. With Secure Boot enabled, firmware enforces signature trust on the loaded bytes.

Independent release qualification remains open. The final x86-64 real-media timing and graphical integration, per-architecture reproducibility, and remaining negative release checks belong to qualification.
