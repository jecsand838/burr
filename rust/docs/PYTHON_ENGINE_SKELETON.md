# Python golden engine (reference executor)

The Python golden engine is the **truth source** for golden characterization tests.
It executes a scenario spec using the existing Python Burr runtime and prints a `burr.run_report/v1` JSON document to stdout.

## Canonical files

- Scenario schema: `rust/spec/schema/scenario.schema.json`
- Run report schema: `rust/spec/schema/run-report.schema.json`
- Script path: `rust/tools/python_golden_engine.py`

## CLI contract (must match the Rust runner)

The golden runner spawns this script and reads **stdout** as JSON.

Required arguments:

- `--scenario <path>`: path to scenario JSON
- `--repo-root <path>`: repo root (for path reporting)
- `--run-dir <path>`: isolated run dir to write artifacts into
- `--no-tracker`: disable tracking (optional)

Notes:

- Do **not** attempt to normalize output inside the Python engine.
  The Rust normalizer is responsible for stability across machines.
- If tracking is enabled, configure `LocalTrackingClient` so it writes under `run_dir`
  (not `~/.burr`) to keep tests isolated.
- If persistence is enabled, use a SQLite DB under `run_dir` and report its path in artifacts.

## Output shape

The script must print a single JSON object to stdout with:

- `run_report_version: "burr.run_report/v1"`
- `environment.paths.{repo_root,run_dir,home_dir,temp_dir}`
- `scenario.{id,spec_version,path,sha256}`
- `runs[0].calls[*]...`
