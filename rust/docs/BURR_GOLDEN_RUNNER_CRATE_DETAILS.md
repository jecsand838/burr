Below is a companion **`burr-golden-runner`** crate skeleton that:

* loads scenario JSON files,
* invokes a **Python reference engine** (subprocess) and/or a **Rust engine** (stubbed until your Rust Burr runtime exists),
* writes **raw + normalized** `run_report.json` artifacts into a run directory,
* normalizes via your **`burr-normalizer`** crate (which itself is wired to `serde_json_path` and RFC 8785 canonical output),
* compares normalized output to a committed golden file,
* prints a **unified diff** on mismatch (via `similar::TextDiff`, which supports unified diff formatting). ([Docs.rs][1])

It also includes a scenario-local normalizer override loader using the convention you requested.

---

## 1) File tree

Create this at: `rust/crates/burr-golden-runner/`

```text
burr-golden-runner/
├─ Cargo.toml
├─ src/
│  ├─ main.rs
│  ├─ lib.rs
│  ├─ cli.rs
│  ├─ error.rs
│  ├─ paths.rs
│  ├─ scenario.rs
│  ├─ engines/
│  │  ├─ mod.rs
│  │  ├─ python.rs
│  │  └─ rust.rs
│  ├─ normalize.rs
│  ├─ goldens.rs
│  ├─ diff.rs
│  └─ runner.rs
└─ tests/
   └─ cli_smoke.rs
```

Add to your `rust/Cargo.toml` workspace:

```toml
[workspace]
members = [
  "crates/burr-normalizer",
  "crates/burr-golden-runner",
]
resolver = "2"
```

---

## 2) `Cargo.toml`

```toml
[package]
name = "burr-golden-runner"
version = "0.1.0"
edition = "2021"
license = "Apache-2.0"
publish = false
description = "Golden-test runner for Apache Burr Rust parity suite."

[[bin]]
name = "burr-golden"
path = "src/main.rs"

[dependencies]
# CLI
clap = { version = "4", features = ["derive"] } # derive/subcommands patterns supported by clap

# Data / config
serde = { version = "1", features = ["derive"] }
serde_json = "1"
serde_yaml = "0.9"

# Errors
thiserror = "1"

# File system / discovery
tempfile = "3"   # TempDir auto-cleans up on drop
walkdir = "2"

# Hashing
sha2 = "0.10"
hex = "0.4"

# Diff output
similar = { version = "2", features = ["text"] } # TextDiff + unified diff formatting

# Your normalizer crate (hyphen becomes underscore in Rust imports)
burr-normalizer = { path = "../burr-normalizer" }

[dev-dependencies]
assert_cmd = "2" # CLI integration testing helper
predicates = "3"
```

---

## 3) `src/main.rs`

```rust
use burr_golden_runner::cli;

fn main() {
    // Keep main tiny; CLI returns exit codes.
    let code = cli::main_entry();
    std::process::exit(code);
}
```

---

## 4) `src/lib.rs`

```rust
pub mod cli;
pub mod diff;
pub mod engines;
pub mod error;
pub mod goldens;
pub mod normalize;
pub mod paths;
pub mod runner;
pub mod scenario;

pub use crate::error::{Result, RunnerError};
pub use crate::runner::{GoldenRunner, RunRequest, RunSummary};
```

---

## 5) `src/error.rs`

```rust
use thiserror::Error;

pub type Result<T> = std::result::Result<T, RunnerError>;

#[derive(Debug, Error)]
pub enum RunnerError {
    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),

    #[error("JSON error: {0}")]
    Json(#[from] serde_json::Error),

    #[error("YAML error: {0}")]
    Yaml(#[from] serde_yaml::Error),

    #[error("normalizer error: {0}")]
    Normalizer(#[from] burr_normalizer::error::NormalizeError),

    #[error("repo discovery error: {0}")]
    RepoDiscovery(String),

    #[error("scenario error: {0}")]
    Scenario(String),

    #[error("engine '{engine}' failed: {details}")]
    EngineFailure { engine: String, details: String },

    #[error("golden file missing for scenario '{scenario_id}': {path}")]
    GoldenMissing { scenario_id: String, path: String },

    #[error("golden mismatch for scenario '{scenario_id}': {path}\n{diff}")]
    GoldenMismatch {
        scenario_id: String,
        path: String,
        diff: String,
    },

    #[error("config error: {0}")]
    Config(String),
}
```

---

## 6) `src/paths.rs`

This defines canonical repo-relative locations:

* `rust/spec/scenarios/*.json`
* `rust/spec/goldens/*.json`
* `rust/spec/normalizer/v1.yaml`

```rust
use crate::error::{Result, RunnerError};
use std::path::{Path, PathBuf};

#[derive(Debug, Clone)]
pub struct RepoPaths {
    pub repo_root: PathBuf,
    pub rust_root: PathBuf,
    pub spec_root: PathBuf,
    pub scenarios_dir: PathBuf,
    pub goldens_dir: PathBuf,
    pub normalizer_base: PathBuf,
}

impl RepoPaths {
    /// Discovers repo_root from a starting directory by walking upward until
    /// we find a `rust/` directory and (optionally) `.git/`.
    pub fn discover(start: &Path) -> Result<Self> {
        let mut cur = start.to_path_buf();
        loop {
            let rust_root = cur.join("rust");
            let spec_root = rust_root.join("spec");

            if rust_root.is_dir() && spec_root.is_dir() {
                let scenarios_dir = spec_root.join("scenarios");
                let goldens_dir = spec_root.join("goldens");
                let normalizer_base = spec_root.join("normalizer").join("v1.yaml");

                return Ok(Self {
                    repo_root: cur.clone(),
                    rust_root,
                    spec_root,
                    scenarios_dir,
                    goldens_dir,
                    normalizer_base,
                });
            }

            if !cur.pop() {
                break;
            }
        }

        Err(RunnerError::RepoDiscovery(format!(
            "could not locate repo root from start dir '{}'",
            start.display()
        )))
    }

    pub fn scenario_path_by_id(&self, scenario_id: &str) -> PathBuf {
        self.scenarios_dir.join(format!("{scenario_id}.json"))
    }

    pub fn scenario_normalize_override_path(&self, scenario_id: &str) -> PathBuf {
        self.scenarios_dir
            .join(format!("{scenario_id}.normalize.yaml"))
    }

    pub fn golden_path_by_id(&self, scenario_id: &str) -> PathBuf {
        self.goldens_dir.join(format!("{scenario_id}.json"))
    }

    /// Default location for a Python helper script (you will create this in-repo).
    pub fn default_python_engine_script(&self) -> PathBuf {
        self.rust_root.join("tools").join("python_golden_engine.py")
    }
}
```

---

## 7) `src/scenario.rs`

This keeps the runner flexible: treat scenarios as JSON documents, only extracting the fields required for orchestration.

```rust
use crate::error::{Result, RunnerError};
use sha2::{Digest, Sha256};
use std::fs;
use std::path::{Path, PathBuf};

#[derive(Debug, Clone)]
pub struct ScenarioDoc {
    pub path: PathBuf,
    pub json: serde_json::Value,
    pub scenario_id: String,
    pub spec_version: String,
    pub sha256_hex: String,
}

impl ScenarioDoc {
    pub fn load(path: &Path) -> Result<Self> {
        let bytes = fs::read(path)?;
        let sha256_hex = {
            let mut h = Sha256::new();
            h.update(&bytes);
            hex::encode(h.finalize())
        };

        let json: serde_json::Value = serde_json::from_slice(&bytes)?;

        let scenario_id = json
            .get("scenario_id")
            .and_then(|v| v.as_str())
            .ok_or_else(|| RunnerError::Scenario(format!("missing or invalid scenario_id in {}", path.display())))?
            .to_owned();

        let spec_version = json
            .get("spec_version")
            .and_then(|v| v.as_str())
            .ok_or_else(|| RunnerError::Scenario(format!("missing or invalid spec_version in {}", path.display())))?
            .to_owned();

        Ok(Self {
            path: path.to_path_buf(),
            json,
            scenario_id,
            spec_version,
            sha256_hex,
        })
    }
}
```

---

## 8) `src/engines/mod.rs`

Trait-based engine dispatch: Python reference and Rust SUT.

```rust
use crate::error::Result;
use crate::scenario::ScenarioDoc;
use std::collections::HashMap;
use std::path::{Path, PathBuf};
use std::sync::Arc;

pub mod python;
pub mod rust;

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum EngineKind {
    Python,
    Rust,
}

impl EngineKind {
    pub fn as_str(&self) -> &'static str {
        match self {
            EngineKind::Python => "python",
            EngineKind::Rust => "rust",
        }
    }
}

#[derive(Debug, Clone)]
pub struct EngineInput<'a> {
    pub scenario: &'a ScenarioDoc,
    pub repo_root: &'a Path,
    pub run_dir: &'a Path,
    pub extra_env: HashMap<String, String>,
}

#[derive(Debug, Clone)]
pub struct EngineOutput {
    pub report_json: serde_json::Value,
    pub stdout: String,
    pub stderr: String,
}

pub trait Engine: Send + Sync {
    fn id(&self) -> &'static str;

    fn generate_run_report(&self, input: EngineInput<'_>) -> Result<EngineOutput>;
}

#[derive(Debug, Clone)]
pub struct EngineRegistry {
    engines: HashMap<String, Arc<dyn Engine>>,
}

impl EngineRegistry {
    pub fn new() -> Self {
        Self { engines: HashMap::new() }
    }

    pub fn with_defaults(paths: &crate::paths::RepoPaths) -> Self {
        let mut reg = Self::new();

        reg.register(Arc::new(python::PythonEngine::new(
            python::PythonEngineConfig {
                python_bin: "python3".to_owned(),
                script_path: paths.default_python_engine_script(),
                extra_env: HashMap::new(),
            },
        )));

        reg.register(Arc::new(rust::RustEngine::new()));

        reg
    }

    pub fn register(&mut self, engine: Arc<dyn Engine>) {
        self.engines.insert(engine.id().to_owned(), engine);
    }

    pub fn get(&self, id: &str) -> Option<Arc<dyn Engine>> {
        self.engines.get(id).cloned()
    }
}
```

---

## 9) `src/engines/python.rs`

Subprocess-based engine. You’ll implement `rust/tools/python_golden_engine.py` to accept args and emit JSON on stdout.

```rust
use crate::engines::{Engine, EngineInput, EngineOutput};
use crate::error::{Result, RunnerError};

use std::collections::HashMap;
use std::process::Command;

#[derive(Debug, Clone)]
pub struct PythonEngineConfig {
    pub python_bin: String,
    pub script_path: std::path::PathBuf,
    pub extra_env: HashMap<String, String>,
}

#[derive(Debug, Clone)]
pub struct PythonEngine {
    cfg: PythonEngineConfig,
}

impl PythonEngine {
    pub fn new(cfg: PythonEngineConfig) -> Self {
        Self { cfg }
    }
}

impl Engine for PythonEngine {
    fn id(&self) -> &'static str {
        "python"
    }

    fn generate_run_report(&self, input: EngineInput<'_>) -> Result<EngineOutput> {
        // Protocol (you implement python_golden_engine.py):
        //   python3 python_golden_engine.py \
        //     --scenario <path> \
        //     --repo-root <repo_root> \
        //     --run-dir <run_dir>
        //
        // stdout: JSON run_report (canonical schema)
        // stderr: debug logs
        let mut cmd = Command::new(&self.cfg.python_bin);
        cmd.arg(&self.cfg.script_path)
            .arg("--scenario")
            .arg(&input.scenario.path)
            .arg("--repo-root")
            .arg(input.repo_root)
            .arg("--run-dir")
            .arg(input.run_dir);

        // Ensure python can import from repo if needed.
        cmd.env("PYTHONPATH", input.repo_root);

        // Merge env maps: engine cfg + request-specific.
        for (k, v) in &self.cfg.extra_env {
            cmd.env(k, v);
        }
        for (k, v) in &input.extra_env {
            cmd.env(k, v);
        }

        let output = cmd.output().map_err(|e| RunnerError::EngineFailure {
            engine: self.id().to_owned(),
            details: format!("failed to spawn python: {e}"),
        })?;

        let stdout = String::from_utf8_lossy(&output.stdout).to_string();
        let stderr = String::from_utf8_lossy(&output.stderr).to_string();

        if !output.status.success() {
            return Err(RunnerError::EngineFailure {
                engine: self.id().to_owned(),
                details: format!("python exit {:?}\nstderr:\n{}", output.status.code(), stderr),
            });
        }

        let report_json: serde_json::Value = serde_json::from_slice(&output.stdout).map_err(|e| {
            RunnerError::EngineFailure {
                engine: self.id().to_owned(),
                details: format!("python returned non-JSON stdout: {e}\nstdout:\n{stdout}\nstderr:\n{stderr}"),
            }
        })?;

        Ok(EngineOutput {
            report_json,
            stdout,
            stderr,
        })
    }
}
```

---

## 10) `src/engines/rust.rs`

Stub until your Rust Burr runtime exists.

```rust
use crate::engines::{Engine, EngineInput, EngineOutput};
use crate::error::{Result, RunnerError};

#[derive(Debug, Clone)]
pub struct RustEngine;

impl RustEngine {
    pub fn new() -> Self {
        Self
    }
}

impl Engine for RustEngine {
    fn id(&self) -> &'static str {
        "rust"
    }

    fn generate_run_report(&self, _input: EngineInput<'_>) -> Result<EngineOutput> {
        Err(RunnerError::EngineFailure {
            engine: self.id().to_owned(),
            details: "Rust engine not implemented yet (wire this into burr-runtime once available)".to_owned(),
        })
    }
}
```

---

## 11) `src/normalize.rs`

Loads the base normalizer config and merges scenario-local overrides, then normalizes a report via `burr-normalizer`.

```rust
use crate::error::Result;
use crate::paths::RepoPaths;
use crate::scenario::ScenarioDoc;

use burr_normalizer::config::{NormalizerConfig, NormalizerOverride};
use burr_normalizer::merge::merge_config;
use burr_normalizer::Normalizer;

use std::fs;
use std::path::Path;

#[derive(Debug, Clone)]
pub struct NormalizationPipeline {
    pub base_config_path: std::path::PathBuf,
}

impl NormalizationPipeline {
    pub fn new(base_config_path: std::path::PathBuf) -> Self {
        Self { base_config_path }
    }

    pub fn for_scenario(paths: &RepoPaths, scenario: &ScenarioDoc) -> Result<Normalizer> {
        // Load base YAML
        let base_yaml = fs::read_to_string(&paths.normalizer_base)?;
        let base: NormalizerConfig = serde_yaml::from_str(&base_yaml)?;

        // Optional override YAML
        let override_path = paths.scenario_normalize_override_path(&scenario.scenario_id);
        let merged = if override_path.exists() {
            let ov_yaml = fs::read_to_string(&override_path)?;
            let ov: NormalizerOverride = serde_yaml::from_str(&ov_yaml)?;
            merge_config(base, ov)?
        } else {
            base
        };

        Ok(Normalizer::new(merged))
    }

    pub fn normalize_to_string(normalizer: &Normalizer, report: serde_json::Value) -> Result<String> {
        Ok(normalizer.normalize_to_string(report)?)
    }

    pub fn normalize_file_to_string(normalizer: &Normalizer, path: &Path) -> Result<String> {
        let bytes = fs::read(path)?;
        let json: serde_json::Value = serde_json::from_slice(&bytes)?;
        Self::normalize_to_string(normalizer, json)
    }
}
```

---

## 12) `src/goldens.rs`

```rust
use crate::error::{Result, RunnerError};
use std::fs;
use std::path::{Path, PathBuf};

#[derive(Debug, Clone)]
pub struct GoldenStore {
    pub dir: PathBuf,
}

impl GoldenStore {
    pub fn new(dir: PathBuf) -> Self {
        Self { dir }
    }

    pub fn golden_path(&self, scenario_id: &str) -> PathBuf {
        self.dir.join(format!("{scenario_id}.json"))
    }

    pub fn read(&self, scenario_id: &str) -> Result<String> {
        let path = self.golden_path(scenario_id);
        if !path.exists() {
            return Err(RunnerError::GoldenMissing {
                scenario_id: scenario_id.to_owned(),
                path: path.display().to_string(),
            });
        }
        Ok(fs::read_to_string(path)?)
    }

    pub fn write(&self, scenario_id: &str, contents: &str) -> Result<()> {
        fs::create_dir_all(&self.dir)?;
        let path = self.golden_path(scenario_id);
        fs::write(path, contents)?;
        Ok(())
    }
}
```

---

## 13) `src/diff.rs`

Uses `similar::TextDiff` and its unified diff formatter (which implements `Display`). ([Docs.rs][1])

```rust
use similar::TextDiff;

pub fn unified_diff(expected: &str, actual: &str, expected_name: &str, actual_name: &str) -> String {
    let diff = TextDiff::from_lines(expected, actual);
    diff.unified_diff()
        .header(expected_name, actual_name)
        .to_string()
}
```

---

## 14) `src/runner.rs`

This is the orchestration layer.

```rust
use crate::diff;
use crate::engines::{EngineInput, EngineKind, EngineRegistry};
use crate::error::{Result, RunnerError};
use crate::goldens::GoldenStore;
use crate::normalize::NormalizationPipeline;
use crate::paths::RepoPaths;
use crate::scenario::ScenarioDoc;

use std::collections::HashMap;
use std::fs;
use std::path::{Path, PathBuf};

#[derive(Debug, Clone)]
pub struct RunRequest {
    pub repo_root: Option<PathBuf>,
    pub scenario_id: Option<String>,
    pub all: bool,

    /// Which engine to run (rust produces SUT output; python produces reference output).
    pub engine: EngineKind,

    /// If set, writes normalized output to the golden file (instead of comparing).
    pub update_goldens: bool,

    /// If set, keep run dirs even when using a temp directory (helpful for debugging).
    pub keep_run_dir: bool,

    /// If provided, use this directory for artifacts instead of a temp dir.
    pub run_dir: Option<PathBuf>,

    /// Optional environment passed to engine subprocesses.
    pub extra_env: HashMap<String, String>,
}

impl Default for RunRequest {
    fn default() -> Self {
        Self {
            repo_root: None,
            scenario_id: None,
            all: false,
            engine: EngineKind::Rust,
            update_goldens: false,
            keep_run_dir: false,
            run_dir: None,
            extra_env: HashMap::new(),
        }
    }
}

#[derive(Debug, Clone)]
pub struct ScenarioOutcome {
    pub scenario_id: String,
    pub ok: bool,
    pub run_dir: PathBuf,
    pub golden_path: PathBuf,
}

#[derive(Debug, Clone)]
pub struct RunSummary {
    pub outcomes: Vec<ScenarioOutcome>,
}

pub struct GoldenRunner {
    pub paths: RepoPaths,
    pub engines: EngineRegistry,
    pub goldens: GoldenStore,
}

impl GoldenRunner {
    pub fn new(paths: RepoPaths) -> Self {
        let engines = EngineRegistry::with_defaults(&paths);
        let goldens = GoldenStore::new(paths.goldens_dir.clone());
        Self { paths, engines, goldens }
    }

    pub fn run(&self, req: RunRequest) -> Result<RunSummary> {
        let repo_root = req.repo_root.clone().unwrap_or_else(|| self.paths.repo_root.clone());

        let scenarios = self.resolve_scenarios(&req)?;
        let engine_id = req.engine.as_str();

        let engine = self.engines.get(engine_id).ok_or_else(|| RunnerError::Config(format!(
            "engine '{engine_id}' not registered"
        )))?;

        let mut outcomes = Vec::new();

        for scenario in scenarios {
            // Create run directory
            let (run_dir, _temp_guard) = self.make_run_dir(&scenario, &req)?;

            // Run engine
            let out = engine.generate_run_report(EngineInput {
                scenario: &scenario,
                repo_root: &repo_root,
                run_dir: &run_dir,
                extra_env: req.extra_env.clone(),
            })?;

            // Persist raw artifacts for debugging
            fs::create_dir_all(&run_dir)?;
            fs::write(run_dir.join("engine.stdout.txt"), &out.stdout)?;
            fs::write(run_dir.join("engine.stderr.txt"), &out.stderr)?;
            fs::write(
                run_dir.join("raw_run_report.json"),
                serde_json::to_string_pretty(&out.report_json)?,
            )?;

            // Normalize (base + scenario override)
            let normalizer = NormalizationPipeline::for_scenario(&self.paths, &scenario)?;
            let normalized = NormalizationPipeline::normalize_to_string(&normalizer, out.report_json)?;

            fs::write(run_dir.join("normalized_run_report.json"), &normalized)?;

            // Compare / update
            let golden_path = self.paths.golden_path_by_id(&scenario.scenario_id);

            if req.update_goldens {
                self.goldens.write(&scenario.scenario_id, &normalized)?;
                outcomes.push(ScenarioOutcome {
                    scenario_id: scenario.scenario_id,
                    ok: true,
                    run_dir,
                    golden_path,
                });
                continue;
            }

            let expected = self.goldens.read(&scenario.scenario_id)?;
            if expected != normalized {
                let udiff = diff::unified_diff(
                    &expected,
                    &normalized,
                    &format!("golden:{}", golden_path.display()),
                    "actual:normalized_run_report.json",
                );

                // Also write copies for tooling
                fs::write(run_dir.join("expected.golden.json"), &expected)?;
                fs::write(run_dir.join("actual.normalized.json"), &normalized)?;

                return Err(RunnerError::GoldenMismatch {
                    scenario_id: scenario.scenario_id,
                    path: golden_path.display().to_string(),
                    diff: udiff,
                });
            }

            outcomes.push(ScenarioOutcome {
                scenario_id: scenario.scenario_id,
                ok: true,
                run_dir,
                golden_path,
            });
        }

        Ok(RunSummary { outcomes })
    }

    fn resolve_scenarios(&self, req: &RunRequest) -> Result<Vec<ScenarioDoc>> {
        if req.all {
            return self.load_all_scenarios();
        }
        if let Some(id) = &req.scenario_id {
            let path = self.paths.scenario_path_by_id(id);
            if !path.exists() {
                return Err(RunnerError::Scenario(format!(
                    "scenario_id '{}' not found at {}",
                    id,
                    path.display()
                )));
            }
            return Ok(vec![ScenarioDoc::load(&path)?]);
        }
        Err(RunnerError::Scenario(
            "either --all or --scenario <id> must be provided".to_owned(),
        ))
    }

    fn load_all_scenarios(&self) -> Result<Vec<ScenarioDoc>> {
        let mut out = Vec::new();
        for entry in walkdir::WalkDir::new(&self.paths.scenarios_dir)
            .into_iter()
            .filter_map(|e| e.ok())
        {
            let p = entry.path();
            if p.is_file() && p.extension().and_then(|s| s.to_str()) == Some("json") {
                out.push(ScenarioDoc::load(p)?);
            }
        }
        out.sort_by(|a, b| a.scenario_id.cmp(&b.scenario_id));
        Ok(out)
    }

    fn make_run_dir(&self, scenario: &ScenarioDoc, req: &RunRequest) -> Result<(PathBuf, Option<tempfile::TempDir>)> {
        if let Some(dir) = &req.run_dir {
            let run_dir = dir.join(&scenario.scenario_id);
            fs::create_dir_all(&run_dir)?;
            return Ok((run_dir, None));
        }

        // TempDir is removed on drop unless kept.
        let td = tempfile::tempdir()?;
        let run_dir = td.path().join(&scenario.scenario_id);
        fs::create_dir_all(&run_dir)?;

        if req.keep_run_dir {
            // Persist it by "leaking" the temp dir; for debugging only.
            // Caller should prefer --run-dir in normal use.
            let path = td.into_path();
            let run_dir = path.join(&scenario.scenario_id);
            return Ok((run_dir, None));
        }

        Ok((run_dir, Some(td)))
    }
}
```

---

## 15) `src/cli.rs`

Uses `clap` derive with subcommands. ([Docs.rs][2])

```rust
use crate::error::{Result, RunnerError};
use crate::engines::EngineKind;
use crate::paths::RepoPaths;
use crate::runner::{GoldenRunner, RunRequest};

use clap::{Parser, Subcommand};
use std::path::PathBuf;

#[derive(Debug, Parser)]
#[command(name = "burr-golden", version, about = "Burr golden-test runner")]
#[command(propagate_version = true)]
pub struct Cli {
    #[command(subcommand)]
    pub cmd: Command,
}

#[derive(Debug, Subcommand)]
pub enum Command {
    /// Run engine output and compare to goldens (or update with --update).
    Run {
        /// Scenario id (filename without .json)
        #[arg(long)]
        scenario: Option<String>,

        /// Run all scenarios under rust/spec/scenarios
        #[arg(long)]
        all: bool,

        /// Engine to run: 'rust' (SUT) or 'python' (reference)
        #[arg(long, default_value = "rust")]
        engine: String,

        /// Update golden files instead of comparing
        #[arg(long)]
        update: bool,

        /// Keep temporary run directory (debugging)
        #[arg(long)]
        keep_run_dir: bool,

        /// Use an explicit output directory for run artifacts
        #[arg(long)]
        run_dir: Option<PathBuf>,

        /// Repo root override (defaults to discovery)
        #[arg(long)]
        repo_root: Option<PathBuf>,
    },

    /// Normalize an arbitrary run_report.json file (useful for debugging)
    Normalize {
        #[arg(long)]
        input: PathBuf,

        #[arg(long)]
        output: Option<PathBuf>,

        #[arg(long)]
        repo_root: Option<PathBuf>,

        /// Optional scenario id for applying scenario-local override rules
        #[arg(long)]
        scenario: Option<String>,
    },

    /// List scenario ids found in rust/spec/scenarios
    List {
        #[arg(long)]
        repo_root: Option<PathBuf>,
    },
}

pub fn main_entry() -> i32 {
    match run_cli() {
        Ok(()) => 0,
        Err(e) => {
            eprintln!("{e}");
            1
        }
    }
}

fn run_cli() -> Result<()> {
    let cli = Cli::parse();

    match cli.cmd {
        Command::Run {
            scenario,
            all,
            engine,
            update,
            keep_run_dir,
            run_dir,
            repo_root,
        } => {
            let start = repo_root
                .clone()
                .unwrap_or(std::env::current_dir().map_err(RunnerError::from)?);
            let paths = RepoPaths::discover(&start)?;
            let runner = GoldenRunner::new(paths);

            let engine_kind = parse_engine(&engine)?;

            let req = RunRequest {
                repo_root,
                scenario_id: scenario,
                all,
                engine: engine_kind,
                update_goldens: update,
                keep_run_dir,
                run_dir,
                extra_env: Default::default(),
            };

            let _summary = runner.run(req)?;
            Ok(())
        }

        Command::Normalize { input, output, repo_root, scenario } => {
            let start = repo_root
                .clone()
                .unwrap_or(std::env::current_dir().map_err(RunnerError::from)?);
            let paths = RepoPaths::discover(&start)?;

            // If scenario is provided, we load its doc to locate override file + merge.
            // Otherwise normalize with base config only.
            let normalizer = if let Some(sid) = scenario {
                let s_path = paths.scenario_path_by_id(&sid);
                let s_doc = crate::scenario::ScenarioDoc::load(&s_path)?;
                crate::normalize::NormalizationPipeline::for_scenario(&paths, &s_doc)?
            } else {
                // base-only
                let base_yaml = std::fs::read_to_string(&paths.normalizer_base)?;
                let base: burr_normalizer::config::NormalizerConfig = serde_yaml::from_str(&base_yaml)?;
                burr_normalizer::Normalizer::new(base)
            };

            let normalized = crate::normalize::NormalizationPipeline::normalize_file_to_string(&normalizer, &input)?;
            match output {
                Some(out) => std::fs::write(out, normalized)?,
                None => print!("{normalized}"),
            }
            Ok(())
        }

        Command::List { repo_root } => {
            let start = repo_root
                .unwrap_or(std::env::current_dir().map_err(RunnerError::from)?);
            let paths = RepoPaths::discover(&start)?;
            let runner = GoldenRunner::new(paths);
            let scenarios = runner.paths.scenarios_dir.clone();

            for entry in walkdir::WalkDir::new(&scenarios).into_iter().filter_map(|e| e.ok()) {
                let p = entry.path();
                if p.is_file() && p.extension().and_then(|s| s.to_str()) == Some("json") {
                    if let Some(stem) = p.file_stem().and_then(|s| s.to_str()) {
                        println!("{stem}");
                    }
                }
            }
            Ok(())
        }
    }
}

fn parse_engine(s: &str) -> Result<EngineKind> {
    match s {
        "python" => Ok(EngineKind::Python),
        "rust" => Ok(EngineKind::Rust),
        other => Err(RunnerError::Config(format!(
            "unknown engine '{other}' (expected 'python' or 'rust')"
        ))),
    }
}
```

---

## 16) `tests/cli_smoke.rs`

A minimal CLI test using `assert_cmd`, which is specifically designed to help test binaries. ([Docs.rs][3])

```rust
use assert_cmd::Command;

#[test]
fn help_works() {
    let mut cmd = Command::cargo_bin("burr-golden").unwrap();
    cmd.arg("--help").assert().success();
}
```

---

## 17) The Python engine script contract (what you’ll implement next)

Create: `rust/tools/python_golden_engine.py` with this **stable contract**:

* Args:

    * `--scenario <path>`: scenario JSON path
    * `--repo-root <path>`
    * `--run-dir <path>`
* Behavior:

    * Loads scenario JSON
    * Runs using Python Burr reference implementation
    * Emits **the full `run_report.json`** to stdout
    * Writes any auxiliary artifacts (tracking/persistence) into `--run-dir` as needed for debugging

This script becomes the “truth source” that generates goldens (initially), while the Rust engine aims to match it.

---

## Notes on why these dependencies are the right fit

* **clap derive** is the fastest way to ship a robust CLI with subcommands and structured args. ([Docs.rs][2])
* **tempfile::TempDir** gives you auto-cleanup run directories (unless you choose to persist for debugging). ([Docs.rs][4])
* **similar::TextDiff + UnifiedDiff** gives you clean unified diffs on mismatch (and `UnifiedDiff` implements `Display`). ([Docs.rs][1])

---

[1]: https://docs.rs/similar/latest/similar/struct.TextDiff.html "TextDiff in similar - Rust"
[2]: https://docs.rs/clap/latest/clap/_derive/_tutorial/index.html "clap::_derive::_tutorial - Rust"
[3]: https://docs.rs/assert_cmd "assert_cmd - Rust"
[4]: https://docs.rs/tempfile/ "tempfile - Rust"
