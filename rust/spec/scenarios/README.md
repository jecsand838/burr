# Golden scenarios (Rust parity suite)

This directory contains **scenario specs** used to generate and validate golden run reports.
Each scenario is a single JSON file describing:
- a graph (actions + transitions + entrypoint),
- an initial state,
- and one or more calls (`step`, `run`, `iterate`, `stream_result`).

## File conventions

- **Scenario id == filename stem**
  - Example: `001_basic_linear_counter.json` has `"scenario_id": "001_basic_linear_counter"`
- **Optional per-scenario normalizer overrides**
  - Place next to the scenario as: `<scenario_id>.normalize.yaml`
  - Example: `006_span_tree.normalize.yaml`
- **Golden files**
  - Generated goldens are written to: `rust/spec/goldens/<scenario_id>.json`
- **Run artifacts**
  - Use `--run-dir rust/.runs` to keep raw + normalized artifacts for debugging.

## Starter pack (first end-to-end loop)

These 5 scenarios are designed to get the initial Python -> golden -> Rust parity loop working quickly:

| Scenario ID | Purpose |
|---|---|
| `001_basic_linear_counter` | `run()` + transition order + `halt_after` stopping at `done` |
| `002_halt_before` | `halt_before` stops *before* an action executes |
| `003_iterate_yields` | `iterate()` yields `(action, result, state)` per step |
| `004_stream_chunks` | streaming action + `stream_result()` container semantics |
| `005_context_capture` | `__context` injection captured into state |

## Generate goldens (Python reference engine)

Run from the repo root:

```bash
for id in 001_basic_linear_counter 002_halt_before 003_iterate_yields 004_stream_chunks 005_context_capture; do \
  cargo run --manifest-path rust/Cargo.toml -p burr-golden-runner --bin burr-golden -- \
    run --engine python --scenario "$id" --update --run-dir rust/.runs; \
done
```

## Compare Rust engine output vs goldens

```bash
for id in 001_basic_linear_counter 002_halt_before 003_iterate_yields 004_stream_chunks 005_context_capture; do \
  cargo run --manifest-path rust/Cargo.toml -p burr-golden-runner --bin burr-golden -- \
    run --engine rust --scenario "$id" --run-dir rust/.runs; \
done
```

These scenarios explicitly target Burr’s documented semantics for `halt_after` / `halt_before` in `run`/`iterate`, and streaming behavior via `stream_result`.

---

## One-liner to generate goldens for the 5 starter-pack scenarios

```bash
for id in 001_basic_linear_counter 002_halt_before 003_iterate_yields 004_stream_chunks 005_context_capture; do cargo run --manifest-path rust/Cargo.toml -p burr-golden-runner --bin burr-golden -- run --engine python --scenario "$id" --update --run-dir rust/.runs; done
```
