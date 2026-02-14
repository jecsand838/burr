Below are (A) **reference implementation sketches** for the three normalizer ops you called out—**`replace_prefixes_from_report`**, **`uuid_map`**, **`timestamp_map`**—in **both Python and Rust**, and (B) a **scenario-local override convention** (file naming + merge semantics + examples) that stays minimal but solves the “parallelism / nondeterministic ordering” and “scenario-specific cleanup” problems.

I’ll ground the choices in:

* **RFC 9535 JSONPath** as the selector dialect. ([RFC Editor][1])
* **RFC 8785 JCS** for canonical JSON serialization. ([RFC Editor][2])
* Burr’s **local tracking** file names and log entry timestamp fields (the parts you’ll normalize heavily). ([GitHub][3])

---

# A) Reference normalizer implementation sketches

## A.1 Core idea: select nodes + mutate safely

You want “implement once, reuse everywhere.” The cleanest path is:

1. **Select** nodes using RFC 9535 JSONPath,
2. Convert node “locations” into **stable paths/pointers**,
3. **Mutate** only leaf scalars (strings) in place.

### Why this is practical in both languages

**Python**: `jsonpath-rfc9535` returns `JSONPathNode` objects that include both a `location` and a normalized `path()`, and (critically) **assigning to `node.value` mutates the source document**. ([PyPI][4])

**Rust**: `serde_json_path` supports `query_located()` which yields locations as a `NormalizedPath`; `NormalizedPath::to_json_pointer()` produces a JSON Pointer string you can feed to `serde_json::Value::pointer_mut()` to mutate in place. ([Docs.rs][5])
Also `LocatedNodeList` gives you iterators over `locations()` (and `nodes()`) and has `dedup()` helpers, which is useful when selectors overlap. ([Docs.rs][6])

---

## A.2 Shared helper: “collect pointers first” (important)

For the three ops you asked about, you only mutate **string leaf nodes**, so pointers remain valid. Still, the most robust approach is:

* **Phase 1:** evaluate JSONPath selectors → produce a list of **stable locations** (normalized path / JSON pointer)
* **Phase 2:** read current string values at those locations; compute mappings (UUID/Timestamp)
* **Phase 3:** write mutated strings back

This avoids weirdness when you do multi-pass mapping or when you have overlapping selectors.

---

# A.3 Op: `replace_prefixes_from_report`

### What it must do

Given config entries like:

```yaml
prefixes:
  - source: "$.environment.paths.run_dir"
    placeholder: "<RUN_DIR>"
  - source: "$.environment.paths.repo_root"
    placeholder: "<REPO_ROOT>"
```

...resolve those `source` JSONPaths **against the report itself**, then replace occurrences of those path prefixes in selected string nodes.

This is essential because Burr tracking uses absolute filesystem paths (e.g., local tracking default dir `~/.burr`, and it writes `graph.json`, `metadata.json`, `log.jsonl`, `children.jsonl` under that directory). ([GitHub][3])

### Replacement policy (recommendation)

Support two modes (default `anywhere`):

* `prefix`: only replace when the string starts with the prefix
* `anywhere`: replace occurrences anywhere in the string (useful for exception messages like `"File /tmp/... line ..."`)

Also normalize slashes early (`\` → `/`), because Windows-like paths can appear in strings even on non-Windows hosts. (Your global rule already does this.)

### Python sketch (using `jsonpath-rfc9535`)

Key library facts we rely on: nodes include `.location`/`.path()` and assigning `node.value` mutates the source data. ([PyPI][4])

```python
import re
from dataclasses import dataclass
from typing import Any, Dict, List, Optional, Tuple, Iterable, Union

import jsonpath_rfc9535 as jsonpath

@dataclass(frozen=True)
class PrefixSpec:
    source: str        # JSONPath selector into the report
    placeholder: str   # e.g. "<RUN_DIR>"

def _slashify(s: str) -> str:
    return s.replace("\\", "/")

def _normalize_prefix(p: str) -> str:
    # normalize separators + drop trailing slash to avoid double slashes in outputs
    p = _slashify(p)
    while p.endswith("/") and len(p) > 1:
        p = p[:-1]
    return p

def resolve_prefixes(report: dict, specs: List[PrefixSpec]) -> List[Tuple[str, str]]:
    """Returns [(normalized_prefix, placeholder)], sorted longest-first."""
    resolved: List[Tuple[str, str]] = []
    for spec in specs:
        node = jsonpath.find_one(spec.source, report)
        if node is None or not isinstance(node.value, str) or not node.value:
            continue
        resolved.append((_normalize_prefix(node.value), spec.placeholder))
    # longest prefix first prevents partial replacements when prefixes nest
    resolved.sort(key=lambda x: len(x[0]), reverse=True)
    return resolved

def replace_prefixes_in_string(s: str, prefixes: List[Tuple[str, str]], mode: str) -> str:
    s2 = _slashify(s)
    for prefix, ph in prefixes:
        if mode == "prefix":
            if s2.startswith(prefix):
                s2 = ph + s2[len(prefix):]
        else:  # "anywhere"
            # replace occurrences even mid-string
            s2 = s2.replace(prefix, ph)
    return s2

def op_replace_prefixes_from_report(report: dict,
                                   select_exprs: List[str],
                                   prefix_specs: List[PrefixSpec],
                                   mode: str = "anywhere") -> None:
    prefixes = resolve_prefixes(report, prefix_specs)

    # Select nodes. Dedup by normalized path so overlapping selectors don't double-apply.
    seen = set()
    nodes = []
    for expr in select_exprs:
        for node in jsonpath.find(expr, report):
            p = node.path()  # normalized JSONPath string
            if p in seen:
                continue
            seen.add(p)
            nodes.append(node)

    for node in nodes:
        if isinstance(node.value, str):
            node.value = replace_prefixes_in_string(node.value, prefixes, mode)
```

### Rust sketch (using `serde_json_path`)

Key library facts: `query_located()` yields locations; `NormalizedPath::to_json_pointer()` works with `Value::pointer_mut()` for mutation. ([Docs.rs][7])

```rust
use regex::Regex;
use serde_json::Value;
use serde_json_path::JsonPath;

#[derive(Clone)]
pub struct PrefixSpec {
    pub source: String,      // JSONPath expression into the report
    pub placeholder: String, // "<RUN_DIR>"
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

fn resolve_prefixes(report: &Value, specs: &[PrefixSpec]) -> Vec<(String, String)> {
    let mut out = Vec::new();
    for spec in specs {
        let jp = JsonPath::parse(&spec.source)
            .unwrap_or_else(|e| panic!("bad JSONPath in prefix spec {}: {e}", spec.source));
        let nodes = jp.query(report);
        // best-effort: first node only
        if let Some(v) = nodes.first() {
            if let Some(s) = v.as_str() {
                if !s.is_empty() {
                    out.push((normalize_prefix(s), spec.placeholder.clone()));
                }
            }
        }
    }
    out.sort_by_key(|(p, _)| std::cmp::Reverse(p.len()));
    out
}

fn replace_prefixes_in_string(mut s: String, prefixes: &[(String, String)], mode: &str) -> String {
    s = slashify(&s);

    for (prefix, ph) in prefixes {
        if mode == "prefix" {
            if s.starts_with(prefix) {
                s = format!("{}{}", ph, &s[prefix.len()..]);
            }
        } else { // anywhere
            s = s.replace(prefix, ph);
        }
    }

    s
}

pub fn op_replace_prefixes_from_report(
    report: &mut Value,
    selectors: &[String],     // rule.select
    prefix_specs: &[PrefixSpec],
    mode: &str,
) {
    let prefixes = resolve_prefixes(report, prefix_specs);

    // Gather pointers first.
    let mut pointers: Vec<String> = Vec::new();
    for sel in selectors {
        let jp = JsonPath::parse(sel).unwrap();
        let located = jp.query_located(report).dedup(); // avoid dup nodes
        for loc in located.locations() {               //
            pointers.push(loc.to_json_pointer());      //
        }
    }

    pointers.sort();
    pointers.dedup();

    for ptr in pointers {
        if let Some(v) = report.pointer_mut(&ptr) {
            if let Some(s) = v.as_str() {
                let replaced = replace_prefixes_in_string(s.to_owned(), &prefixes, mode);
                *v = Value::String(replaced);
            }
        }
    }
}
```

---

# A.4 Op: `uuid_map`

### What it must do

Replace UUIDs found **inside any string** with stable placeholders:

* `aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee` → `<UUID#1>`, etc.

You want this because Burr uses IDs like `app_id`, span IDs, etc. (The tracker models have `span_id: str`, for example). ([GitHub][8])

### Stability strategy (strongly recommended)

Implement `assign: "sorted"` by default:

* Collect all UUID matches across selected nodes
* `sort()` unique values
* Assign placeholder numbers in that sorted order
  This makes placeholder numbering stable even if event ordering differs due to concurrency.

### Python sketch

```python
UUID_RE = re.compile(r"(?i)\b[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}\b")

def build_uuid_map(strings: List[str], placeholder_tpl: str = "<UUID#{n}>") -> Dict[str, str]:
    found = set()
    for s in strings:
        for m in UUID_RE.findall(s):
            found.add(m.lower())
    ordered = sorted(found)
    mapping = {}
    for i, u in enumerate(ordered, start=1):
        mapping[u] = placeholder_tpl.format(n=i)
    return mapping

def replace_uuids(s: str, mapping: Dict[str, str]) -> str:
    def repl(match: re.Match) -> str:
        key = match.group(0).lower()
        return mapping.get(key, match.group(0))
    return UUID_RE.sub(repl, s)

def op_uuid_map(report: dict, select_exprs: List[str], placeholder_tpl: str = "<UUID#{n}>") -> None:
    nodes = []
    seen = set()
    for expr in select_exprs:
        for node in jsonpath.find(expr, report):
            p = node.path()
            if p in seen:
                continue
            seen.add(p)
            nodes.append(node)

    strings = [node.value for node in nodes if isinstance(node.value, str)]
    mapping = build_uuid_map(strings, placeholder_tpl)

    for node in nodes:
        if isinstance(node.value, str):
            node.value = replace_uuids(node.value, mapping)
```

### Rust sketch

```rust
use regex::Regex;
use serde_json::Value;
use serde_json_path::JsonPath;
use std::collections::{BTreeMap, BTreeSet};

fn uuid_regex() -> Regex {
    Regex::new(r"(?i)\b[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}\b")
        .expect("valid UUID regex")
}

fn build_uuid_map(strings: &[String], placeholder_tpl: &str) -> BTreeMap<String, String> {
    let re = uuid_regex();
    let mut found: BTreeSet<String> = BTreeSet::new();
    for s in strings {
        for m in re.find_iter(s) {
            found.insert(m.as_str().to_ascii_lowercase());
        }
    }
    let mut map = BTreeMap::new();
    for (i, u) in found.into_iter().enumerate() {
        let n = i + 1;
        map.insert(u, placeholder_tpl.replace("{n}", &n.to_string()));
    }
    map
}

fn replace_uuids_in_string(s: &str, map: &BTreeMap<String, String>) -> String {
    let re = uuid_regex();
    re.replace_all(s, |caps: &regex::Captures| {
        let key = caps.get(0).unwrap().as_str().to_ascii_lowercase();
        map.get(&key).cloned().unwrap_or_else(|| caps.get(0).unwrap().as_str().to_string())
    }).into_owned()
}

pub fn op_uuid_map(report: &mut Value, selectors: &[String], placeholder_tpl: &str) {
    // 1) Gather pointers
    let mut pointers: Vec<String> = Vec::new();
    for sel in selectors {
        let jp = JsonPath::parse(sel).unwrap();
        let located = jp.query_located(report).dedup();
        for loc in located.locations() {
            pointers.push(loc.to_json_pointer()); // can mutate via pointer_mut
        }
    }
    pointers.sort();
    pointers.dedup();

    // 2) Collect current strings
    let mut strings: Vec<String> = Vec::new();
    for ptr in &pointers {
        if let Some(v) = report.pointer(ptr) {
            if let Some(s) = v.as_str() {
                strings.push(s.to_owned());
            }
        }
    }

    // 3) Build stable map and apply
    let map = build_uuid_map(&strings, placeholder_tpl);
    for ptr in pointers {
        if let Some(v) = report.pointer_mut(&ptr) {
            if let Some(s) = v.as_str() {
                *v = Value::String(replace_uuids_in_string(s, &map));
            }
        }
    }
}
```

---

# A.5 Op: `timestamp_map`

### What it must do

Replace timestamp-like strings with stable placeholders:

* `2026-02-14T18:42:11.123456` → `<TS#0001>`

You need this because Burr tracking records include many datetime fields, e.g. `BeginEntryModel.start_time`, `EndEntryModel.end_time`, span start/end times, attribute `time_logged`, and stream timestamps (`stream_init_time`, `first_item_time`, etc.). ([GitHub][8])
Also the tracking client uses `datetime.datetime.now()` in some places (e.g., child relationship `event_time`), which is inherently nondeterministic. ([GitHub][3])

### Stability strategy (recommended)

Use the same approach as UUIDs:

* Collect all timestamp matches
* Sort unique matches lexicographically (ISO timestamps sort chronologically when consistent)
* Assign `<TS#0001>`, `<TS#0002>`, … in that order

If you want even more stability, you can parse into `DateTime` and sort by parsed value; but lexicographic is typically good for ISO-ish strings.

### Python sketch

```python
TS_RE = re.compile(
    r"\b\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}(?:\.\d{1,9})?(?:Z|[+-]\d{2}:\d{2})?\b"
)

def build_ts_map(strings: List[str], placeholder_tpl: str = "<TS#{n:04}>") -> Dict[str, str]:
    found = set()
    for s in strings:
        for m in TS_RE.findall(s):
            found.add(m)
    ordered = sorted(found)
    mapping = {}
    for i, ts in enumerate(ordered, start=1):
        mapping[ts] = placeholder_tpl.format(n=i)
    return mapping

def replace_timestamps(s: str, mapping: Dict[str, str]) -> str:
    return TS_RE.sub(lambda m: mapping.get(m.group(0), m.group(0)), s)

def op_timestamp_map(report: dict, select_exprs: List[str], placeholder_tpl: str = "<TS#{n:04}>") -> None:
    nodes = []
    seen = set()
    for expr in select_exprs:
        for node in jsonpath.find(expr, report):
            p = node.path()
            if p in seen:
                continue
            seen.add(p)
            nodes.append(node)

    strings = [node.value for node in nodes if isinstance(node.value, str)]
    mapping = build_ts_map(strings, placeholder_tpl)

    for node in nodes:
        if isinstance(node.value, str):
            node.value = replace_timestamps(node.value, mapping)
```

### Rust sketch

```rust
use regex::Regex;
use serde_json::Value;
use serde_json_path::JsonPath;
use std::collections::{BTreeMap, BTreeSet};

fn ts_regex() -> Regex {
    Regex::new(r"\b\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}(?:\.\d{1,9})?(?:Z|[+-]\d{2}:\d{2})?\b")
        .expect("valid TS regex")
}

fn build_ts_map(strings: &[String], placeholder_tpl: &str) -> BTreeMap<String, String> {
    let re = ts_regex();
    let mut found: BTreeSet<String> = BTreeSet::new();
    for s in strings {
        for m in re.find_iter(s) {
            found.insert(m.as_str().to_owned());
        }
    }
    let mut map = BTreeMap::new();
    for (i, ts) in found.into_iter().enumerate() {
        let n = i + 1;
        let ph = placeholder_tpl.replace("{n:04}", &format!("{:04}", n))
                                .replace("{n}", &n.to_string());
        map.insert(ts, ph);
    }
    map
}

fn replace_ts_in_string(s: &str, map: &BTreeMap<String, String>) -> String {
    let re = ts_regex();
    re.replace_all(s, |caps: &regex::Captures| {
        let key = caps.get(0).unwrap().as_str();
        map.get(key).cloned().unwrap_or_else(|| key.to_owned())
    }).into_owned()
}

pub fn op_timestamp_map(report: &mut Value, selectors: &[String], placeholder_tpl: &str) {
    // pointers first
    let mut pointers: Vec<String> = Vec::new();
    for sel in selectors {
        let jp = JsonPath::parse(sel).unwrap();
        let located = jp.query_located(report).dedup();
        for loc in located.locations() {
            pointers.push(loc.to_json_pointer());
        }
    }
    pointers.sort();
    pointers.dedup();

    // collect strings
    let mut strings = Vec::new();
    for ptr in &pointers {
        if let Some(v) = report.pointer(ptr) {
            if let Some(s) = v.as_str() {
                strings.push(s.to_owned());
            }
        }
    }

    // stable map + apply
    let map = build_ts_map(&strings, placeholder_tpl);
    for ptr in pointers {
        if let Some(v) = report.pointer_mut(&ptr) {
            if let Some(s) = v.as_str() {
                *v = Value::String(replace_ts_in_string(s, &map));
            }
        }
    }
}
```

---

## A.6 Canonical JSON output (JCS) in both languages

You already proposed RFC 8785 JCS as the terminal step, which is ideal for deterministic goldens. ([RFC Editor][2])

### Python

Use a JCS implementation like `rfc8785` or `jcs` (both explicitly target RFC 8785). ([PyPI][9])

**Sketch:**

```python
import rfc8785  # or import jcs

def canonicalize_json_bytes(obj: Any) -> bytes:
    # rfc8785.py
    return rfc8785.canonicalize(obj)

def canonicalize_json_string(obj: Any) -> str:
    return canonicalize_json_bytes(obj).decode("utf-8")
```

### Rust

Use an RFC 8785 compatible canonicalizer, e.g. `serde_json_canonicalizer` (drop-in to_string/to_vec), which explicitly claims RFC 8785 compatibility. ([Docs.rs][10])

**Sketch:**

```rust
use serde_json_canonicalizer::to_string; // RFC8785 compatible

pub fn canonicalize(value: &serde_json::Value) -> String {
    to_string(value).expect("canonicalize")
}
```

---

# B) Scenario-local override convention + merge semantics

## B.1 File naming & lookup convention

Keep it dead simple and “automatic”:

* Scenario spec:
  `spec/scenarios/<scenario_id>.json`

* Optional scenario normalizer overlay:
  `spec/scenarios/<scenario_id>.normalize.yaml`

Runner behavior:

1. Load base normalizer: `spec/normalizer/v1.yaml`
2. If `<scenario_id>.normalize.yaml` exists, load overlay and merge.

This gives you per-scenario tweaks without bloating the scenario spec schema.

---

## B.2 Minimal override file format

### `spec/scenarios/<scenario_id>.normalize.yaml`

```yaml
normalizer_override_version: v1

# Remove base rules by id (useful when a scenario needs different ordering)
disable_rules:
  - sort_children

# Replace rules by id (exact match required)
replace_rules:
  - id: timestamp_placeholders
    op: timestamp_map
    select: "$..*"
    when: { type: string }
    args:
      placeholder: "<TS>"          # override to constant (optional style)

# Add new rules (with optional placement anchor)
add_rules:
  - rule:
      id: sort_tracking_log
      op: sort_array
      select: "$.runs[*].artifacts.tracking.files.log"
      args:
        by: ["type", "sequence_id", "action_sequence_id", "span_id"]
    position:
      before: uuid_placeholders

# Optional output tweaks
output_override:
  pretty: true
```

### Why this is enough

* `disable_rules` handles “this scenario needs a different ordering strategy”
* `replace_rules` handles “same op but different args/selectors”
* `add_rules` handles “scenario-specific sort/redact rules”
* `position` handles the “sort-before-mapping” stability requirement without forcing you to disable/re-add entire rule sequences

---

## B.3 Merge algorithm (exact, deterministic)

Given:

* Base config: `NormalizerConfig { output, rules[] }`
* Override: `NormalizerOverride { disable_rules, replace_rules, add_rules, output_override }`

Merge:

1. Start with base `rules` list in order.
2. Remove any rule whose `id` is in `disable_rules`.
3. For each entry in `replace_rules`:

    * Find a rule with the same `id` in the current list.
    * Replace it *in place* (preserving its position).
    * If not found: **error** (catches typos early).
4. For each `add_rules` item:

    * If `position.before` is set: insert directly before that rule id (error if not found).
    * Else if `position.after` is set: insert directly after that rule id (error if not found).
    * Else append to end.
    * If inserted rule id already exists: **error** (prevents accidental duplicates).
5. If `output_override` provided: shallow-merge into base output (override specified keys only).

This ensures the resulting rule order is always deterministic.

---

## B.4 Two concrete override examples

### Example 1: parallelism-heavy scenario (sort logs deterministically)

This scenario likely produces `tracking.files.log` where event ordering can vary (especially spans/attributes). You sort it before UUID/timestamp mapping to stabilize placeholder numbering.

```yaml
normalizer_override_version: v1

add_rules:
  - rule:
      id: sort_tracking_log
      op: sort_array
      select: "$.runs[*].artifacts.tracking.files.log"
      args:
        by: ["type", "sequence_id", "action_sequence_id", "span_id"]
    position:
      before: uuid_placeholders
```

This is motivated by the fact Burr log entries include spans (`span_id`), step entries (`sequence_id`), etc. ([GitHub][8])

### Example 2: a scenario with OS-specific error message text

Maybe a test deliberately triggers a filesystem error; macOS/Linux/Windows error strings differ. Redact that one field only:

```yaml
normalizer_override_version: v1

add_rules:
  - rule:
      id: redact_os_error_message
      op: redact
      select: "$.runs[*].calls[*].result.exception.message"
      args:
        replace: "<OS_ERROR>"
```

---

## B.5 Implementing override merge (Rust sketch)

```rust
#[derive(serde::Deserialize)]
struct OverrideConfig {
    normalizer_override_version: String,
    disable_rules: Option<Vec<String>>,
    replace_rules: Option<Vec<Rule>>,
    add_rules: Option<Vec<AddRule>>,
    output_override: Option<OutputConfig>,
}

#[derive(serde::Deserialize)]
struct AddRule {
    rule: Rule,
    position: Option<Position>,
}
#[derive(serde::Deserialize)]
struct Position {
    before: Option<String>,
    after: Option<String>,
}

fn merge(base: NormalizerConfig, ov: OverrideConfig) -> NormalizerConfig {
    let mut rules = base.rules;

    // 1) disable
    if let Some(disable) = ov.disable_rules {
        rules.retain(|r| !disable.contains(&r.id));
    }

    // 2) replace
    if let Some(replace) = ov.replace_rules {
        for rr in replace {
            let idx = rules.iter().position(|r| r.id == rr.id)
                .unwrap_or_else(|| panic!("replace_rules: unknown id {}", rr.id));
            rules[idx] = rr;
        }
    }

    // 3) add
    if let Some(adds) = ov.add_rules {
        for add in adds {
            if rules.iter().any(|r| r.id == add.rule.id) {
                panic!("add_rules: duplicate id {}", add.rule.id);
            }
            match add.position {
                Some(Position { before: Some(b), after: _ }) => {
                    let idx = rules.iter().position(|r| r.id == b)
                        .unwrap_or_else(|| panic!("add_rules.before: unknown id {}", b));
                    rules.insert(idx, add.rule);
                }
                Some(Position { before: _, after: Some(a) }) => {
                    let idx = rules.iter().position(|r| r.id == a)
                        .unwrap_or_else(|| panic!("add_rules.after: unknown id {}", a));
                    rules.insert(idx + 1, add.rule);
                }
                _ => rules.push(add.rule),
            }
        }
    }

    // 4) output override
    let mut out = base;
    out.rules = rules;
    if let Some(oo) = ov.output_override {
        out.output.canonicalization = oo.canonicalization.unwrap_or(out.output.canonicalization);
        out.output.pretty = oo.pretty.unwrap_or(out.output.pretty);
    }
    out
}
```

---

## B.6 One extra tip: make selector dialect explicit

Keep `jsonpath_dialect: rfc9535` in the base config and require it in overrides too. It prevents accidental use of non-standard JSONPath quirks and keeps selectors portable. ([RFC Editor][1])

---

[1]: https://www.rfc-editor.org/rfc/rfc9535 "RFC 9535: JSONPath: Query Expressions for JSON"
[2]: https://www.rfc-editor.org/rfc/rfc8785 "RFC 8785: JSON Canonicalization Scheme (JCS)"
[3]: https://raw.githubusercontent.com/apache/burr/main/burr/tracking/client.py "raw.githubusercontent.com"
[4]: https://pypi.org/project/jsonpath-rfc9535/ "jsonpath-rfc9535 · PyPI"
[5]: https://docs.rs/serde_json_path "serde_json_path - Rust"
[6]: https://docs.rs/serde_json_path/latest/serde_json_path/struct.LocatedNodeList.html "LocatedNodeList in serde_json_path - Rust"
[7]: https://docs.rs/serde_json_path/latest/serde_json_path/struct.NormalizedPath.html "NormalizedPath in serde_json_path - Rust"
[8]: https://raw.githubusercontent.com/apache/burr/main/burr/tracking/common/models.py "raw.githubusercontent.com"
[9]: https://pypi.org/project/rfc8785/ "rfc8785"
[10]: https://docs.rs/serde_json_canonicalizer/latest/serde_json_canonicalizer/ "serde_json_canonicalizer - Rust"
