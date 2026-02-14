# Rust Burr (workspace)

This directory contains the Rust-native implementation of Apache Burr, developed **in-repo** alongside the Python reference implementation.

## Quick start (from repo root)

Rust workspace tests:

```bash
cargo test --manifest-path rust/Cargo.toml
```

Golden harness (once the crates exist):

```bash
# list scenarios
cargo run --manifest-path rust/Cargo.toml -p burr-golden-runner --bin burr-golden -- list

# (re)generate goldens using Python reference engine
cargo run --manifest-path rust/Cargo.toml -p burr-golden-runner --bin burr-golden --   run --engine python --all --update --run-dir rust/.runs

# compare Rust engine output against goldens
cargo run --manifest-path rust/Cargo.toml -p burr-golden-runner --bin burr-golden --   run --engine rust --all --run-dir rust/.runs
```

Python golden engine unit tests (once implemented):

```bash
python3 -m unittest discover -s rust/tools/tests -v
```

## Canonical specs (do not fork these)

- Scenario schema: `rust/spec/schema/scenario.schema.json`
- Run report schema: `rust/spec/schema/run-report.schema.json`
- Normalizer config: `rust/spec/normalizer/v1.yaml`

The goal is: **Python engine generates run reports → normalize → commit goldens → Rust engine matches goldens**.
