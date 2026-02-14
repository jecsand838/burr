# Development loop (golden-driven parity)

This project uses **characterization (golden) tests**: we treat the Python Burr engine as the reference, record normalized outputs, then implement Rust until it matches.

## Adding a new scenario

1) Add `rust/spec/scenarios/<scenario_id>.json`
2) (Optional) Add `rust/spec/scenarios/<scenario_id>.normalize.yaml` for per-scenario normalization overrides
3) Generate goldens with the Python reference engine:

```bash
cargo run --manifest-path rust/Cargo.toml -p burr-golden-runner --bin burr-golden --   run --engine python --scenario <scenario_id> --update --run-dir rust/.runs
```

4) Run comparisons against the Rust engine:

```bash
cargo run --manifest-path rust/Cargo.toml -p burr-golden-runner --bin burr-golden --   run --engine rust --scenario <scenario_id> --run-dir rust/.runs
```

## Updating normalization

- Base rules live in `rust/spec/normalizer/v1.yaml`.
- Scenario overrides live next to the scenario JSON as `<scenario_id>.normalize.yaml`.

The normalizer is the only place we should “stabilize” nondeterminism (UUIDs, timestamps, absolute paths, nondeterministic ordering, etc.).

## Useful dirs

- `rust/spec/goldens/`: committed normalized run reports (`<scenario_id>.json`)
- `rust/.runs/`: local run artifacts (ignored by git)
