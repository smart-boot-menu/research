# Boot Security, Supply Chain & Server Auth Research

**Last verified:** 2026-11-26  
**Primary Sources:** `rhboot/shim`, Microsoft Secure Boot 2023 CA transition guidance, TPM 2.0 / BitLocker PCR specifications, TUF v1.0.33, Sigstore/SLSA, Rust `axum`/`sqlx`/`webauthn-rs`/`rcgen`/`rustls` crates.

---

## 1. UEFI Secure Boot & The 2026 Microsoft Certificate Transition

### Critical 2026 Reality: Microsoft 2011 Certificates Expired in June 2026
As of **November 2026**, Microsoft's original 2011 Secure Boot certificates have reached expiration:
- **Microsoft Corporation KEK CA 2011** (expired June 24, 2026) -> replaced by **`Microsoft Corporation KEK 2K CA 2023`**.
- **Microsoft Windows Production PCA 2011** (expired October 2026) -> replaced by **`Windows UEFI CA 2023`**.
- **Microsoft Corporation UEFI CA 2011** (third-party / shim signing CA, expired June 2026) -> replaced by **`Microsoft UEFI CA 2023`**.

### Implications for Smart Boot Menu (`sbm-uefi` & `sbm-core`)
1. **Do NOT assume machines only have 2011 keys OR only have 2023 keys:** Older hardware (like an older x86 HP laptop) may still carry 2011 keys in `db` unless firmware/Windows updated them; newer 2025–2026 PCs carry **2023 CAs**.
2. **Secure Boot Deployment Path for SBM:**
   - **Development / SBM 0.1:** Support **Custom Secure Boot Keys** (user-enrolled PK/KEK/db or MOK) and Secure Boot disabled in QEMU/OVMF lab tests, plus automated OVMF Secure Boot testing with SBM test keys.
   - **Production Distribution (Post-0.1):** Ship **`rhboot/shim` 16.1+** (August 2025 baseline, still current standard for `rhboot/shim-review` in 2026) signed by **Microsoft UEFI CA 2023** (requiring formal `shim-review`, reproducible builds, **SBAT** `.sbat` section in `sbm-uefi.efi`, and **NX compatibility** `MOK_POLICY_REQUIRE_NX`!).
   - **`sbm-uefi` MUST include a valid `.sbat` PE section and `IMAGE_DLLCHARACTERISTICS_NX_COMPAT`** from early development so future shim signing is not blocked by architectural debt.
   - **Shim 16.1 UKI support:** Shim 16.1 natively verifies signed Unified Kernel Images (UKIs) via Loader Protocol v2 — allowing `sbm-uefi` to invoke `shim`'s verification protocol before launching `sbm-core.efi`!

---

## 2. TPM 2.0 Measured Boot & BitLocker Recovery Key Hazards

### How BitLocker Breaks When Bootloaders Change
During UEFI boot, the firmware extends hashes into TPM 2.0 PCRs (`PCR_new = H(PCR_old || digest)`):
- **PCR 0:** Core firmware code & CPU microcode.
- **PCR 4:** Boot manager code & executed EFI applications (`sbm-uefi.efi`, `shim`, `bootmgfw.efi`).
- **PCR 7:** Secure Boot state, PK/KEK/db/dbx variables, and the **signing authority** that validated the executed EFI binaries.
- **PCR 11:** BitLocker access control (and `systemd-stub` UKI measurements on Linux).

**Windows BitLocker Default Validation Profile (UEFI + Secure Boot):**
- Seals the Volume Master Key (VMK) to **PCR 7 + PCR 11** (or **PCR 0, 2, 4, 11** when Secure Boot is disabled or non-standard).
- **CRITICAL WARNING FOR `install` AND `sbm-uefi`:**
  - Inserting SBM into the boot path (`BootOrder` -> `sbm-uefi` -> `bootmgfw.efi`) **WILL alter PCR 4** and (if Secure Boot keys/MOK/shim authorities change) **WILL alter PCR 7**, triggering a **BitLocker 48-digit Recovery Key prompt on Windows reboot**!
  - **Mandatory SBM Safety Rule:** Before installing SBM or modifying `BootOrder` on any machine with a detected Windows/BitLocker volume, `install` MUST warn the user explicitly, instruct them to verify they have their BitLocker Recovery Key backed up (or suspend BitLocker for one reboot via `manage-bde -protectors -disable C: -RebootCount 1` when installing from Windows), and require explicit confirmation.

---

## 3. Software Supply-Chain & Update Verification

| Standard / Crate | Verified 2026 Version | Role in SBM |
|---|---|---|
| **TUF (The Update Framework)** | Spec `v1.0.33`; Rust `tough` `v0.24.0` (Jul 2026) | Canonical trust framework for `sbm-catalog` metadata, SBM self-updates (`install`), and `sbm-core` A/B updates — protects against rollback, freeze, mix-and-match, and compromised mirror attacks |
| **Ed25519 / Detached Signatures (`ed25519-dalek`)** | `2.x` (`no_std` compatible!) | Compact, fast signature verification usable inside `sbm-uefi` (`no_std`) for remote command envelopes and signed policy/manifest validation |
| **Sigstore / Cosign & SLSA Level 3** | Active 2026 | CI release signing and build provenance attestation for GitHub Releases artifacts |
| **Reproducible Builds** | `SOURCE_DATE_EPOCH`, `--remap-path-prefix`, pinned `Cargo.lock`, deterministic `mkosi` builds | Required both for user trust and future `rhboot/shim-review` submission |

---

## 4. `sbm-server` Stack & Cryptographic Identity (November 2026)

### Recommended Rust Backend Stack (Verified Versions)
- **Web/API Server:** **`axum 0.8.9`** + **`tokio 1.54.1`** (`MIT`) — idiomatic, high-performance async Rust with first-class WebSocket support.
- **Database:** **`sqlx 0.9.1`** (Nov 2026; MSRV 1.94; note split feature flags `runtime-tokio` + `tls-rustls`) supporting:
  - **SQLite (WAL mode)** as the zero-configuration default for home/homelab self-hosting (single binary + single DB file!).
  - **PostgreSQL** as an optional scale-out backend for multi-operator / lab / enterprise deployments.
- **Web Dashboard Frontend:** **`SvelteKit 3.4.0`** (TypeScript; SvelteKit 3.0 released Oct 2026) compiled to static assets embedded in `sbm-server` (or served alongside).
  - *Why SvelteKit over Leptos (`0.8.23`) / Dioxus (`0.7.10`)?* Trivial integration with `@novnc/novnc` (remote desktop) and `xterm.js` (remote terminal), small JS bundle (vs multi-MB WASM), fast UI iteration, and strong ecosystem for complex forms/tables. Backend + protocol types remain 100% Rust via JSON Schema / OpenAPI generated from `sbm-common`.
- **Operator Authentication:** **Passkeys / WebAuthn (`webauthn-rs 0.6.2`, Oct 2026)** as primary passwordless MFA auth, plus optional OIDC for enterprise SSO, and scoped RBAC roles (e.g., `viewer`, `boot-operator`, `storage-admin` — strictly separating non-destructive boot selection from destructive disk/install operations!).
- **Device Enrollment & Authentication:**
  - **Explicit opt-in enrollment** using short-lived one-time enrollment tokens (QR code / 8-char code shown in `sbm-core` or typed into `install`) — NEVER auto-enroll devices silently.
  - Device generates an **Ed25519 keypair** (backed by **TPM 2.0** via `tss-esapi` when a TPM is present, or protected local storage otherwise) and receives an **`sbm-server`-issued X.509 client certificate** (`rcgen 0.14.10`, `rustls >= 0.23.46` mTLS).
  - Every remote command is wrapped in a **signed `CommandEnvelope`** (`sbm-protocol`) containing: `command_id` (UUIDv7), `device_id`, `issued_at`, `expires_at`, monotonic `nonce`/`sequence`, `required_scope`, and operator/server Ed25519 signature -> prevents replay attacks, cross-device command injection, and MitM tampering even if TLS terminates at a reverse proxy!
