# CLI parity mapping (Python → Rust)

This document is the **living map** from the existing Python CLI behavior to the Rust CLI we build in `rust/crates/burr-cli`.

> Integrations are out of scope for initial parity; this focuses on core runtime + tracking server/UI + golden tooling.

## Golden harness

| Python | Rust |
|---|---|
| (n/a) | `burr golden list` |
| (n/a) | `burr golden run --engine python|rust --scenario <id>|--all [--update]` |
| (n/a) | `burr golden normalize --input <file> [--scenario <id>]` |

## Tracking server / UI

| Python | Rust |
|---|---|
| `burr server` (tracking UI/backend) | `burr server` (Rust backend; UI unchanged) |

## Telemetry / usage analytics

Python Burr includes anonymous usage analytics by default, with opt-out via code, config file, or env var.
Rust Burr should preserve the same opt-out semantics and document any differences.
