# Scenario spec (canonical)

Golden scenarios are **language-neutral inputs** that drive both:

- the **Python reference engine** (truth source) and
- the **Rust engine** (implementation under test)

## Canonical files

- Schema: `rust/spec/schema/scenario.schema.json`
- Scenarios: `rust/spec/scenarios/*.json`
- Optional scenario-local normalizer override: `rust/spec/scenarios/<scenario_id>.normalize.yaml`

## File conventions

- `scenario_id` **must equal** the filename stem.
- `spec_version` is fixed: `burr.scenario/v1`.

## Top-level fields (high level)

- `identifiers` (optional): stable identifiers used for tracking/persistence:
  - `app_id` (string)
  - `partition_key` (string)
- `initial_state`: JSON object used to initialize Burr state.
- `graph`:
  - `entrypoint`: first action name
  - `actions`: map of action name → builtin action spec
  - `transitions`: ordered list evaluated top-to-bottom
- `calls`: a list of execution operations we run against the application:
  - `step`, `run`, `iterate`, `stream_result`
  - async variants exist for later slices: `astep`, `arun`, `aiterate`, `astream_result`

## Builtin-only actions (initial parity scope)

To keep scenarios language-agnostic, we only allow builtin actions in the scenario spec.
Builtin actions are implemented identically in:
- `rust/tools/python_golden_engine.py` (Python reference)
- Rust runtime crates (`burr-core`/`burr-runtime`)

Initial builtin set (see schema for exact shapes):
- `noop`
- `set` (object `updates`)
- `increment` (`field`, optional `by`)
- `append` (`field`, `item`)
- `fail` (`message`)
- `context_capture` (captures `__context` into state)
- `stream_chunks` (yields a fixed list of chunks, used for streaming parity)
- `span_tree` (emits nested span events for tracing parity)

## Conditions

Transitions use a minimal condition language:
- `default` (always true)
- `when` with `equals` map (exact JSON equality)
- `expr` (string expression; implement minimally then grow as needed)
- boolean composition: `and`, `or`, `not`

## Tracking and persistence knobs

Scenarios may include optional `tracking` and `persistence` config blocks to enable:
- local filesystem tracking artifacts
- SQLite persistence for resume/fork semantics

These blocks are still language-neutral: they describe behavior/paths, not code.

See also: `rust/docs/CONTRACT_READY_PROPOSAL.md` for the run report contract.
