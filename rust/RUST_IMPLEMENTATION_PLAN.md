## 0) Parity target and constraints

Per the Rust-implementation proposal in Issue **#653**, the “done” target is **full functional parity** with the current Python Burr for: core runtime semantics (actions, transitions, state semantics, sync/async execution, parallelism, hooks, typing-system behavior), persistence & tracking (including comparable/portable run data + on-disk formats), telemetry/observability parity, tracking server+UI parity (ideally UI unchanged by matching the server API), and CLI parity (or a documented mapping). ([GitHub][1])

You also explicitly want to exclude *Python integrations* for the initial implementation, while still laying the architectural foundation for future “Burr-like integrations” in Rust.

The public Burr behavior we need to match is documented primarily in the **Concepts** and **API reference** docs (state, applications, transitions, lifecycle hooks, tracking, persistence, parallelism, typing, etc.). ([Burr][2])

---

## 1) Multi-layer Rust skeleton folder structure

A clean way to support the “multi-layer golden suite → skeleton → slice-by-slice” workflow is a **Cargo workspace** in a subdirectory (so it doesn’t interfere with the existing Python packaging). This mirrors the logical subsystem boundaries already present in Burr’s docs and tests.

Here’s a suggested **repo-local** layout (Rust lives in the same repo as requested):

```text
apache-burr/
├─ burr/                         # existing Python package (reference impl)
├─ telemetry/
│  └─ ui/                        # existing React UI (must remain usable)
├─ tests/                        # existing Python tests (reference)
├─ rust/                         # NEW: Rust workspace root
│  ├─ Cargo.toml                 # workspace members + shared deps/versions
│  ├─ README.md                  # Rust Burr goals + parity matrix
│  ├─ crates/
│  │  ├─ burr-core/
│  │  │  ├─ Cargo.toml
│  │  │  └─ src/
│  │  │     ├─ lib.rs
│  │  │     ├─ state.rs          # immutable State + ops + field serde registry
│  │  │     ├─ serde.rs          # serialize/deserialize + dispatch
│  │  │     ├─ condition.rs      # when/expr/default + inversion + display
│  │  │     ├─ action.rs         # Action trait, function actions, reducers
│  │  │     ├─ graph.rs          # Graph + GraphBuilder + Transition model
│  │  │     ├─ types.rs          # shared types (IDs, pointers, timestamps)
│  │  │     └─ error.rs          # canonical error types
│  │  │
│  │  ├─ burr-runtime/
│  │  │  ├─ Cargo.toml
│  │  │  └─ src/
│  │  │     ├─ lib.rs
│  │  │     ├─ builder.rs        # ApplicationBuilder
│  │  │     ├─ application.rs    # Application (step/iterate/run + async)
│  │  │     ├─ execution.rs      # internal run loop + halt logic + seq IDs
│  │  │     ├─ streaming.rs      # streaming actions + containers + hooks
│  │  │     ├─ tracing.rs        # tracer factory + spans + context propagation
│  │  │     ├─ parallelism.rs    # RunnableGraph/SubGraphTask/TaskBasedParallelAction
│  │  │     └─ context.rs        # ApplicationContext equivalent
│  │  │
│  │  ├─ burr-lifecycle/
│  │  │  ├─ Cargo.toml
│  │  │  └─ src/
│  │  │     ├─ lib.rs
│  │  │     ├─ hooks.rs          # hook traits (sync + async)
│  │  │     └─ adapter_set.rs    # LifecycleAdapterSet ordering + fanout
│  │  │
│  │  ├─ burr-persistence/
│  │  │  ├─ Cargo.toml
│  │  │  └─ src/
│  │  │     ├─ lib.rs
│  │  │     ├─ traits.rs         # BaseStateLoader/Saver/Persister (+ async)
│  │  │     ├─ models.rs         # PersistedStateData + status model
│  │  │     └─ sqlite.rs         # SQLite persister parity (core supported impl)
│  │  │
│  │  ├─ burr-tracking/
│  │  │  ├─ Cargo.toml
│  │  │  └─ src/
│  │  │     ├─ lib.rs
│  │  │     ├─ client.rs         # LocalTrackingClient (filesystem)
│  │  │     ├─ models.rs         # Begin/End entries, spans, attributes, etc.
│  │  │     ├─ fs_layout.rs      # ~/.burr/<project>/<app_id>/ conventions
│  │  │     └─ normalize.rs      # normalization used by goldens
│  │  │
│  │  ├─ burr-server/
│  │  │  ├─ Cargo.toml
│  │  │  └─ src/
│  │  │     ├─ lib.rs
│  │  │     ├─ main.rs           # server binary entry
│  │  │     ├─ api/
│  │  │     │  ├─ mod.rs
│  │  │     │  ├─ v0.rs          # /api/v0 endpoints (UI contract)
│  │  │     │  └─ models.rs      # BackendSpec, Project, ApplicationPage, etc.
│  │  │     ├─ openapi.rs        # openapi generation (contract tests)
│  │  │     └─ static_assets.rs  # serve telemetry/ui build (optional)
│  │  │
│  │  ├─ burr-cli/
│  │  │  ├─ Cargo.toml
│  │  │  └─ src/
│  │  │     ├─ main.rs           # `burr` CLI
│  │  │     ├─ commands/
│  │  │     │  ├─ serve.rs
│  │  │     │  ├─ build_ui.rs
│  │  │     │  ├─ admin_server.rs
│  │  │     │  └─ telemetry.rs
│  │  │     └─ compat.rs         # mapping to existing python CLI behavior
│  │  │
│  │  ├─ burr-typing/
│  │  │  ├─ Cargo.toml
│  │  │  └─ src/
│  │  │     ├─ lib.rs
│  │  │     ├─ typing_system.rs  # TypingSystem trait parity
│  │  │     └─ serde_bridge.rs   # serde-based typed state views
│  │  │
│  │  └─ burr-plugins/
│  │     ├─ Cargo.toml
│  │     └─ src/
│  │        ├─ lib.rs
│  │        ├─ registry.rs       # future integration registry (feature-gated)
│  │        └─ traits.rs         # tracer/persister/tracker plugin traits
│  │
│  ├─ spec/                      # NEW: language-neutral spec + goldens
│  │  ├─ schema/                 # JSON schema(s) for scenarios + outputs
│  │  ├─ scenarios/              # executable scenarios (inputs)
│  │  ├─ goldens/                # normalized outputs from Python reference
│  │  └─ openapi/                # pinned OpenAPI JSON for UI contract
│  │
│  ├─ tests/                     # NEW: Rust test harness (insta/snapshots)
│  │  ├─ core_state.rs
│  │  ├─ core_runtime.rs
│  │  ├─ persistence.rs
│  │  ├─ tracking_client.rs
│  │  ├─ server_contract.rs
│  │  ├─ cli_contract.rs
│  │  └─ golden_runner.rs
│  │
│  └─ tools/
│     ├─ gen_goldens.py          # run Python Burr to generate spec/goldens
│     ├─ normalize.py            # normalizers (timestamps, UUIDs, paths)
│     └─ contract_extract.py     # OpenAPI extraction from python server
└─ ...
```

Why this structure matches Burr’s reality:

* Burr’s **public API surface** is explicitly divided into Applications, Graph, Actions, State, Serde, Persistence, Tracking, Lifecycle, Parallelism. ([Burr][3])
* The tracking system writes application + step start/end events to `~/.burr/<project>/<app_id>` by default, and is modeled as lifecycle hooks. ([Burr][4])
* The tracking server and UI are developed separately (server on port **7241**, UI on **3000**) and the server can run standalone (uvicorn in Python today). ([Burr][5])

---

## 2) Golden tests and API contracts to write before implementation

### 2.1 Golden-test suite philosophy (multi-layer)

Issue #653 explicitly proposes characterization / golden testing using the Python implementation as reference. ([GitHub][1])
To make this sustainable and *language-neutral*:

* Define **scenario inputs** as JSON/YAML specs (graph + deterministic actions + initial state + execution mode).
* Generate **golden outputs** by running the scenario through Python Burr and then normalizing nondeterminism (UUIDs, timestamps, temp paths).
* Run the same scenario through Rust Burr and compare to the stored golden output.

This yields “spec by example” and supports incremental slice implementation.

---

### 2.2 Layer 0: State semantics + serialization contracts

These come directly from Burr’s State API reference and Python tests:

**Contract: `State` is immutable, map-like, and supports the following operations**: `update`, `append`, `extend`, `increment`, `merge`, `subset`, `wipe`, `keys`, `get_all`, plus serialization/deserialization and field-level serde registration. ([Burr][2])

Golden tests (store golden as `state/<name>.json`):

1. `state_access__index_and_get`

    * `state["foo"]`, `state.get("foo")`, missing key errors, default behavior. ([GitHub][6])
2. `state_contains__in_operator`
3. `state_get_all__returns_copy`
4. `state_merge__rhs_overwrites_lhs`
5. `state_subset__keeps_only_keys_ignore_missing_true`
6. `state_update__upsert_semantics`
7. `state_append__list_upsert_and_append`
8. `state_extend__list_upsert_and_extend`
9. `state_append_extend__multi_key`
10. `state_wipe__delete_list`
11. `state_wipe__keep_list`
12. `state_append__validate_non_appendable_raises`
13. `state_extend__validate_non_extendable_raises`
14. `state_increment__ints_and_upsert`
15. `state_increment__validate_non_integer_raises`
16. `state_keys__returns_list_and_preserves_order`

    * Python explicitly tests that `keys()` returns a **list** and preserves insertion order. ([GitHub][6])
17. `state_field_level_serde__roundtrip`

    * `register_field_serde` modifies `State.serialize()` output and `State.deserialize()` restores original value. ([Burr][2])
18. `state_field_level_serde__bad_serializer_shape_raises`
19. `state_field_level_serde__register_requires_kwargs`
20. `state_typing_system__preserved_across_ops`

    * State operations should preserve attached typing system pointer. ([GitHub][6])

**API contracts (Rust) to freeze now:**

* `State` must preserve key order (use `IndexMap` or equivalent).
* `serialize()` / `deserialize()` must be stable and deterministic for JSON.
* Field serde registry must be global per field name (mirrors docs). ([Burr][2])

---

### 2.3 Layer 1: Conditions / transitions contracts

From the Transitions concept doc:

* Conditions can be `when(k=v)`, `expr("...")`, or `default`.
* Conditions are evaluated in order; the first `True` transition wins.
* If no condition matches, execution stops early (warn/undefined behavior).
* `~condition` inverts a condition.
* All keys referenced in a condition must exist in state or it errors. ([Burr][7])

Golden tests (`transitions/<name>.json`):

21. `condition_when__true_false`
22. `condition_when__missing_key_errors`
23. `condition_expr__basic_comparisons`
24. `condition_expr__missing_key_errors`
25. `condition_default__always_true`
26. `condition_inversion__tilde_operator`
27. `transition_ordering__first_match_wins`
28. `transition_tuple_len2__defaults_to_default_condition` ([Burr][7])
29. `no_transition_match__halts_early_contract`

    * Capture: last action/result/state; plus “halted due to no valid transition” marker in normalized output.

**API contracts (Rust) to freeze now:**

* Deterministic condition stringification for UI graph (“condition: string”). ([Apache Git Repositories][8])

---

### 2.4 Layer 2: GraphBuilder + Graph contracts

From Applications/Graph docs: GraphBuilder supports adding actions + transitions, then `.build()` returns Graph. ([Burr][9])

Golden tests (`graph/<name>.json`):

30. `graphbuilder_add_actions__by_value_and_by_name`
31. `graphbuilder_add_graph__name_conflict_errors`
32. `graphbuilder_transitions__from_list_to_single`
33. `graphbuilder_build__graph_is_static`
34. `graph_get_action__returns_action_or_none` ([Burr][10])
35. `graph_transition_serialization__stable`

**API contracts (Rust):**

* Graph representation must be serializable to the UI’s `ApplicationModel` + `TransitionModel` shapes (entrypoint/actions/transitions). ([Apache Git Repositories][8])

---

### 2.5 Layer 3: Action model contracts (sync, async, reducer, binding)

From API and Python tests, Burr has:

* `Action` abstraction with `reads`, `writes`, `inputs`, `run(...) -> dict`, `update(result,state)->State`.
* Sync execution must reject async actions in sync mode and vice versa.
* Reducers validate write-set semantics.
* There is internal dunder-parameter remapping and tracer injection (`__tracer`). ([GitHub][11])

Golden tests (`actions/<name>.json`):

36. `action_run__sync_function_returns_dict`
37. `action_run__sync_function_with_inputs`
38. `action_run__sync_rejects_async_action` ([GitHub][11])
39. `action_run__rejects_non_dict_result` ([GitHub][11])
40. `action_update__applies_state_delta`
41. `reducer__modifies_state`
42. `reducer__deletes_state_keys`
43. `reducer_writes_validation__errors_on_invalid_contract`
44. `dunder_param_remap__tracer_and_context_reserved_names`

**API contracts (Rust):**

* An action must be able to declare `reads/writes/inputs` metadata (needed for typing system + UI).
* Provide both (a) trait-object actions and (b) ergonomic function-based actions.

---

### 2.6 Layer 4: Application execution contracts (sync + async)

From Applications concept doc, Burr exposes:

* `step` / `astep`
* `iterate` / `aiterate`
* `run` / `arun`
* `stream_result` / `astream_result` ([Burr][9])

And from API docs:

* `with_entrypoint`, `with_state`, `with_graph`, `with_actions`, `with_transitions`, `with_hooks`, `with_identifiers`, etc. ([Burr][10])

Golden tests (`runtime/<name>.json`):

45. `app_builder__minimal_build`
46. `app_step__executes_entrypoint_then_transition`
47. `app_step__sequence_id_increments`
48. `app_state_internal_keys__prior_step_and_sequence_id_present`

* Burr examples show `__PRIOR_STEP` / `__SEQUENCE_ID` appear in state snapshots. ([Burr][12])

49. `app_iterate__halts_after_action`
50. `app_iterate__halts_before_action`
51. `app_run__returns_final_tuple`
52. `app_inputs__only_first_action_supported_in_iterate_run_contract` ([Burr][9])
53. `app_halt_conditions__first_encountered_wins`
54. `app_no_transition_match__halts_with_warning_marker`
55. `app_async_step__astep_parity`
56. `app_async_iterate__aiterate_parity`
57. `app_async_run__arun_parity`
58. `app_error__action_exception_propagation_and_state`
59. `app_error__post_step_hooks_receive_exception`
60. `app_end_to_end__collatz_characterization`

* Burr’s own e2e tests include collatz; mirror as a golden scenario in Rust. ([GitHub][13])

**API contracts (Rust):**

* Provide sync + async execution APIs (Tokio feature-gated, if needed).
* Preserve Burr halting semantics and return payload structure.

---

### 2.7 Layer 5: Lifecycle hooks ordering + parameter contracts

The Lifecycle API specifies hooks and their required parameters for pre/post step, application execute calls, spans, streams, etc. ([Burr][14])

Golden tests (`hooks/<name>.json`):

61. `hooks_pre_post_run_step__order_and_payload`
62. `hooks_pre_post_execute_call__order_and_method_enum`
63. `hooks_post_application_create__called_on_build`
64. `hooks_async_variants__called_in_async_mode`
65. `hooks_future_kwargs__ignored_but_accepted_contract`
66. `hooks_multiple_adapters__stable_ordering`
67. `hooks_error_path__post_step_called_with_exception` ([Burr][14])

**API contracts (Rust):**

* A `LifecycleAdapterSet` equivalent that preserves hook order and fans out safely.
* Sync and async hook traits (mirrors doc classes). ([Burr][14])

---

### 2.8 Layer 6: Streaming actions + stream hooks contracts

Burr supports streaming actions (`stream_result` / `astream_result`) and stream hook events (pre start, item, end). ([Burr][9])

Golden tests (`streaming/<name>.json`):

68. `streaming_action__yields_items_and_final_state`
69. `stream_hooks__pre_start_item_end_called_with_timestamps`
70. `async_streaming_action__parity`
71. `streaming_error__end_hook_receives_exception_marker`
72. `stream_container__collects_items_contract`

---

### 2.9 Layer 7: Tracing inside actions (spans) contracts

Burr injects a tracer factory (via `__tracer`) that creates spans, which then trigger lifecycle span hooks (`pre_span_start`, `post_span_end`, etc.). ([Burr][15])

Golden tests (`tracing/<name>.json`):

73. `tracer_factory__nested_spans_generate_begin_end_events`
74. `span_parenting__parent_span_id_and_dependencies`
75. `span_hooks__called_in_correct_order`
76. `span_async__parity`

---

### 2.10 Layer 8: Parallelism + recursive applications contracts

Parallelism API provides `RunnableGraph`, `SubGraphTask`, and `TaskBasedParallelAction` (and helpers like mapping actions across states). ([Burr][16])

Golden tests (`parallelism/<name>.json`):

77. `runnable_graph__wraps_single_action`
78. `subgraph_task__runs_sub_application_and_returns_state` ([Burr][16])
79. `task_based_parallel_action__runs_tasks_in_parallel_and_reduces`
80. `map_states__cartesian_mapping_contract`
81. `parallel_executor__threadpool_default_sync`
82. `parallel_async__tokio_gather_default`
83. `recursive_app_tracking__child_pointer_logged`

---

### 2.11 Layer 9: Persistence contracts (state saver/loader + initialize_from)

From API reference, persisters save `(partition_key, app_id, sequence_id, position, state, status)` and can `load()` latest or specific sequence. ([Burr][17])

Golden tests (`persistence/<name>.json`):

84. `persister_save__completed_state_written_each_step`
85. `persister_save__failed_state_writes_pre_action_state_and_failed_status` ([Burr][17])
86. `persister_load__latest_completed`
87. `persister_load__specific_sequence_id`
88. `persister_list_app_ids__by_partition_key`
89. `initialize_from__resume_at_next_action_true`
90. `initialize_from__resume_at_next_action_false_uses_default_entrypoint`
91. `forking__fork_from_app_id_and_sequence_id_sets_parent_pointer`
92. `identifiers__with_identifiers_affects_lookup_and_tracking` ([Burr][10])

**API contracts (Rust):**

* Traits analogous to `BaseStateLoader/BaseStateSaver/(Async…)`.
* At least one “core” persister implemented (SQLite is already a “supported sync implementation” in Python; Rust should match). ([GitHub][11])

---

### 2.12 Layer 10: Tracking client contracts (LocalTrackingClient filesystem format)

Burr tracking concept + reference specify:

* Project → application (trace-like) → steps data model.
* `LocalTrackingClient` writes to `~/.burr` by default and creates `~/.burr/<project>/<app_id>` application directories; logs graph + step begin/end lines. ([Burr][4])

We also have direct evidence the tracking client:

* Filters inputs with a filterlist containing `__tracer` (so tracer injection isn’t logged as input). ([GitHub][18])
* Uses models like `BeginEntryModel`, `EndEntryModel`, `BeginSpanModel`, `EndSpanModel`, stream models, attribute models, pointers, etc. ([GitHub][18])

Golden tests (`tracking_client/<name>.json` + golden directories):

93. `tracking_fs_layout__creates_project_and_app_dirs`
94. `tracking_writes_application_graph__on_post_application_create`
95. `tracking_logs_step_begin_end__per_step_jsonl_or_equivalent`
96. `tracking_logs_span_begin_end__per_span`
97. `tracking_logs_stream_events__initialize_first_item_end`
98. `tracking_inputs_filtered__tracer_removed`
99. `tracking_format_exception__stable_string`
100. `tracking_load__reconstructs_state_entrypoint_from_logs`
101. `tracking_app_log_exists__true_false`
102. `tracking_list_app_ids__returns_sorted_or_stable_contract`
103. `tracking_copy__forking_tracker_statefulness` ([Burr][19])

**On-disk format contract to pin early:**

* Explicitly document and test:

    * directory layout
    * filenames
    * JSON schema for each record type
    * ordering guarantees

This is essential because Issue #653 requires portability/comparability of recorded runs. ([GitHub][1])

---

### 2.13 Layer 11: Tracking server + UI API contract (OpenAPI-first)

The dev docs show the tracking server exists and the UI assumes it at port **7241**; the UI is a separate React app. ([Burr][5])
The UI client is generated with `openapi-typescript-codegen` (so the server’s OpenAPI spec is the “source of truth”). ([GitHub][20])

**Server API contract tests should be built around OpenAPI snapshots:**

* Extract Python server `/openapi.json` and commit to `rust/spec/openapi/python-openapi-v0.json`.
* Rust server tests must ensure:

    * `GET /openapi.json` matches **exactly** (modulo ordering)
    * endpoint behavior matches on representative fixtures (projects/apps/log retrieval)

Key endpoints we can already confirm from the generated client + UI usage:

* Health:

    * `GET /ready` ([GitHub][20])
* Metadata:

    * `GET /api/v0/metadata/app_spec` → `BackendSpec` ([GitHub][20])
* Apps listing (UI uses this):

    * `DefaultService.getAppsApiV0ProjectIdPartitionKeyAppsGet(projectId, "__none__")` ([Apache Git Repositories][21])
* Demo endpoints (used by the UI “demos” section):

    * `/api/v0/streaming_chatbot/response/{project_id}/{app_id}` is POSTed to directly via fetch, and streams `data: {json}` events. ([Apache Git Repositories][22])
    * Additional demo routers exist (chatbot/email assistant/deep researcher), but those are “integration-ish”; you can stub them initially while keeping the core tracking UI usable. ([Apache Git Repositories][8])

Golden tests (`server_contract/<name>.json`):

104. `openapi_snapshot__matches_python_reference`
105. `ready_endpoint__200_ok`
106. `metadata_app_spec__shape_and_fields`
107. `projects_list__deterministic_order_and_shape`
108. `apps_list__pagination_and_sorting_matches_ui_expectations`
109. `application_logs__steps_spans_attributes_roundtrip`
110. `annotations_crud__create_update_list`
111. `static_assets_serving__optional_when_enabled`
112. `cors_and_proxy__ui_dev_mode_compat` ([Burr][5])

**API contracts (Rust server):**

* Paths, query params, models, and status codes must match the OpenAPI snapshot.
* Response JSON must match field names exactly (e.g., UI models show `from_` and `to` in `TransitionModel`). ([Apache Git Repositories][8])

---

### 2.14 Layer 12: CLI parity contracts

Dev docs show CLI commands used today:

* Start server (python wrapper) example:

    * `burr-admin-server --no-open --dev-mode` (plus env var `BURR_SERVE_STATIC=false`) ([Burr][5])
* Build UI:

    * `burr-admin-build-ui` ([Burr][5])

And the Python `burr` CLI uses `click`, invokes `npm install` + `npm run build` for UI, and opens a browser when server is ready. ([GitHub][23])

Golden tests (`cli_contract/<name>.txt` + `.json`):

113. `cli_help__top_level_and_subcommands`
114. `cli_serve__flags_and_defaults_match`
115. `cli_build_ui__invokes_npm_commands_or_errors_helpfully`
116. `cli_no_open__does_not_open_browser`
117. `cli_dev_mode__disables_static_serving_contract`
118. `cli_exit_codes__nonzero_on_failure`
119. `cli_telemetry__emits_event_when_enabled`

---

### 2.15 Cross-cutting: Usage analytics / data privacy contract

Burr collects anonymous usage data by default (application build, execution functions, CLI commands), with multiple opt-out mechanisms (function call, config file key, env var). ([Burr][24])

Golden tests (`telemetry/<name>.json`):

120. `telemetry_enabled__default_true_and_creates_uuid_config`
121. `telemetry_disable_programmatically__no_events`
122. `telemetry_disable_env_var__no_events`
123. `telemetry_disable_config__no_events` ([Burr][24])

---

## 3) Detailed technical implementation plan (to pass all tests above)

This plan aligns to your Codex workflow: define specs → write tests/shell → implement slice-by-slice until all pass. ([GitHub][1])

### Phase A — Repo + spec scaffolding (no Burr logic yet)

**A1. Create Rust workspace + crates (skeleton only)**

* Add `rust/` workspace and empty crates (as in folder structure).
* Add CI hook (optional initially) to run `cargo test -p burr-*`.

**A2. Add language-neutral scenario spec + golden harness**

* Create `rust/spec/schema/scenario.schema.json`.
* Create `rust/tools/gen_goldens.py` that:

    * imports Python Burr,
    * runs a scenario,
    * captures:

        * executed action names
        * per-step inputs/results/state
        * hook events
        * persistence records
        * tracking records
    * normalizes nondeterminism:

        * UUIDs (app_id)
        * timestamps
        * OS-specific paths
* Create Rust test harness that:

    * loads scenario,
    * runs Rust Burr,
    * produces same normalized output shape,
    * compares to `spec/goldens/...`.

**Acceptance criteria:** harness runs and fails meaningfully with unimplemented behavior.

---

### Phase B — Core data model and deterministic semantics

**B1. Implement `State`**

* Use `indexmap::IndexMap<String, serde_json::Value>` internally for:

    * insertion order stability (required by tests). ([GitHub][6])
* Implement:

    * `get`, `index`, `contains`, `get_all` (clone)
    * `update`, `merge`, `subset(ignore_missing=true default)`
    * `append`, `extend` with validation (list-like)
    * `increment` with validation (int-like)
    * `wipe(delete|keep)`
    * `keys() -> Vec<String>`
* Implement field-level serde registry:

    * global `HashMap<String, (serializer, deserializer)>`
    * enforce “must accept kwargs” equivalent: in Rust, mimic by requiring a context param `SerdeContext` even if unused; if not provided, compile-time prevents it, so write tests around runtime registration input validation to preserve behavior.

**Pass tests:** 1–20.

---

**B2. Implement `serialize/deserialize` and stable JSON**

* Represent `State.serialize()` as plain JSON object.
* If field serde exists for a key, replace its value with serializer output (must be JSON object); else error like Python. ([GitHub][6])
* Implement `State::deserialize` applying deserializer for registered fields.

---

**B3. Implement conditions + transitions**

* Define `Condition` enum:

    * `When { kv: Vec<(String, Value)> }`
    * `Expr { expr: String }` (use a small safe expression evaluator; see note below)
    * `Default`
    * `Not(Box<Condition>)`
* Implement evaluation semantics per docs: ordering, inversion, missing keys error. ([Burr][7])
* Implement deterministic stringification for UI/graph.

**Pass tests:** 21–29.

> Expr evaluator note: to match Python behavior you’ll need a constrained expression language over state variables (comparisons, boolean ops, parentheses). Implement a small parser (e.g., `pest`/`nom`) and interpret against `serde_json::Value` with type rules matching Python.

---

### Phase C — Actions + Graph + execution engine (sync first)

**C1. GraphBuilder + Graph**

* Implement Graph structure:

    * actions map `{name -> Action}`
    * transitions list preserving insertion order
* Validate uniqueness on add.
* `build()` freezes graph (clone / Arc).

**Pass tests:** 30–35.

---

**C2. Action trait + function actions**

* Trait `Action`:

    * `fn name(&self) -> &str`
    * `fn reads(&self) -> &[String]`
    * `fn writes(&self) -> &[String]`
    * `fn inputs(&self) -> &[String]`
    * `fn run(&self, state: &State, inputs: &Inputs, ctx: &mut ActionCtx) -> Result<Map<String, Value>>`
    * `fn update(&self, result: &Map<String, Value>, state: &State) -> Result<State>`
* Provide `FnAction` wrapper for closures.
* Provide reducer support with write-set validation.

**Pass tests:** 36–44.

---

**C3. ApplicationBuilder + sync execution**

* `ApplicationBuilder` methods:

    * `with_actions`, `with_transitions`, `with_graph(s)`, `with_state`, `with_entrypoint`
    * `with_identifiers(app_id, partition_key, sequence_id)`
    * `with_hooks(...)`
    * `with_tracker(...)` and `with_state_persister(...)` later
* `Application` sync:

    * store:

        * current state (immutable)
        * current action pointer (position)
        * `sequence_id`
        * identifiers
        * `LifecycleAdapterSet`
* Implement `step(inputs?)`:

    * choose action based on current position (initial entrypoint if first step)
    * call pre hooks
    * call action.run
    * apply update
    * set internal state keys:

        * `__SEQUENCE_ID`
        * `__PRIOR_STEP`
    * compute next action via transitions
    * call post hooks
* Implement `iterate(halt_after, halt_before, inputs?)` generator semantics.
* Implement `run(...)` wrapper.

These semantics are described in docs and tested heavily in Python. ([Burr][9])

**Pass tests:** 45–54 and basic e2e 60.

---

### Phase D — Lifecycle hooks, telemetry events, async/streaming

**D1. LifecycleAdapterSet + hook ordering**

* Implement hook traits mirroring the Lifecycle API reference:

    * PreRunStepHook, PostRunStepHook, PostApplicationCreateHook, Pre/Post execute call hooks, etc. ([Burr][14])
* Ensure every public “execute call” (`step/run/iterate/stream…`) wraps with pre/post execute call hooks and passes a method enum like Python’s `ExecuteMethod`. ([Burr][14])
* Provide `future_kwargs` equivalent: Rust can accept an `Extensions` map that is passed through to support forward compatibility.

**Pass tests:** 61–67.

---

**D2. Async runtime parity**

* Introduce `async` variants:

    * `astep`, `aiterate`, `arun`
* Keep sync and async code paths sharing a common “state machine core” where possible to prevent divergence.
* Add async hook traits (mirroring reference). ([Burr][14])

**Pass tests:** 55–57 + 64.

---

**D3. Streaming actions parity**

* Add `StreamingAction` abstraction:

    * sync returns iterator of items + a final reducer step to update state
    * async returns stream
* Implement `stream_result` / `astream_result` plus stream hooks:

    * pre_start_stream
    * post_stream_item
    * post_end_stream ([Burr][14])

**Pass tests:** 68–72.

---

**D4. Tracing inside actions parity**

* Implement:

    * `TracerFactory` injected into action context (Rust: pass via `ActionCtx`)
    * `ActionSpanTracer` RAII guard which triggers:

        * pre_span_start hooks (and async)
        * post_end_span hooks (and async) ([Burr][15])
* Ensure parent/child span IDs and dependencies match the UI models. ([Apache Git Repositories][8])

**Pass tests:** 73–76.

---

### Phase E — Parallelism + recursion

**E1. Implement parallel primitives**

* `RunnableGraph` creation from Action/callable/graph. ([Burr][16])
* `SubGraphTask` runs sub-application:

    * propagate parent context
    * optionally share tracker/persister pointers
* `TaskBasedParallelAction`:

    * yields tasks, executes them concurrently, reduces states.

**E2. Execution backends**

* Sync: default threadpool executor (like Python docs mention threadpool executor). ([Burr][10])
* Async: use `tokio::task::JoinSet` / `futures::future::join_all`.

**Pass tests:** 77–83.

---

### Phase F — Persistence + `initialize_from` + forking

**F1. Persistence traits**

* Define trait set mirroring Python:

    * `BaseStateLoader`, `BaseStateSaver`, `BaseStatePersister` and async variants. ([Burr][17])
* Model `PersistedStateData` and status semantics (“completed” vs “failed”). ([Burr][17])

**F2. SQLite persister**

* Implement SQLite schema compatible with Python `SQLLitePersister` behavior (read/write by partition_key/app_id/sequence_id). ([GitHub][11])
* Ensure save semantics:

    * if status failed, state should reflect pre-action state (per docs). ([Burr][17])

**F3. `initialize_from`**

* Implement builder method to load state/entrypoint/sequence_id and resume depending on `resume_at_next_action`. ([Burr][12])
* Forking:

    * set parent pointer, assign new app_id if forked.

**Pass tests:** 84–92.

---

### Phase G — Tracking client parity (filesystem)

**G1. Implement Rust `LocalTrackingClient`**

* Mirror Python behavior:

    * default `storage_dir = ~/.burr`
    * create dirs `~/.burr/<project>/<app_id>` ([Burr][19])
    * on application create: write static graph representation
    * on pre/post step: write begin/end entries
* Input filtering: exclude `__tracer` from logged inputs. ([GitHub][18])
* Use file locks where appropriate (Python uses `fcntl` when available). ([GitHub][18])

**G2. Implement tracking models**

* Define Rust structs matching UI expectations:

    * ApplicationModel, TransitionModel, BeginEntryModel, EndEntryModel, BeginSpanModel, EndSpanModel, stream models, etc. ([Apache Git Repositories][8])

**G3. Load/list helpers**

* `app_log_exists`, `list_app_ids`, `load(...)` semantics (load latest completed if sequence_id not given). ([Burr][19])

**Pass tests:** 93–103.

---

### Phase H — Tracking server in Rust (UI-compatible)

**H1. Contract-first: pin OpenAPI**

* Add tool to run python server and extract `/openapi.json` into `rust/spec/openapi/python-openapi-v0.json`.
* Add Rust test that compares Rust server `/openapi.json` to pinned snapshot.

**H2. Implement endpoints**

* Start with minimal endpoints required for UI core:

    * `/ready`
    * `/api/v0/metadata/app_spec`
    * projects/apps/log retrieval endpoints used by UI
* Add annotations endpoints next.

The UI uses a generated client and expects stable contract. ([GitHub][20])

**H3. Static serving + dev mode**

* Support:

    * `BURR_SERVE_STATIC=false` dev mode (proxy UI separately). ([Burr][5])
    * optional static asset serving for production.

**Pass tests:** 104–112.

---

### Phase I — CLI parity (Rust)

**I1. Implement `burr` CLI + admin commands**

* Mirror existing behavior:

    * run server on port 7241
    * optionally open browser when `/ready` is 200 (Python polls then opens). ([GitHub][23])
    * build UI via npm commands (or provide guidance if npm missing). ([GitHub][23])
* Provide a compatibility layer:

    * keep old command names if possible (`burr-admin-server`, `burr-admin-build-ui`) as additional binaries or subcommands.

**I2. CLI contract tests**

* Snapshot `--help` text, flags, exit codes.

**Pass tests:** 113–119.

---

### Phase J — Telemetry opt-out parity

Implement minimal telemetry event emission with the same opt-out mechanisms. ([Burr][24])

**Pass tests:** 120–123.

---

## 4) Laying the foundation for integrations (Rust-native roadmap + gaps)

Even though “integrations are out-of-scope for initial parity,” you can (and should) design the Rust code so integrations are **just additional implementations** of stable traits:

### 4.1 Integration foundation to bake in now

* **Plugin/registry layer** (`burr-plugins`):

    * `Tracker` trait (tracking client)
    * `Persister` trait (persistence backends)
    * `LifecycleAdapter` trait objects
    * `TracingBridge` trait (OpenTelemetry, etc.)
* **Feature flags** per integration:

    * `otel`, `redis`, `mongodb`, `s3`, etc.
* **No core crate depends on heavy integrations**:

    * `burr-core` stays dependency-light (mirrors Burr’s “dependency-free core library” messaging). ([Apache Git Repositories][25])

### 4.2 Rust-native replacements mapping (follow-up roadmap)

Below are realistic “Rust equivalents” to the Python integrations listed in Burr docs, with gaps noted:

* **OpenTelemetry integration**

    * Python has an OpenTelemetry bridge via lifecycle hooks. ([Burr][26])
    * Rust: `tracing` + `opentelemetry` + `tracing-opentelemetry` (good parity).
    * Gap: “instrument lots of AI/vendor SDKs” is easier in Python; Rust instrumentation exists but is less standardized.

* **Storage/persistence integrations**

    * Python supports multiple persisters (SQLite, Redis, MongoDB, etc.). ([Burr][17])
    * Rust: `sqlx` (SQLite/Postgres), `redis` crate, `mongodb` crate, `aws-sdk-s3` for S3.
    * Gap: matching *exact* Python on-disk formats and migration behavior may require explicit schema versioning tests.

* **Streamlit integration**

    * Rust does not have a direct Streamlit equivalent.
    * Roadmap: keep the existing React UI as the canonical UI, optionally add:

        * a small Rust web UI (Leptos/Yew) later, or
        * embed the existing UI assets and serve them from Rust server (already in-scope).

* **LangChain / LlamaIndex / Haystack style integrations**

    * Python’s ecosystem is much richer.
    * Rust options exist but are fragmented:

        * `langchain-rust`, `rig`, `llm-chain`, plus direct vendor SDK crates.
    * Gap: fewer “batteries included” adapters; likely need Burr-native patterns rather than wrapper integrations.

* **Ray (distributed execution)**

    * Python has Ray integration docs. ([Burr][24])
    * Rust: no Ray equivalent; future options are:

        * NATS/JetStream + workers
        * Tokio + distributed task queues
        * Temporal (Rust SDK exists) as a “distributed workflow backend” (though Burr’s model differs)

The key is: if you design your core around stable traits + lifecycle events (as Burr already does), integrations become “just adapters.”

---

## Closing note

The most important early decisions (that you should lock in the golden suite) are:

1. **State + serialization determinism** (key order, field serde, JSON stability). ([Burr][2])
2. **Execution semantics** (step/iterate/run, halting, sequence IDs, internal keys). ([Burr][9])
3. **Lifecycle hook ordering + payload** (because tracking, tracing, persistence are built on it). ([Burr][14])
4. **Tracking + server API contracts** (because the UI depends on exact shapes/paths). ([Apache Git Repositories][21])

[1]: https://github.com/apache/burr/issues/653 "https://github.com/apache/burr/issues/653"
[2]: https://burr.apache.org/reference/state/ "https://burr.apache.org/reference/state/"
[3]: https://burr.apache.org/reference/ "https://burr.apache.org/reference/"
[4]: https://burr.apache.org/concepts/tracking/ "https://burr.apache.org/concepts/tracking/"
[5]: https://burr.apache.org/contributing/iterating/ "https://burr.apache.org/contributing/iterating/"
[6]: https://raw.githubusercontent.com/apache/burr/main/tests/core/test_state.py "https://raw.githubusercontent.com/apache/burr/main/tests/core/test_state.py"
[7]: https://burr.apache.org/concepts/transitions/ "https://burr.apache.org/concepts/transitions/"
[8]: https://apache.googlesource.com/burr/%2B/448f38046ed420895b874e2c1e094b92ff703fcc%5E%21/ "https://apache.googlesource.com/burr/%2B/448f38046ed420895b874e2c1e094b92ff703fcc%5E%21/"
[9]: https://burr.apache.org/concepts/state-machine/ "https://burr.apache.org/concepts/state-machine/"
[10]: https://burr.apache.org/reference/application/ "https://burr.apache.org/reference/application/"
[11]: https://raw.githubusercontent.com/apache/burr/main/tests/core/test_application.py "https://raw.githubusercontent.com/apache/burr/main/tests/core/test_application.py"
[12]: https://burr.apache.org/examples/chatbots/gpt-like-chatbot/ "https://burr.apache.org/examples/chatbots/gpt-like-chatbot/"
[13]: https://raw.githubusercontent.com/apache/burr/main/tests/test_end_to_end.py "https://raw.githubusercontent.com/apache/burr/main/tests/test_end_to_end.py"
[14]: https://burr.apache.org/reference/lifecycle/ "https://burr.apache.org/reference/lifecycle/"
[15]: https://burr.apache.org/reference/visibility/ "https://burr.apache.org/reference/visibility/"
[16]: https://burr.apache.org/reference/parallelism/ "https://burr.apache.org/reference/parallelism/"
[17]: https://burr.apache.org/reference/persister/ "https://burr.apache.org/reference/persister/"
[18]: https://raw.githubusercontent.com/apache/burr/main/burr/tracking/client.py "https://raw.githubusercontent.com/apache/burr/main/burr/tracking/client.py"
[19]: https://burr.apache.org/reference/tracking/ "https://burr.apache.org/reference/tracking/"
[20]: https://raw.githubusercontent.com/apache/burr/main/telemetry/ui/src/api/services/DefaultService.ts "https://raw.githubusercontent.com/apache/burr/main/telemetry/ui/src/api/services/DefaultService.ts"
[21]: https://apache.googlesource.com/burr/%2B/refs/heads/add_model_costs/telemetry/ui/src/examples/Common.tsx "https://apache.googlesource.com/burr/%2B/refs/heads/add_model_costs/telemetry/ui/src/examples/Common.tsx"
[22]: https://apache.googlesource.com/burr/%2B/refs/heads/add_model_costs/telemetry/ui/src/examples/StreamingChatbot.tsx "https://apache.googlesource.com/burr/%2B/refs/heads/add_model_costs/telemetry/ui/src/examples/StreamingChatbot.tsx"
[23]: https://raw.githubusercontent.com/apache/burr/main/burr/cli/__main__.py "https://raw.githubusercontent.com/apache/burr/main/burr/cli/__main__.py"
[24]: https://burr.apache.org/reference/telemetry/ "https://burr.apache.org/reference/telemetry/"
[25]: https://apache.googlesource.com/burr/%2B/2eca07db7ca1d91f7763c348da2be1beec1cc14e "https://apache.googlesource.com/burr/%2B/2eca07db7ca1d91f7763c348da2be1beec1cc14e"
[26]: https://burr.apache.org/reference/integrations/opentelemetry/ "https://burr.apache.org/reference/integrations/opentelemetry/"
