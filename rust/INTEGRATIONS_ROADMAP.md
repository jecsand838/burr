# Integrations roadmap (post-parity)

Initial Rust parity explicitly excludes Python-first integrations, but we want the architecture to support **Burr-like integrations** in Rust.

This file tracks:
- integrations present in Python Burr
- candidate Rust-native equivalents
- gaps to close

## Out-of-scope for initial parity (examples)

- LLM provider integrations (OpenAI, Anthropic, etc.)
- Framework adapters (LangChain/LangGraph equivalents)
- Observability exporters beyond the minimal tracing foundation
- Hosted persistence backends beyond SQLite
- S3 tracking backend (beyond the local filesystem tracking parity)

## Candidate Rust-native building blocks (examples)

- HTTP clients: `reqwest`
- Async runtime: `tokio`
- Serialization: `serde`, `serde_json`
- SQLite: `rusqlite` (or `sqlx` if async is desired)
- OpenTelemetry: `opentelemetry` ecosystem (as needed for parity)

## Gaps / follow-ups

- Evaluate ergonomic typed-state story (derive macros vs runtime typing)
- Decide on plugin boundary (feature-gated crates vs dynamic loading)
- Define a stable “integration registry” API (see `burr-plugins` in the plan)
