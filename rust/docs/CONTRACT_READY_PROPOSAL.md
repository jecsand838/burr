# Run report contract (canonical)

This document defines the **golden run report** format produced by both engines and compared by the golden harness.

## Canonical files

- Schema: `rust/spec/schema/run-report.schema.json`
- Normalizer base config: `rust/spec/normalizer/v1.yaml`
- Goldens: `rust/spec/goldens/<scenario_id>.json`

## Versioning

Top-level key:
- `run_report_version: "burr.run_report/v1"`

This string is a stable contract identifier for the golden harness.

## Required sections

### `environment.paths`

Must include absolute paths (the normalizer will rewrite them):

- `repo_root`
- `run_dir`
- `home_dir`
- `temp_dir`

These are required so `replace_prefixes_from_report` can reliably stabilize
machine-specific paths.

### `scenario`

Must include:

- `id` (scenario_id)
- `spec_version` (scenario spec version string)
- `path` (scenario file path used for execution)
- `sha256` (sha256 of the scenario JSON content)

### `runs[*].engine`

Each run includes engine metadata:

- `kind`: `"python"` or `"rust"`
- `version`: optional (e.g., python package version, git sha, etc.)

### `runs[*].calls[*]`

Each executed call records:

- `op`
- `ok` (boolean)
- `exception` (if any)
- `return` (return payload for the call)
- optional `events` and state snapshots, as needed for parity

## Artifacts

`artifacts.tracking` and `artifacts.persistence` are optional sections that can embed
or reference additional run artifacts (e.g., local tracking files or a SQLite DB path).

## Tracking server / UI parity (separate, pinned contract)

The Burr UI uses an OpenAPI-generated client. The Rust server parity plan is:
- snapshot the Python server OpenAPI document into `rust/spec/openapi/python_openapi.json`
- add Rust contract tests to keep UI unchanged by matching the OpenAPI contract

See `rust/spec/openapi/README.md` and `rust/RUST_IMPLEMENTATION_PLAN.md`.
