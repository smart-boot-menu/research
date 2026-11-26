# Remote Management & Out-of-Band Ecosystem Research

**Last verified:** 2026-11-26  
**Scope:** Remote desktop, mesh networking, P2P NAT traversal, hardware IP-KVM, and enterprise out-of-band (BMC/AMT) protocols evaluated for SBM (`sbm-core`, `sbm-server`, `sbm-agent`).

---

## Summary Comparison Matrix

| Project | Primary Language(s) | License (SPDX) | Latest Verified Status (Nov 2026) | Verdict | Integration Strategy |
|---|---|---|---|---|---|
| **RustDesk** | Rust, Flutter/Dart | `AGPL-3.0-only` | Very active (`rustdesk/rustdesk`, `hbbs`/`hbbr`) | **Optional Wrap / Study** | Do NOT link into MIT/Apache-2.0 crates (AGPL-3.0); optionally launch/integrate as an external remote-desktop tool or study video/input pipeline |
| **noVNC** | JavaScript / HTML5 | `MPL-2.0` | Active (`novnc/noVNC`) | **Reuse** (Web Dashboard) | Embed `@novnc/novnc` in `sbm-server` web dashboard over authenticated WSS proxy to `wayvnc` inside `sbm-core` |
| **MeshCentral** | Node.js / JavaScript | `Apache-2.0` | Community-maintained (Intel ended funding 2023) | **Study** | Study agent enrollment, terminal/desktop multiplexing, and Intel AMT integration |
| **Tailscale** | Go | `BSD-3-Clause` (client) / Proprietary (control plane) | Very active | **Optional Integration** | Allow users to connect `sbm-core` / `sbm-server` over an existing Tailscale network; never make it mandatory |
| **Headscale** | Go | `BSD-3-Clause` | Very active | **Optional Integration** | Self-hosted Tailscale control plane; great optional deployment companion |
| **WireGuard** | C (Linux kernel), Rust (`boringtun` `0.7.1`, `gotatun`) | `GPL-2.0-only` (kernel) / `BSD-3-Clause` (`boringtun`/`gotatun`) | Very active (`gotatun` deployed by Mullvad Jun 2026; `boringtun` `0.7.1` May 2026) | **Reuse** | Kernel WireGuard in `sbm-core` (optional VPN transport); `gotatun`/`boringtun` if userspace WG ever needed |
| **Iroh (`n0-computer/iroh`)** | Rust | `MIT OR Apache-2.0` | Very active (2026) | **Evaluate / Optional Phase 2** | Pure-Rust QUIC P2P + NAT hole-punching + relay dialed by Ed25519 `EndpointId` — ideal optional NAT-traversal transport without centralized SaaS |
| **JetKVM** | Go, TypeScript, C | `MIT` (software) / Open HW | Very active (BuildJet) | **Optional Integration** | Integrate optional JetKVM webhook/API links from `sbm-server` for true pre-OS out-of-band BIOS/UEFI video+HID |
| **PiKVM (`kvmd`)** | Python, C | `GPL-3.0-or-later` | Very active (`v4.238`, Nov 2026) | **Optional Integration** | Integrate via PiKVM HTTP/WebSocket API for ATX power + HDMI/USB HID OOB control |
| **Intel AMT / vPro** (`Device Management Toolkit`) | Go, TypeScript (replaced Open AMT Cloud Toolkit; MeshCommander inactive) | `Apache-2.0` | Active under `device-management-toolkit` | **Optional Phase 3** | Support optional AMT power/SOL/KVM adapter on enterprise vPro hardware |
| **DMTF Redfish** | Open REST/JSON Standard | Standard (DSP0266) | **Release 2026.2** (`DSP0266 v1.25.0`, Aug–Oct 2026) | **Support (Phase 2/3)** | Implement clean Rust Redfish client in `sbm-server` for server BMC power/boot-override/virtual-media control |
| **IPMI** | Legacy BMC Standard | Standard (deprecated in favor of Redfish) | Legacy / insecure cipher history | **Avoid** (use Redfish instead) | Avoid building new IPMI dependencies; recommend Redfish |
| **RustKVM** (`Rust-KVM/rustkvm`) | Rust (2024 ed.) + Leptos WASM | `GPL-2.0-only` | Active (2025–2026) | **Study** | Interesting pure-Rust RK3588 IP-KVM reference |

---

## Architectural Synthesis for SBM Remote Management

1. **Zero-Dependency Local Operation First:** SBM MUST work 100% locally with no `sbm-server`, no cloud account, and no internet connection.
2. **Primary Control Channel (`sbm-uefi`, `sbm-core`, `sbm-agent` ↔ `sbm-server`):**
   - **HTTPS (TLS 1.3) + WebSockets** (and optional HTTP/3 QUIC in future phases) using mutual authentication (device Ed25519 identity + signed command envelopes).
   - Works cleanly through corporate proxies, NATs (outbound device-initiated connections), and standard reverse proxies (Caddy/Nginx/Traefik).
   - For `sbm-uefi` (fast path): **strictly read-only policy fetch + boot-attempt reporting** over short bounded timeouts (e.g., 2–3 seconds max); **NO arbitrary remote code execution or shell in UEFI**.
3. **Remote Desktop & Terminal in `sbm-core` (Management Path):**
   - **Browser-native zero-client remote desktop:** Run `wayvnc` (ISC license) on `sbm-core`'s Wayland compositor, tunneled over the device's authenticated `sbm-server` WSS connection to **`noVNC` (`MPL-2.0`)** embedded in the `sbm-server` web dashboard. Zero client software required; clean licensing (`ISC` + `MPL-2.0`, avoids `AGPL-3.0` contamination).
   - **Browser-native remote terminal:** Rust PTY multiplexer in `sbm-core` streamed over the same authenticated WebSocket to `xterm.js` in the web dashboard.
   - **Optional RustDesk / Hardware IP-KVM:** Power users can optionally enable RustDesk (as an external module) or attach PiKVM / JetKVM / Redfish BMC adapters to `sbm-server` for true pre-UEFI power-cycle and BIOS setup access.
4. **Optional NAT Traversal / P2P (Phase 2):**
   - **Iroh (`MIT OR Apache-2.0`)** is a standout Rust-native option for optional direct P2P QUIC sessions (e.g., direct high-bandwidth remote desktop or disk backup streaming between operator and `sbm-core` without routing video through `sbm-server`), while standard self-hosted HTTPS/WSS remains the zero-magic default.
