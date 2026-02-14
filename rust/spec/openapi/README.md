# Tracking server OpenAPI snapshots

This directory stores the pinned OpenAPI contract for the tracking server that powers the existing Burr UI.

Planned workflow:
1) Extract the OpenAPI JSON from the Python server and write it to `python_openapi.json`.
2) Add contract tests so the Rust server emits an equivalent OpenAPI document.

See `rust/RUST_IMPLEMENTATION_PLAN.md` and the Codex prompt sequence.
