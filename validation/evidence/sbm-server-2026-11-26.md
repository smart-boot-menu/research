# sbm-server release timings, 2026-11-26

These numbers come from one run of `sbm-server/tools/measure.py` on the development Mac. They are the script's printed JSON. They are not the UEFI frame targets in `docs/testing/benchmarks.md`, and they are not a physical boot.

## Host

- macOS 27.0 (26A428), arm64, Apple M5, 10 cores.
- `rustc 1.100.0 (0ef1bc386 2026-11-09)`.
- Client: Python 3.15.2, `OpenSSL 4.1.0 23 Oct 2026` (`ssl.OPENSSL_VERSION`). Requests go to `https://127.0.0.1:18543`.
- No QEMU. The process is the release server binary on the host.

## Source

- `sbm-server` `4840ee3118083c1d054ad1f0d8839286d5cb4576`. That commit changes only `tools/measure.py`.
- The measured binary is `target/release/sbm-server`, built from the Rust sources in the parent commit `b6b7a4d3df58a10fc8b7a04e0a18d4d27e5a835b`. Those Rust sources are unchanged in `4840ee3`.
- `sbm-common` `28d38450954f0b93815e12f31da92370b13a7462`.
- SQLite in a fresh data directory under `sbm-server/target/measure`. The directory is deleted at the end of the script.

## Method

`tools/measure.py` prints the JSON below. N=10. One cold start is separate.

Startup samples use `time.monotonic` immediately before `Popen` until the first `GET /healthz` returns 200. The loop sleeps 1 ms between attempts. The cold start generates the CA and the server certificate. Each warm start copies that data directory first, so the certificates already exist, and the copy is not inside the timed section.

Request samples are the wall time of one `urlopen` plus `read()`, after the server is already up. The script drops the first 2 requests of each request series. The boot-policy series uses the certificate from `sbm-sim identity` for the device enrolled as `measure-vm`.

The summary uses Python 3 `round(percent/100 * (n - 1))` on the sorted samples. At n=10 that index for p50 is 4, because `round` uses half-to-even. The index for p95 is 9, so p95 equals max.

## Result

Cold start to healthz: 56.477 ms. One sample.

Warm start to healthz, in run order, milliseconds: 43.492, 43.564, 42.345, 42.657, 42.886, 42.562, 42.824, 43.304, 42.678, 43.629.

Script summary: n=10, min 42.345, p50 42.824, p95 43.629, max 43.629.

Healthz request, in run order, milliseconds, after 2 dropped warmups: 1.215, 1.177, 1.174, 1.126, 1.12, 1.089, 1.09, 1.062, 1.056, 7.951.

Script summary: n=10, min 1.056, p50 1.12, p95 7.951, max 7.951.

Boot-policy request, in run order, milliseconds, after 2 dropped warmups: 1.837, 1.8, 1.691, 1.653, 1.684, 1.611, 1.544, 1.514, 1.541, 1.5.

Script summary: n=10, min 1.5, p50 1.611, p95 1.837, max 1.837.

## What this does not say

The server was not under other load that this note measured. One machine, one cold start, and ten warm starts is the whole sample. A later run can differ. Nothing here was measured inside UEFI or `sbm-core`.
