# Normalizer config

- Base config: `v1.yaml` (version: `burr.normalizer/v1`)
- Scenario overrides: `rust/spec/scenarios/<scenario_id>.normalize.yaml` (version: `burr.normalizer_override/v1`)

The normalizer is responsible for stabilizing:
- machine-specific paths
- UUIDs
- timestamps
- nondeterministic ordering (via `sort_array`)
