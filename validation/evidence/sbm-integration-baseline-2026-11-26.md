# SBM 0.0.1 integration baseline — 2026-11-26

## Hosted CI and builder

| Repository | Commit | Run and job | Outcome |
|---|---|---|---|
| sbm-core | 7f389cb293fa55cd4448c6705d085f759025cf26 | [45887012618 / 140140116178](https://github.com/smart-boot-menu/sbm-core/actions/runs/45887012618/job/140140116178) | Linux check failed before tests |
| sbm-lab | 4c4ba5d22e7bd42cc6c5f82fd46f2da0d0e318f9 | [45879871319 / 140116202746](https://github.com/smart-boot-menu/sbm-lab/actions/runs/45879871319/job/140116202746) | x86_64 probe passed; both AArch64 probes missed the first expected marker |
| sbm-uefi | 52657522ac734c2aec064130f0a91fcbe79b8b51 | [45884152684](https://github.com/smart-boot-menu/sbm-uefi/actions/runs/45884152684) | Host checks and EFI cross-builds passed |
| sbm-server | 4840ee3118083c1d054ad1f0d8839286d5cb4576 | [45886593960](https://github.com/smart-boot-menu/sbm-server/actions/runs/45886593960) | Server and simulator checks passed |
| sbm-catalog | 820aa333061446809f6190b56d25f3ae957e8261 | [45810920164](https://github.com/smart-boot-menu/sbm-catalog/actions/runs/45810920164) | Catalog checks passed |
| sbm-as | f0f81fb20cb3d71950aacb45b8507da49ab9b7fe | [45811513557](https://github.com/smart-boot-menu/sbm-as/actions/runs/45811513557) | Apple adapter checks passed |
| sbm-common | 28d38450954f0b93815e12f31da92370b13a7462 | [45802294888](https://github.com/smart-boot-menu/sbm-common/actions/runs/45802294888) | Shared checks passed |

Artifact list queries returned zero entries for these runs. The Core Linux job and the existing Fedora builder log both reported E0658 at the peer-credential call and E0599 at the UID method accesses. Selected builder output follows.

```text
error[E0658]: use of unstable library feature `peer_credentials_unix_socket`
   --> crates/sbm-cored/src/lib.rs:886:18
886 |     match stream.peer_cred() {
error[E0599]: no method named `uid` found for struct `UCred` in the current scope
   --> crates/sbm-cored/src/lib.rs:887:26
error: could not compile `sbm-cored` (lib) due to 3 previous errors
```

The builder had no build/images directory. The inspected builder disk is distinct from the release image. The hosted smoke output reported QEMU 1:10.2.1+ds-1ubuntu3.3 and qemu-efi-aarch64 2025.11-3ubuntu7.4. The workflow did not upload per-guest logs.

```text
x86_64-efi-launch    success=true
aarch64-efi-launch   success=false  missing marker efi-launched
aarch64-chainload    success=false  missing marker efi-launched
```

## Local checks and artifact observations

Host Rust was 1.100.0 on AArch64 macOS. Locked offline host tests passed with 6 tests for common, 23 for UEFI, 17 for Core, 25 for Lab, 11 for Server, 15 for Catalog, and 17 for the Apple adapter. Core's host test result excludes the failing Linux implementation. Server integration exercises sbm-sim.

```text
shasum -a 256 sbm-uefi/artifacts/*.efi
c34c4ada6148bc59fc572a9f39dcd02f37eb5e8120b1e00271f1f7bb9e2ed184  sbm-uefi/artifacts/sbm-aarch64.efi
ae89503dc7fd84405c30ed80b684f1175b91258637e5a50906adaffbdd43dbb5  sbm-uefi/artifacts/sbm-fixture-aarch64.efi
3a0255e83d8466bf05519690572197541d3643989679753bd0a2be99ea705fb7  sbm-uefi/artifacts/sbm-fixture-x86_64.efi
8c09f37ff91f83e460a46a155a31ead566efc3275b5c144c8abd848d40b5bbfb  sbm-uefi/artifacts/sbm-x86_64.efi
```

The PE headers identify the first two as AArch64 and the last two as x86_64. The archived UEFI menu chainload used EFI fixtures. This baseline contains no combined-image qualification run.
