# CI and admission fixes, 2026-11-26

Diagnosis of the two failing GitHub Actions runs. This note does not replace the wave 2 integration audit.

## sbm-uefi run 45900855497

The check job failed `cargo fmt` on `crates/xtask/src/qemu.rs`. The UEFI release lint and build did not run. The formatted join, plus a cancel path that SIGKILLs only the QEMU pid xtask started, is the host-side change. xtask still reserves `efi` at 1024 MiB and 1 vCPU.

## sbm-lab run 45900849263

The check job passed. The boot jobs did not.

- x86_64 smoke, both x86 product menus, and `secure-signed-probe` passed.
- `aarch64-efi-launch` and `aarch64-chainload` ran to the 720 s floor. `sbm-uefi-menu-aarch64` ran to about 1200 s. Each serial log contained only `UEFI firmware (version 2025.11-3ubuntu7.4 built at 12:58:05 on Oct  8 2026)`. `qemu.stderr` was empty. The argv used `-cpu max`, `-m 1024`, `-smp 1`, and TCG. `-cpu max` enables pointer authentication, and that AAVMF image does not get past the version line under cross-architecture TCG. Cross-architecture aarch64 TCG now requests `cortex-a57`. Same-architecture TCG stays on `max`.
- `secure-unsigned-rejected` and `secure-tampered-rejected` printed `Access Denied` in about 3.7 s. The harness still stored `guest did not emit efi-launched` because the Ubuntu line is not an `edk2-` banner. A denial scenario that does not expect `efi-launched` now clears that one failure when its assertions pass.

## Ledger

The 2× guest footprint and the Mac rows are unchanged: 16384 MiB normal is 5120 MiB of guest RAM, warning plus swap 1968/3072 is 3584 MiB, and critical is 0. The 6144 MiB floor now starts at 15360 MiB so a Linux `MemTotal` near 15999 MiB still admits a 4096 MiB build guest. Missing `MemAvailable` is unknown pressure. A finite cgroup memory limit caps the host total.

Reservation ids no longer collide inside one process in the same second. The owner recorded by the shell helper is `$$` (or the helper's parent when the helper is a short-lived process). Kill and reap consult `ps -o lstart=` so a reused pid is not the recorded process. `resources run` does not count its own argv as a QEMU process. SIGINT and SIGTERM kill only the QEMU pid that process started. Graphical recovery scenarios were not changed.
