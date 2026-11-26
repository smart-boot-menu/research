# `sbm-server` stack re-verification (2026-11-26)

Agent 05 re-checked the ADR-007 / `TECHNOLOGY_STACK.md` pins against crates.io and the npm registry on 2026-11-26 before writing the server. These are registry facts, not performance measurements.

| Component | Pin used by `sbm-server` | Registry fact on 2026-11-26 |
|---|---|---|
| `axum` | `0.8.9` | `max_stable_version` `0.8.9` |
| `tokio` | `1.54.1` | `max_stable_version` `1.54.1` |
| `sqlx` | `0.9.1` | `max_stable_version` `0.9.1` |
| `rustls` | `0.23.46` | `max_stable_version` `0.23.46` (`0.24.0-dev.3` exists and was not used) |
| `rcgen` | `0.14.10` | `max_stable_version` `0.14.10` |
| `webauthn-rs` | `0.6.2` | `max_stable_version` `0.6.2` |
| `ed25519-dalek` | `3.0.0` | `max_stable_version` `3.0.0` |
| `@sveltejs/kit` | `3.4.0` | Published. Newer `3.4.1` exists; the server pins the ADR version |
| `svelte` | `5.59.2` | Current stable on that date; ADR names SvelteKit, not a Svelte patch |
| `@sveltejs/adapter-static` | `4.0.1` | Current release whose peer dependency is `@sveltejs/kit` `^3` |
| `@novnc/novnc` | `1.7.0` | Current stable, license `MPL-2.0` |
| `@xterm/xterm` | `6.0.0` | Package name moved from `xterm`; `6.0.0` is the 2026 line |

`rcgen` 0.14 signs a certificate for an external subject public key (`CertificateParams::signed_by(&impl PublicKeyData, &Issuer)`). That matches Spec 06: the device generates the Ed25519 key, and the server returns only `client_cert_pem`.

The SvelteKit dashboard is compiled to static files and embedded in the Rust binary. It does not call out to a hosted frontend platform.

Local toolchain used for the implementation: `rustc 1.100.0`, `cargo 1.100.0`, Node `v26.9.0`.
