Below is an **exact Rust crate skeleton** you can drop into your Burr Rust workspace as `burr-normalizer`, with:

* **RFC 9535 JSONPath selection** via `serde_json_path` (it intends to adhere to RFC 9535 and exposes `JsonPath::query_located` → `LocatedNodeList` with `NormalizedPath::to_json_pointer()` so you can mutate `serde_json::Value` with `pointer_mut`). ([Docs.rs][1])
* **RFC 8785 canonical JSON output** via `serde_json_canonicalizer::to_string/to_vec`. ([Docs.rs][2])

It includes modules, config structs, error types, a trait-based op registry/dispatch, plus “good enough” implementations for the ops you sketched (regex replace, prefix replacement, uuid/timestamp mapping, redact, drop, sort array, canonicalize_json).

---

## File tree

Create this at `rust/crates/burr-normalizer/`:

```text
burr-normalizer/
├─ Cargo.toml
├─ src/
│  ├─ lib.rs
│  ├─ config.rs
│  ├─ context.rs
│  ├─ error.rs
│  ├─ jsonpath.rs
│  ├─ merge.rs
│  └─ ops/
│     ├─ mod.rs
│     ├─ args.rs
│     ├─ canonicalize_json.rs
│     ├─ drop.rs
│     ├─ redact.rs
│     ├─ replace_prefixes_from_report.rs
│     ├─ replace_regex.rs
│     ├─ sort_array.rs
│     ├─ timestamp_map.rs
│     └─ uuid_map.rs
└─ tests/
   └─ smoke.rs
```

And add it to your workspace `rust/Cargo.toml`:

```toml
[workspace]
members = [
  "crates/burr-normalizer",
  # ... other crates ...
]
resolver = "2"
```

---

## `Cargo.toml`

```toml
[package]
name = "burr-normalizer"
version = "0.1.0"
edition = "2021"
license = "Apache-2.0"
publish = false
description = "Golden-test normalizer for Apache Burr run_report.json artifacts."

[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
serde_yaml = "0.9"
regex = "1"
thiserror = "1"

# JSONPath (RFC 9535 oriented) selection + located queries
serde_json_path = "0.7"

# RFC 8785 canonical JSON output
serde_json_canonicalizer = "0.3"

[dev-dependencies]
serde_json = "1"
```

---

## `src/lib.rs`

```rust
//! burr-normalizer: stable normalization for Burr golden run_report.json artifacts.
//!
//! Highlights:
//! - JSONPath selection via serde_json_path (RFC 9535 oriented).
//! - Canonical output via serde_json_canonicalizer (RFC 8785 compatible).

pub mod config;
pub mod context;
pub mod error;
pub mod jsonpath;
pub mod merge;
pub mod ops;

use crate::config::{Canonicalization, NormalizerConfig, OutputConfig};
use crate::context::NormalizeContext;
use crate::error::{NormalizeError, Result};
use crate::ops::OpRegistry;

use serde_json::Value;

/// Main entry point for applying a NormalizerConfig to a run_report JSON value.
#[derive(Debug, Clone)]
pub struct Normalizer {
    config: NormalizerConfig,
    registry: OpRegistry,
}

impl Normalizer {
    /// Create a Normalizer with the default op registry.
    pub fn new(config: NormalizerConfig) -> Self {
        Self {
            config,
            registry: OpRegistry::with_defaults(),
        }
    }

    /// Create a Normalizer with a custom op registry (useful for plugins).
    pub fn with_registry(config: NormalizerConfig, registry: OpRegistry) -> Self {
        Self { config, registry }
    }

    pub fn config(&self) -> &NormalizerConfig {
        &self.config
    }

    /// Normalize in-place, returning the context (which may include output overrides).
    pub fn normalize_in_place(&self, report: &mut Value) -> Result<NormalizeContext> {
        let mut ctx = NormalizeContext::new();

        for rule in &self.config.rules {
            let op = self.registry.get(&rule.op).ok_or_else(|| NormalizeError::UnknownOp {
                op: rule.op.clone(),
                rule_id: rule.id.clone(),
            })?;

            op.apply(report, rule, &mut ctx)?;
        }

        Ok(ctx)
    }

    /// Convenience: normalize by value-copy.
    pub fn normalize_to_value(&self, mut report: Value) -> Result<(Value, NormalizeContext)> {
        let ctx = self.normalize_in_place(&mut report)?;
        Ok((report, ctx))
    }

    /// Normalize and render to a string using the (possibly overridden) output config.
    pub fn normalize_to_string(&self, report: Value) -> Result<String> {
        let (report, ctx) = self.normalize_to_value(report)?;
        let out = self.effective_output_config(&ctx);
        render_json(&report, &out)
    }

    fn effective_output_config(&self, ctx: &NormalizeContext) -> OutputConfig {
        let mut out = self.config.output.clone();
        if let Some(c) = ctx.output_overrides.canonicalization.clone() {
            out.canonicalization = c;
        }
        if let Some(p) = ctx.output_overrides.pretty {
            out.pretty = p;
        }
        out
    }
}

/// Render normalized JSON deterministically.
pub fn render_json(value: &Value, out: &OutputConfig) -> Result<String> {
    match out.canonicalization {
        Canonicalization::Rfc8785 => {
            // Canonical output is inherently non-pretty; ignore out.pretty.
            let s = serde_json_canonicalizer::to_string(value)
                .map_err(|e| NormalizeError::Canonicalization(e.to_string()))?;
            Ok(s)
        }
        Canonicalization::None => {
            if out.pretty {
                Ok(serde_json::to_string_pretty(value)?)
            } else {
                Ok(serde_json::to_string(value)?)
            }
        }
    }
}
```

---

## `src/error.rs`

```rust
use thiserror::Error;

pub type Result<T> = std::result::Result<T, NormalizeError>;

#[derive(Debug, Error)]
pub enum NormalizeError {
    #[error("unknown op '{op}' in rule '{rule_id}'")]
    UnknownOp { op: String, rule_id: String },

    #[error("invalid JSONPath '{expr}': {source}")]
    JsonPathParse {
        expr: String,
        #[source]
        source: serde_json_path::ParseError,
    },

    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),

    #[error("YAML parse error: {0}")]
    Yaml(#[from] serde_yaml::Error),

    #[error("JSON parse/serialize error: {0}")]
    Json(#[from] serde_json::Error),

    #[error("regex error: {0}")]
    Regex(#[from] regex::Error),

    #[error("rule '{rule_id}' missing required arg '{arg}'")]
    MissingArg { rule_id: String, arg: &'static str },

    #[error("rule '{rule_id}' arg '{arg}' has invalid type (expected {expected})")]
    InvalidArgType {
        rule_id: String,
        arg: String,
        expected: &'static str,
    },

    #[error("rule '{rule_id}' select yielded incompatible node for op '{op}': {details}")]
    TypeMismatch {
        rule_id: String,
        op: String,
        details: String,
    },

    #[error("config merge error: {0}")]
    Merge(String),

    #[error("canonicalization error: {0}")]
    Canonicalization(String),

    #[error("not implemented: {0}")]
    NotImplemented(String),
}
```

---

## `src/config.rs`

```rust
use serde::{Deserialize, Serialize};
use serde_json::Value as JsonValue;

fn default_args() -> JsonValue {
    JsonValue::Object(serde_json::Map::new())
}

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct NormalizerConfig {
    pub normalizer_version: String,
    pub jsonpath_dialect: JsonPathDialect,
    pub output: OutputConfig,
    pub rules: Vec<Rule>,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct OutputConfig {
    pub canonicalization: Canonicalization,
    pub pretty: bool,
}

#[derive(Debug, Clone, Deserialize, Serialize, PartialEq, Eq)]
pub enum Canonicalization {
    #[serde(rename = "none")]
    None,
    #[serde(rename = "rfc8785")]
    Rfc8785,
}

#[derive(Debug, Clone, Deserialize, Serialize, PartialEq, Eq)]
pub enum JsonPathDialect {
    #[serde(rename = "rfc9535")]
    Rfc9535,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct Rule {
    pub id: String,
    pub op: String,

    #[serde(deserialize_with = "deserialize_select")]
    pub select: SelectExprs,

    #[serde(default)]
    pub when: Option<When>,

    #[serde(default = "default_args")]
    pub args: JsonValue,
}

impl Rule {
    pub fn selectors(&self) -> Vec<&str> {
        match &self.select {
            SelectExprs::One(s) => vec![s.as_str()],
            SelectExprs::Many(v) => v.iter().map(|s| s.as_str()).collect(),
        }
    }
}

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct When {
    #[serde(rename = "type")]
    pub typ: Option<JsonType>,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum JsonType {
    String,
    Number,
    Boolean,
    Null,
    Object,
    Array,
}

#[derive(Debug, Clone, Serialize)]
pub enum SelectExprs {
    One(String),
    Many(Vec<String>),
}

fn deserialize_select<'de, D>(deserializer: D) -> Result<SelectExprs, D::Error>
where
    D: serde::Deserializer<'de>,
{
    // Accept either a string or array of strings.
    #[derive(Deserialize)]
    #[serde(untagged)]
    enum Raw {
        One(String),
        Many(Vec<String>),
    }

    let raw = Raw::deserialize(deserializer)?;
    Ok(match raw {
        Raw::One(s) => SelectExprs::One(s),
        Raw::Many(v) => SelectExprs::Many(v),
    })
}

/* ---------------- Scenario-local override types ---------------- */

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct NormalizerOverride {
    pub normalizer_override_version: String,

    #[serde(default)]
    pub disable_rules: Vec<String>,

    #[serde(default)]
    pub replace_rules: Vec<Rule>,

    #[serde(default)]
    pub add_rules: Vec<AddRule>,

    pub output_override: Option<OutputOverride>,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct AddRule {
    pub rule: Rule,
    pub position: Option<RulePosition>,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct RulePosition {
    pub before: Option<String>,
    pub after: Option<String>,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct OutputOverride {
    pub canonicalization: Option<Canonicalization>,
    pub pretty: Option<bool>,
}
```

---

## `src/context.rs`

```rust
use crate::config::Canonicalization;
use crate::error::{NormalizeError, Result};

use serde_json_path::JsonPath;
use std::collections::HashMap;

#[derive(Debug, Default, Clone)]
pub struct OutputOverrides {
    pub canonicalization: Option<Canonicalization>,
    pub pretty: Option<bool>,
}

/// Per-normalization run context.
/// - Holds JSONPath parse cache.
/// - Holds output overrides set by ops (e.g. canonicalize_json).
#[derive(Debug)]
pub struct NormalizeContext {
    pub output_overrides: OutputOverrides,
    jsonpath_cache: HashMap<String, JsonPath>,
}

impl NormalizeContext {
    pub fn new() -> Self {
        Self {
            output_overrides: OutputOverrides::default(),
            jsonpath_cache: HashMap::new(),
        }
    }

    /// Parse JSONPath once and cache it. (serde_json_path::JsonPath::parse)
    pub fn jsonpath(&mut self, expr: &str) -> Result<&JsonPath> {
        if self.jsonpath_cache.contains_key(expr) {
            // SAFETY: key exists.
            return Ok(self.jsonpath_cache.get(expr).unwrap());
        }

        let parsed = JsonPath::parse(expr).map_err(|e| NormalizeError::JsonPathParse {
            expr: expr.to_owned(),
            source: e,
        })?;

        self.jsonpath_cache.insert(expr.to_owned(), parsed);
        Ok(self.jsonpath_cache.get(expr).unwrap())
    }
}
```

---

## `src/jsonpath.rs`

```rust
//! JSONPath selection helpers.
//!
//! We use `JsonPath::query_located` to get locations (NormalizedPath) and then
//! `NormalizedPath::to_json_pointer()` so we can mutate a `serde_json::Value`
//! via `Value::pointer_mut`.

use crate::config::{JsonType, Rule, When};
use crate::context::NormalizeContext;
use crate::error::Result;

use serde_json::Value;

pub fn select_pointers(report: &Value, rule: &Rule, ctx: &mut NormalizeContext) -> Result<Vec<String>> {
    select_pointers_raw(report, &rule.selectors(), rule.when.as_ref(), ctx)
}

pub fn select_pointers_raw(
    report: &Value,
    selectors: &[&str],
    when: Option<&When>,
    ctx: &mut NormalizeContext,
) -> Result<Vec<String>> {
    let mut pointers: Vec<String> = Vec::new();

    for sel in selectors {
        let jp = ctx.jsonpath(sel)?;
        let located = jp.query_located(report).dedup();
        for loc in located.locations() {
            pointers.push(loc.to_json_pointer());
        }
    }

    // Stable ordering + dedup.
    pointers.sort();
    pointers.dedup();

    if let Some(w) = when {
        if let Some(t) = &w.typ {
            pointers.retain(|ptr| match report.pointer(ptr) {
                Some(v) => matches_type(v, t),
                None => false,
            });
        }
    }

    Ok(pointers)
}

fn matches_type(v: &Value, t: &JsonType) -> bool {
    match t {
        JsonType::String => v.is_string(),
        JsonType::Number => v.is_number(),
        JsonType::Boolean => v.is_boolean(),
        JsonType::Null => v.is_null(),
        JsonType::Object => v.is_object(),
        JsonType::Array => v.is_array(),
    }
}
```

---

## `src/merge.rs`

```rust
use crate::config::{AddRule, NormalizerConfig, NormalizerOverride, OutputOverride, Rule, RulePosition};
use crate::error::{NormalizeError, Result};

pub fn merge_config(mut base: NormalizerConfig, ov: NormalizerOverride) -> Result<NormalizerConfig> {
    // 1) disable
    if !ov.disable_rules.is_empty() {
        base.rules.retain(|r| !ov.disable_rules.contains(&r.id));
    }

    // 2) replace (in-place)
    for rr in ov.replace_rules {
        let idx = base
            .rules
            .iter()
            .position(|r| r.id == rr.id)
            .ok_or_else(|| NormalizeError::Merge(format!("replace_rules: unknown rule id '{}'", rr.id)))?;
        base.rules[idx] = rr;
    }

    // 3) add
    for add in ov.add_rules {
        if base.rules.iter().any(|r| r.id == add.rule.id) {
            return Err(NormalizeError::Merge(format!(
                "add_rules: duplicate rule id '{}'",
                add.rule.id
            )));
        }

        insert_rule(&mut base.rules, add)?;
    }

    // 4) output override (shallow)
    if let Some(OutputOverride {
        canonicalization,
        pretty,
    }) = ov.output_override
    {
        if let Some(c) = canonicalization {
            base.output.canonicalization = c;
        }
        if let Some(p) = pretty {
            base.output.pretty = p;
        }
    }

    Ok(base)
}

fn insert_rule(rules: &mut Vec<Rule>, add: AddRule) -> Result<()> {
    match add.position {
        None => {
            rules.push(add.rule);
            Ok(())
        }
        Some(RulePosition {
            before: Some(b),
            after: _,
        }) => {
            let idx = rules
                .iter()
                .position(|r| r.id == b)
                .ok_or_else(|| NormalizeError::Merge(format!("add_rules.before: unknown id '{}'", b)))?;
            rules.insert(idx, add.rule);
            Ok(())
        }
        Some(RulePosition {
            before: _,
            after: Some(a),
        }) => {
            let idx = rules
                .iter()
                .position(|r| r.id == a)
                .ok_or_else(|| NormalizeError::Merge(format!("add_rules.after: unknown id '{}'", a)))?;
            rules.insert(idx + 1, add.rule);
            Ok(())
        }
        Some(RulePosition {
            before: None,
            after: None,
        }) => {
            rules.push(add.rule);
            Ok(())
        }
    }
}
```

---

## `src/ops/mod.rs`

```rust
pub mod args;
pub mod canonicalize_json;
pub mod drop;
pub mod redact;
pub mod replace_prefixes_from_report;
pub mod replace_regex;
pub mod sort_array;
pub mod timestamp_map;
pub mod uuid_map;

use crate::config::Rule;
use crate::context::NormalizeContext;
use crate::error::Result;

use serde_json::Value;
use std::collections::HashMap;
use std::sync::Arc;

/// Trait-based op dispatch.
pub trait Op: Send + Sync {
    fn name(&self) -> &'static str;
    fn apply(&self, report: &mut Value, rule: &Rule, ctx: &mut NormalizeContext) -> Result<()>;
}

#[derive(Debug, Clone)]
pub struct OpRegistry {
    ops: HashMap<String, Arc<dyn Op>>,
}

impl OpRegistry {
    pub fn new() -> Self {
        Self { ops: HashMap::new() }
    }

    pub fn with_defaults() -> Self {
        let mut reg = Self::new();
        reg.register(Arc::new(replace_regex::ReplaceRegexOp));
        reg.register(Arc::new(replace_prefixes_from_report::ReplacePrefixesFromReportOp));
        reg.register(Arc::new(uuid_map::UuidMapOp));
        reg.register(Arc::new(timestamp_map::TimestampMapOp));
        reg.register(Arc::new(redact::RedactOp));
        reg.register(Arc::new(drop::DropOp));
        reg.register(Arc::new(sort_array::SortArrayOp));
        reg.register(Arc::new(canonicalize_json::CanonicalizeJsonOp));
        reg
    }

    pub fn register(&mut self, op: Arc<dyn Op>) {
        self.ops.insert(op.name().to_owned(), op);
    }

    pub fn get(&self, name: &str) -> Option<&Arc<dyn Op>> {
        self.ops.get(name)
    }
}
```

---

## `src/ops/args.rs`

```rust
use crate::error::{NormalizeError, Result};
use serde::de::DeserializeOwned;
use serde_json::{Map, Value};

#[derive(Debug, Clone)]
pub struct Args<'a> {
    pub rule_id: &'a str,
    pub op: &'a str,
    pub args: &'a Value,
}

impl<'a> Args<'a> {
    pub fn new(rule_id: &'a str, op: &'a str, args: &'a Value) -> Self {
        Self { rule_id, op, args }
    }

    pub fn as_object(&self) -> Result<&'a Map<String, Value>> {
        self.args.as_object().ok_or_else(|| NormalizeError::InvalidArgType {
            rule_id: self.rule_id.to_owned(),
            arg: "args".to_owned(),
            expected: "object",
        })
    }

    pub fn get_required_str(&self, key: &'static str) -> Result<&'a str> {
        let obj = self.as_object()?;
        let v = obj.get(key).ok_or_else(|| NormalizeError::MissingArg {
            rule_id: self.rule_id.to_owned(),
            arg: key,
        })?;
        v.as_str().ok_or_else(|| NormalizeError::InvalidArgType {
            rule_id: self.rule_id.to_owned(),
            arg: key.to_owned(),
            expected: "string",
        })
    }

    pub fn get_optional_str(&self, key: &'static str) -> Result<Option<&'a str>> {
        let obj = self.as_object()?;
        match obj.get(key) {
            None => Ok(None),
            Some(v) => v.as_str().map(Some).ok_or_else(|| NormalizeError::InvalidArgType {
                rule_id: self.rule_id.to_owned(),
                arg: key.to_owned(),
                expected: "string",
            }),
        }
    }

    pub fn get_optional_bool(&self, key: &'static str) -> Result<Option<bool>> {
        let obj = self.as_object()?;
        match obj.get(key) {
            None => Ok(None),
            Some(v) => v.as_bool().ok_or_else(|| NormalizeError::InvalidArgType {
                rule_id: self.rule_id.to_owned(),
                arg: key.to_owned(),
                expected: "boolean",
            }).map(Some),
        }
    }

    pub fn get_optional_value(&self, key: &'static str) -> Result<Option<Value>> {
        let obj = self.as_object()?;
        Ok(obj.get(key).cloned())
    }

    pub fn get_required_string_array(&self, key: &'static str) -> Result<Vec<String>> {
        let obj = self.as_object()?;
        let v = obj.get(key).ok_or_else(|| NormalizeError::MissingArg {
            rule_id: self.rule_id.to_owned(),
            arg: key,
        })?;

        let arr = v.as_array().ok_or_else(|| NormalizeError::InvalidArgType {
            rule_id: self.rule_id.to_owned(),
            arg: key.to_owned(),
            expected: "array of strings",
        })?;

        let mut out = Vec::with_capacity(arr.len());
        for item in arr {
            let s = item.as_str().ok_or_else(|| NormalizeError::InvalidArgType {
                rule_id: self.rule_id.to_owned(),
                arg: key.to_owned(),
                expected: "array of strings",
            })?;
            out.push(s.to_owned());
        }
        Ok(out)
    }

    /// Deserialize args into a typed struct (clone-based; good enough for config-scale data).
    pub fn deserialize<T: DeserializeOwned>(&self) -> Result<T> {
        serde_json::from_value(self.args.clone()).map_err(NormalizeError::from)
    }
}
```

---

## `src/ops/replace_regex.rs`

```rust
use crate::config::Rule;
use crate::context::NormalizeContext;
use crate::error::Result;
use crate::jsonpath;
use crate::ops::{args::Args, Op};

use regex::Regex;
use serde_json::Value;

#[derive(Debug)]
pub struct ReplaceRegexOp;

impl Op for ReplaceRegexOp {
    fn name(&self) -> &'static str {
        "replace_regex"
    }

    fn apply(&self, report: &mut Value, rule: &Rule, ctx: &mut NormalizeContext) -> Result<()> {
        let a = Args::new(&rule.id, self.name(), &rule.args);
        let pattern = a.get_required_str("pattern")?;
        let replace = a.get_required_str("replace")?;

        let re = Regex::new(pattern)?;
        let pointers = jsonpath::select_pointers(report, rule, ctx)?;

        for ptr in pointers {
            if let Some(v) = report.pointer_mut(&ptr) {
                if let Some(s) = v.as_str() {
                    let replaced = re.replace_all(s, replace).into_owned();
                    *v = Value::String(replaced);
                }
            }
        }
        Ok(())
    }
}
```

---

## `src/ops/replace_prefixes_from_report.rs`

```rust
use crate::config::Rule;
use crate::context::NormalizeContext;
use crate::error::Result;
use crate::jsonpath;
use crate::ops::{args::Args, Op};

use serde::Deserialize;
use serde_json::Value;

#[derive(Debug)]
pub struct ReplacePrefixesFromReportOp;

#[derive(Debug, Deserialize)]
struct PrefixSpec {
    source: String,
    placeholder: String,
}

#[derive(Debug, Deserialize)]
#[serde(rename_all = "snake_case")]
enum Mode {
    Prefix,
    Anywhere,
}

fn default_mode() -> Mode {
    Mode::Anywhere
}

#[derive(Debug, Deserialize)]
struct ReplacePrefixesArgs {
    #[serde(default)]
    prefixes: Vec<PrefixSpec>,
    #[serde(default = "default_mode")]
    mode: Mode,
}

impl Op for ReplacePrefixesFromReportOp {
    fn name(&self) -> &'static str {
        "replace_prefixes_from_report"
    }

    fn apply(&self, report: &mut Value, rule: &Rule, ctx: &mut NormalizeContext) -> Result<()> {
        let a = Args::new(&rule.id, self.name(), &rule.args);
        let args: ReplacePrefixesArgs = a.deserialize()?;

        // Resolve prefixes from the report (best-effort: first node only).
        let mut resolved: Vec<(String, String)> = Vec::new();
        for p in args.prefixes {
            let jp = ctx.jsonpath(&p.source)?;
            let nodes = jp.query(report);
            if let Some(v) = nodes.first() {
                if let Some(s) = v.as_str() {
                    let prefix = normalize_prefix(s);
                    if !prefix.is_empty() {
                        resolved.push((prefix, p.placeholder));
                    }
                }
            }
        }

        // Longest-first prevents partial replacement when prefixes nest.
        resolved.sort_by_key(|(p, _)| std::cmp::Reverse(p.len()));

        let pointers = jsonpath::select_pointers(report, rule, ctx)?;
        for ptr in pointers {
            if let Some(v) = report.pointer_mut(&ptr) {
                if let Some(s) = v.as_str() {
                    let mut out = slashify(s);

                    for (prefix, ph) in &resolved {
                        match args.mode {
                            Mode::Prefix => {
                                if out.starts_with(prefix) {
                                    out = format!("{}{}", ph, &out[prefix.len()..]);
                                }
                            }
                            Mode::Anywhere => {
                                out = out.replace(prefix, ph);
                            }
                        }
                    }

                    *v = Value::String(out);
                }
            }
        }

        Ok(())
    }
}

fn slashify(s: &str) -> String {
    s.replace('\\', "/")
}

fn normalize_prefix(p: &str) -> String {
    let mut p = slashify(p);
    while p.ends_with('/') && p.len() > 1 {
        p.pop();
    }
    p
}
```

---

## `src/ops/uuid_map.rs`

```rust
use crate::config::Rule;
use crate::context::NormalizeContext;
use crate::error::Result;
use crate::jsonpath;
use crate::ops::{args::Args, Op};

use regex::Regex;
use serde_json::Value;
use std::collections::{BTreeMap, BTreeSet};

#[derive(Debug)]
pub struct UuidMapOp;

impl Op for UuidMapOp {
    fn name(&self) -> &'static str {
        "uuid_map"
    }

    fn apply(&self, report: &mut Value, rule: &Rule, ctx: &mut NormalizeContext) -> Result<()> {
        let a = Args::new(&rule.id, self.name(), &rule.args);
        let placeholder_tpl = a.get_optional_str("placeholder")?.unwrap_or("<UUID#{n}>");

        let re = Regex::new(r"(?i)\b[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}\b")?;

        let pointers = jsonpath::select_pointers(report, rule, ctx)?;
        let mut strings: Vec<String> = Vec::new();

        for ptr in &pointers {
            if let Some(v) = report.pointer(ptr) {
                if let Some(s) = v.as_str() {
                    strings.push(s.to_owned());
                }
            }
        }

        // Stable mapping: sort unique UUIDs.
        let mut found: BTreeSet<String> = BTreeSet::new();
        for s in &strings {
            for m in re.find_iter(s) {
                found.insert(m.as_str().to_ascii_lowercase());
            }
        }

        let mut map: BTreeMap<String, String> = BTreeMap::new();
        for (i, u) in found.into_iter().enumerate() {
            let n = i + 1;
            map.insert(u, format_placeholder(placeholder_tpl, n));
        }

        for ptr in pointers {
            if let Some(v) = report.pointer_mut(&ptr) {
                if let Some(s) = v.as_str() {
                    let out = re
                        .replace_all(s, |caps: &regex::Captures| {
                            let key = caps.get(0).unwrap().as_str().to_ascii_lowercase();
                            map.get(&key)
                                .cloned()
                                .unwrap_or_else(|| caps.get(0).unwrap().as_str().to_owned())
                        })
                        .into_owned();
                    *v = Value::String(out);
                }
            }
        }

        Ok(())
    }
}

fn format_placeholder(tpl: &str, n: usize) -> String {
    tpl.replace("{n:04}", &format!("{:04}", n))
        .replace("{n}", &n.to_string())
}
```

---

## `src/ops/timestamp_map.rs`

```rust
use crate::config::Rule;
use crate::context::NormalizeContext;
use crate::error::Result;
use crate::jsonpath;
use crate::ops::{args::Args, Op};

use regex::Regex;
use serde_json::Value;
use std::collections::{BTreeMap, BTreeSet};

#[derive(Debug)]
pub struct TimestampMapOp;

impl Op for TimestampMapOp {
    fn name(&self) -> &'static str {
        "timestamp_map"
    }

    fn apply(&self, report: &mut Value, rule: &Rule, ctx: &mut NormalizeContext) -> Result<()> {
        let a = Args::new(&rule.id, self.name(), &rule.args);
        let placeholder_tpl = a.get_optional_str("placeholder")?.unwrap_or("<TS#{n:04}>");

        let re = Regex::new(
            r"\b\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}(?:\.\d{1,9})?(?:Z|[+-]\d{2}:\d{2})?\b",
        )?;

        let pointers = jsonpath::select_pointers(report, rule, ctx)?;
        let mut strings: Vec<String> = Vec::new();

        for ptr in &pointers {
            if let Some(v) = report.pointer(ptr) {
                if let Some(s) = v.as_str() {
                    strings.push(s.to_owned());
                }
            }
        }

        // Stable mapping: sort unique timestamp strings.
        let mut found: BTreeSet<String> = BTreeSet::new();
        for s in &strings {
            for m in re.find_iter(s) {
                found.insert(m.as_str().to_owned());
            }
        }

        let mut map: BTreeMap<String, String> = BTreeMap::new();
        for (i, ts) in found.into_iter().enumerate() {
            let n = i + 1;
            map.insert(ts, format_placeholder(placeholder_tpl, n));
        }

        for ptr in pointers {
            if let Some(v) = report.pointer_mut(&ptr) {
                if let Some(s) = v.as_str() {
                    let out = re
                        .replace_all(s, |caps: &regex::Captures| {
                            let key = caps.get(0).unwrap().as_str();
                            map.get(key)
                                .cloned()
                                .unwrap_or_else(|| caps.get(0).unwrap().as_str().to_owned())
                        })
                        .into_owned();
                    *v = Value::String(out);
                }
            }
        }

        Ok(())
    }
}

fn format_placeholder(tpl: &str, n: usize) -> String {
    tpl.replace("{n:04}", &format!("{:04}", n))
        .replace("{n}", &n.to_string())
}
```

---

## `src/ops/redact.rs`

```rust
use crate::config::Rule;
use crate::context::NormalizeContext;
use crate::error::Result;
use crate::jsonpath;
use crate::ops::{args::Args, Op};

use serde_json::Value;

#[derive(Debug)]
pub struct RedactOp;

impl Op for RedactOp {
    fn name(&self) -> &'static str {
        "redact"
    }

    fn apply(&self, report: &mut Value, rule: &Rule, ctx: &mut NormalizeContext) -> Result<()> {
        let a = Args::new(&rule.id, self.name(), &rule.args);
        let replacement = a
            .get_optional_value("replace")?
            .unwrap_or_else(|| Value::String("<REDACTED>".to_owned()));

        let pointers = jsonpath::select_pointers(report, rule, ctx)?;
        for ptr in pointers {
            if let Some(v) = report.pointer_mut(&ptr) {
                *v = replacement.clone();
            }
        }
        Ok(())
    }
}
```

---

## `src/ops/drop.rs`

```rust
use crate::config::Rule;
use crate::context::NormalizeContext;
use crate::error::Result;
use crate::jsonpath;
use crate::ops::Op;

use serde_json::Value;
use std::collections::BTreeMap;

#[derive(Debug)]
pub struct DropOp;

impl Op for DropOp {
    fn name(&self) -> &'static str {
        "drop"
    }

    fn apply(&self, report: &mut Value, rule: &Rule, ctx: &mut NormalizeContext) -> Result<()> {
        let mut pointers = jsonpath::select_pointers(report, rule, ctx)?;
        // Remove deeper nodes first to avoid invalidating child pointers.
        pointers.sort_by_key(|p| std::cmp::Reverse(p.len()));

        // Group removals by parent pointer for correct array index handling.
        let mut groups: BTreeMap<String, Vec<String>> = BTreeMap::new();
        let mut drop_root = false;

        for ptr in pointers {
            if ptr.is_empty() {
                drop_root = true;
                continue;
            }
            let (parent, token) = split_parent(&ptr);
            groups.entry(parent).or_default().push(token);
        }

        if drop_root {
            *report = Value::Null;
        }

        // Process parents deepest-first (length desc, then lex).
        let mut parents: Vec<String> = groups.keys().cloned().collect();
        parents.sort_by(|a, b| b.len().cmp(&a.len()).then_with(|| a.cmp(b)));

        for parent_ptr in parents {
            let tokens = groups.get(&parent_ptr).cloned().unwrap_or_default();

            if let Some(parent) = report.pointer_mut(&parent_ptr) {
                match parent {
                    Value::Object(map) => {
                        for t in tokens {
                            let key = unescape_token(&t);
                            map.remove(&key);
                        }
                    }
                    Value::Array(arr) => {
                        // Remove indices descending.
                        let mut idxs: Vec<usize> = tokens
                            .into_iter()
                            .filter_map(|t| unescape_token(&t).parse::<usize>().ok())
                            .collect();
                        idxs.sort_by(|a, b| b.cmp(a));
                        idxs.dedup();
                        for i in idxs {
                            if i < arr.len() {
                                arr.remove(i);
                            }
                        }
                    }
                    _ => {
                        // Ignore: cannot drop child from scalar.
                    }
                }
            }
        }

        Ok(())
    }
}

fn split_parent(pointer: &str) -> (String, String) {
    // pointer like "/a/b/0". parent "/a/b", token "0".
    if let Some((parent, token)) = pointer.rsplit_once('/') {
        let parent_ptr = parent.to_owned(); // parent includes leading '/' or empty
        (parent_ptr, token.to_owned())
    } else {
        ("".to_owned(), pointer.to_owned())
    }
}

fn unescape_token(tok: &str) -> String {
    // JSON Pointer unescape: "~1" -> "/", "~0" -> "~"
    tok.replace("~1", "/").replace("~0", "~")
}
```

---

## `src/ops/sort_array.rs`

```rust
use crate::config::Rule;
use crate::context::NormalizeContext;
use crate::error::Result;
use crate::jsonpath;
use crate::ops::{args::Args, Op};

use serde_json::Value;

#[derive(Debug)]
pub struct SortArrayOp;

#[derive(Debug, Clone, PartialEq, Eq, PartialOrd, Ord)]
struct SortAtom {
    kind: u8,
    repr: String,
}

#[derive(Debug, Clone, PartialEq, Eq, PartialOrd, Ord)]
struct SortKey(Vec<SortAtom>);

impl Op for SortArrayOp {
    fn name(&self) -> &'static str {
        "sort_array"
    }

    fn apply(&self, report: &mut Value, rule: &Rule, ctx: &mut NormalizeContext) -> Result<()> {
        let a = Args::new(&rule.id, self.name(), &rule.args);
        let by = a.get_required_string_array("by")?;
        let strict = a.get_optional_bool("strict")?.unwrap_or(false);

        let pointers = jsonpath::select_pointers(report, rule, ctx)?;
        for ptr in pointers {
            let Some(v) = report.pointer_mut(&ptr) else { continue };

            let Value::Array(arr) = v else {
                if strict {
                    // In strict mode, selected nodes must be arrays.
                    // (Otherwise ignore non-array nodes.)
                    // Keep as soft behavior for now.
                }
                continue;
            };

            // Drain -> sort -> rebuild to avoid recomputing keys many times.
            let mut pairs: Vec<(SortKey, Value)> = arr
                .drain(..)
                .map(|el| (make_key(&el, &by), el))
                .collect();

            pairs.sort_by(|a, b| a.0.cmp(&b.0));
            *arr = pairs.into_iter().map(|(_, el)| el).collect();
        }

        Ok(())
    }
}

fn make_key(el: &Value, by: &[String]) -> SortKey {
    let mut atoms = Vec::with_capacity(by.len());
    for path in by {
        let v = get_by_dotted_path(el, path);
        atoms.push(atom_from_value(v));
    }
    SortKey(atoms)
}

fn get_by_dotted_path<'a>(mut v: &'a Value, path: &str) -> Option<&'a Value> {
    for part in path.split('.') {
        if part.is_empty() {
            continue;
        }
        match v {
            Value::Object(map) => {
                v = map.get(part)?;
            }
            Value::Array(arr) => {
                let idx: usize = part.parse().ok()?;
                v = arr.get(idx)?;
            }
            _ => return None,
        }
    }
    Some(v)
}

fn atom_from_value(v: Option<&Value>) -> SortAtom {
    match v {
        None => SortAtom { kind: 0, repr: "".to_owned() },
        Some(Value::Null) => SortAtom { kind: 0, repr: "".to_owned() },
        Some(Value::Bool(b)) => SortAtom { kind: 1, repr: b.to_string() },
        Some(Value::Number(n)) => SortAtom { kind: 2, repr: n.to_string() },
        Some(Value::String(s)) => SortAtom { kind: 3, repr: s.clone() },
        Some(other) => {
            // For objects/arrays, use canonical JSON to get a stable string.
            let repr = serde_json_canonicalizer::to_string(other)
                .unwrap_or_else(|_| other.to_string());
            SortAtom { kind: 4, repr }
        }
    }
}
```

---

## `src/ops/canonicalize_json.rs`

```rust
use crate::config::{Canonicalization, Rule};
use crate::context::NormalizeContext;
use crate::error::Result;
use crate::ops::{args::Args, Op};

use serde_json::Value;

#[derive(Debug)]
pub struct CanonicalizeJsonOp;

impl Op for CanonicalizeJsonOp {
    fn name(&self) -> &'static str {
        "canonicalize_json"
    }

    fn apply(&self, _report: &mut Value, rule: &Rule, ctx: &mut NormalizeContext) -> Result<()> {
        let a = Args::new(&rule.id, self.name(), &rule.args);

        // Optional scheme override. Default to rfc8785 for compatibility with prior examples.
        let scheme = a.get_optional_str("scheme")?.unwrap_or("rfc8785");
        match scheme {
            "rfc8785" => ctx.output_overrides.canonicalization = Some(Canonicalization::Rfc8785),
            "none" => ctx.output_overrides.canonicalization = Some(Canonicalization::None),
            _ => {
                // Keep as no-op for unknown schemes; you can hard-error later if preferred.
            }
        }

        Ok(())
    }
}
```

---

## `tests/smoke.rs`

```rust
use burr_normalizer::config::NormalizerConfig;
use burr_normalizer::Normalizer;
use serde_json::json;

#[test]
fn smoke_uuid_and_ts_map() {
    let cfg_yaml = r#"
normalizer_version: v1
jsonpath_dialect: rfc9535
output:
  canonicalization: none
  pretty: true
rules:
  - id: uuid
    op: uuid_map
    select: "$..*"
    when: { type: string }
    args:
      placeholder: "<UUID#{n}>"
  - id: ts
    op: timestamp_map
    select: "$..*"
    when: { type: string }
    args:
      placeholder: "<TS#{n:04}>"
"#;

    let cfg: NormalizerConfig = serde_yaml::from_str(cfg_yaml).unwrap();
    let norm = Normalizer::new(cfg);

    let mut report = json!({
        "id": "00000000-0000-0000-0000-0000000000aa",
        "t": "2026-02-14T18:42:11.123456Z",
        "nested": { "x": "00000000-0000-0000-0000-0000000000bb" }
    });

    let _ctx = norm.normalize_in_place(&mut report).unwrap();

    assert_eq!(report["id"], "<UUID#1>");
    assert_eq!(report["nested"]["x"], "<UUID#2>");
    assert_eq!(report["t"], "<TS#0001>");
}
```

---

## Where this skeleton is “already correct” vs “intentionally loose”

* It’s **fully wired** to `serde_json_path` and uses `query_located().dedup().locations()` with `NormalizedPath::to_json_pointer()` for mutation via `Value::pointer_mut()`. ([Docs.rs][3])
* It’s **fully wired** to `serde_json_canonicalizer` for RFC 8785 output in `render_json`. ([Docs.rs][2])
* It implements the ops in a way that’s stable for goldens (dedup + pointer sorting, UUID/timestamp mappings are sorted uniques).

Things you may tighten later depending on your testing philosophy:

* Make `sort_array` strict by default.
* Decide whether `canonicalize_json` should be redundant (since you also have `output.canonicalization`) or should override output more explicitly.

If you want, I can also provide a companion **`burr-golden-runner`** crate skeleton that:

* loads scenarios,
* runs Python goldens (via subprocess) or reads committed goldens,
* runs Rust Burr,
* emits `run_report.json`,
* then normalizes via this crate and compares.

[1]: https://docs.rs/serde_json_path "serde_json_path - Rust"
[2]: https://docs.rs/serde_json_canonicalizer/latest/serde_json_canonicalizer/ "serde_json_canonicalizer - Rust"
[3]: https://docs.rs/serde_json_path/latest/serde_json_path/struct.LocatedNodeList.html "LocatedNodeList in serde_json_path - Rust"
