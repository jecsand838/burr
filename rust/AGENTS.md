# Codex project instructions (Rust Burr)

This repository is `apache/burr`. The Rust workspace lives under `./rust/`.

## The one canonical contract set

**Authoritative specs** live under `rust/spec/`:

- Scenario input: `rust/spec/schema/scenario.schema.json` (spec version: `burr.scenario/v1`)
- Run report output: `rust/spec/schema/run-report.schema.json` (version: `burr.run_report/v1`)
- Normalizer config: `rust/spec/normalizer/v1.yaml` + overrides `rust/spec/scenarios/<scenario_id>.normalize.yaml`
- Tracking server contract snapshot (later): `rust/spec/openapi/python_openapi.json`

If any other file disagrees with those schemas, update the file to match the schemas (do not create competing formats).

## Non‑negotiable workflow rules

1) **TDD first**: write/extend tests (or golden scenarios) before implementing runtime code.
2) **Python Burr is the reference** for behavior. Characterize with goldens first, then implement Rust until goldens pass.
3) **Determinism comes from the normalizer**, not from “hiding” values in the engines.
4) Keep changes **scoped**:
   - Do not modify existing Python Burr runtime unless explicitly asked.
   - It is OK to add `rust/tools/*` helper scripts and their tests.

## Commands Codex should run

From repo root:

- Rust: `cargo test --manifest-path rust/Cargo.toml`
- Single crate: `cargo test --manifest-path rust/Cargo.toml -p <crate>`
- Rust formatting: `cargo fmt --manifest-path rust/Cargo.toml`
- Rust linting: `cargo clippy --manifest-path rust/Cargo.toml -p <crate> -- -D warnings`
- Python golden engine tests: `python3 -m unittest discover -s rust/tools/tests -v`

Golden loop (once the runner exists):

- Generate/update goldens (python reference):  
  `cargo run --manifest-path rust/Cargo.toml -p burr-golden-runner --bin burr-golden -- run --engine python --all --update --run-dir rust/.runs`
- Compare rust engine to goldens:  
  `cargo run --manifest-path rust/Cargo.toml -p burr-golden-runner --bin burr-golden -- run --engine rust --all --run-dir rust/.runs`

## Output expectations

- Prefer small, reviewable diffs.
- Keep folder structure stable and names consistent with `rust/RUST_IMPLEMENTATION_PLAN.md`.
- Summarize changed files + how to run tests at the end of each step.
