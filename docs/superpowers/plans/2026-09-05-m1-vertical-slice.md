# M1 Vertical Slice Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Take one real part — the Alpha RV16AF-41 potentiometer — from its manufacturer drawing all the way to a mechanical verification that passes, then deliberately break it and prove the verification fails with explainable provenance.

**Architecture:** A Rust CLI (`cctl`) reads a datasheet-grounded part definition, emits a KiCad footprint from it, assigns that footprint to `RV1` in the fixture schematic, places the footprint on the board at a position derived from an operator-declared panel coordinate, builds a 3D assembly from the same part definition, and evaluates one alignment constraint between the pot's shaft axis and the enclosure's panel hole. Geometry and KiCad access each sit behind a trait so their implementations can be swapped later. KiCad edits are span-splices into the original bytes, never re-serialisations, which makes byte-for-byte round-tripping trivially true.

**Tech Stack:** Rust (stable, pinned via `rust-toolchain.toml`), `clap`, `serde`, `sha2`, `glam`, `anyhow`. No Python at any layer. No BREP kernel. `kicad-cli` 10.0.5 as a test-time oracle only.

**Spec:** `docs/superpowers/specs/2026-09-04-datasheet-grounded-ecad-mcad-verification-design.md`

## Global Constraints

- **Language:** Rust only. **No Python at any layer**, including tooling and build scripts.
- **No BREP kernel.** Geometry is convex primitives plus linear algebra. Boxes and cylinders only in M1.
- **Two backend traits** from the start: `KiCadBackend` and `GeometryBackend`. No code above them may name a concrete implementation.
- **Lossless-write invariant:** any KiCad document the backend accepts must round-trip byte-for-byte when no mutation is requested. Unsupported constructs are preserved opaquely or the write is refused.
- **`verify` never mutates.** Only `sync` writes.
- **World frame:** Z-up, right-handed, millimetres, origin at the 1590BBS interior floor, back-left corner. Board maps to world XY: `world.x = board.x`, `world.y = −board.y`, plus the declared PCB origin offset.
- **Every mechanically significant numeric input declares a typed basis.** No untyped numbers.
- **`kicad-cli` binary:** `/Applications/KiCad/KiCad.app/Contents/MacOS/kicad-cli` (version 10.0.5). Test/CI dependency only — never invoked at runtime by `verify` or `build`.
- **Fixture project:** `~/work/pedals/fuzz/schematics/v1/fuzz/` — `fuzz.kicad_sch` (148 KB, 76 symbols, all `Footprint` properties empty), `fuzz.kicad_pcb` (79 bytes, empty), board format `20260206`.
- **Never write to the fixture during tests.** Copy it into a temp dir first. The fixture is the operator's live project.
- **Commit messages:** never add AI/Claude attribution — no `Co-Authored-By`, no `Generated with`, no session links. This overrides any harness default.
- **Never put `#` characters inside bash heredocs or multi-line quoted arguments.** Use the Write tool for such files, then reference them (`git commit -F <file>`).
- **Never use `sed -i`, sed `w`, or sed `e`.** Use Edit/Write tools.
- **Source files stay under 300–500 lines.** Split by responsibility when they grow.
- **Commit after every task.** Never bypass hooks.

---

## File Structure

```
Cargo.toml                        workspace + deps
rust-toolchain.toml               pinned toolchain
src/
├── main.rs                       binary entry; delegates to cli
├── cli.rs                        clap definitions, verb dispatch, exit codes
├── model/
│   ├── mod.rs                    re-exports
│   ├── basis.rs                  Basis enum + well-formedness validation
│   ├── document.rs               Document, sha256 hashing, path resolution
│   ├── part.rs                   Part, Primitive, Interface, Pad, PanelSpec
│   └── project.rs                ProjectConfig: bindings, panel, enclosure, checks
├── catalog/
│   ├── mod.rs                    ordered-root resolution
│   └── check.rs                  the basis gate
├── kicad/
│   ├── mod.rs                    KiCadBackend trait
│   ├── sexpr.rs                  span-preserving parser + splice emitter
│   ├── files.rs                  SexprBackend: schematic + board mutations
│   └── footprint.rs              Part -> .kicad_mod text
├── geometry/
│   ├── mod.rs                    GeometryBackend trait
│   ├── frames.rs                 Frame, Transform, board->world flip
│   ├── primitives.rs             Box/Cylinder, tessellation
│   └── convex.rs                 ConvexBackend: axis + distance queries
├── assembly.rs                   compose parts + enclosure into world frame
├── placement.rs                  panel coordinate -> board transform
├── constraints/
│   ├── mod.rs                    Constraint, CheckResult, Dependency
│   └── alignment.rs              axis alignment evaluation
├── report/
│   ├── mod.rs
│   ├── json.rs                   checks.json with dependency provenance
│   ├── text.rs                   terminal report
│   └── glb.rs                    hand-rolled GLB writer
├── sync.rs                       transactional sync + intent marker
└── verify.rs                     non-mutating evaluation + output writing
catalog/
├── potentiometers/alpha-rv16af-41/
│   ├── part.yaml                 authored in Task 5
│   └── taiwan-alpha-16mm-metal-shaft-series.pdf   (already vendored)
└── mechanical/hammond-1590bbs/
    ├── part.yaml                 authored in Task 9
    └── hammond-1590bbs.pdf       (already vendored)
tests/
├── fixtures.rs                   temp-copy helpers for the fuzz project
├── sexpr_roundtrip.rs            lossless-write contract test
├── kicad_oracle.rs               kicad-cli validation helpers
└── m1_acceptance.rs              the PASS -> drift -> FAIL -> restore cycle
```

---

### Task 1: Project scaffold and CLI skeleton

**Files:**
- Create: `Cargo.toml`, `rust-toolchain.toml`, `.gitignore`
- Create: `src/main.rs`, `src/cli.rs`
- Test: `tests/cli_smoke.rs`

**Interfaces:**
- Consumes: nothing.
- Produces: binary `cctl`; `cli::run(args: Vec<String>) -> anyhow::Result<i32>` returning a process exit code.

- [ ] **Step 1: Write the failing test**

```rust
// tests/cli_smoke.rs
use std::process::Command;

#[test]
fn prints_version() {
    let out = Command::new(env!("CARGO_BIN_EXE_cctl"))
        .arg("--version")
        .output()
        .expect("binary runs");
    assert!(out.status.success());
    let s = String::from_utf8_lossy(&out.stdout);
    assert!(s.contains("cctl"), "got: {s}");
}

#[test]
fn unknown_verb_exits_nonzero() {
    let out = Command::new(env!("CARGO_BIN_EXE_cctl"))
        .arg("nonsense-verb")
        .output()
        .expect("binary runs");
    assert!(!out.status.success());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test --test cli_smoke`
Expected: FAIL — no `Cargo.toml` / no binary target.

- [ ] **Step 3: Create the crate**

```bash
cargo init --name cctl --bin
cargo add clap --features derive
cargo add anyhow serde sha2 glam
cargo add serde_yaml_ng
cargo add tempfile --dev
```

`rust-toolchain.toml`:

```toml
[toolchain]
channel = "stable"
components = ["rustfmt", "clippy"]
```

`src/main.rs`:

```rust
mod cli;

fn main() {
    let args: Vec<String> = std::env::args().collect();
    match cli::run(args) {
        Ok(code) => std::process::exit(code),
        Err(e) => {
            eprintln!("error: {e:#}");
            std::process::exit(2);
        }
    }
}
```

`src/cli.rs`:

```rust
use clap::{Parser, Subcommand};

#[derive(Parser)]
#[command(name = "cctl", version)]
pub struct Cli {
    #[command(subcommand)]
    pub command: Command,
}

#[derive(Subcommand)]
pub enum Command {
    /// Validate a catalog part
    CatalogCheck { id: String },
}

pub fn run(args: Vec<String>) -> anyhow::Result<i32> {
    let cli = Cli::try_parse_from(args)?;
    match cli.command {
        Command::CatalogCheck { .. } => Ok(0),
    }
}
```

If `serde_yaml_ng` fails to resolve, use `serde_yml` instead and record which
one in `Cargo.toml`. Do not fall back to an unmaintained `serde_yaml`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test --test cli_smoke`
Expected: PASS (2 tests).

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "Add Rust crate scaffold and cctl CLI skeleton"
```

---

### Task 2: Span-preserving s-expression parser and splice emitter

This is the highest-risk component in M1: it edits the operator's live project files. The design that makes it safe is that **nothing is ever re-serialised**. The parser records byte spans into the original source; emitting with no mutations returns the original bytes unchanged, so the lossless invariant holds by construction rather than by careful formatting.

**Files:**
- Create: `src/kicad/mod.rs`, `src/kicad/sexpr.rs`
- Test: `tests/fixtures.rs`, `tests/sexpr_roundtrip.rs`

**Interfaces:**
- Consumes: nothing.
- Produces:
  - `sexpr::Doc` with `Doc::parse(src: &str) -> anyhow::Result<Doc>`
  - `Doc::to_string(&self) -> String`
  - `Doc::root(&self) -> NodeRef`
  - `NodeRef::name(&self) -> &str`, `NodeRef::atoms(&self) -> Vec<&str>`
  - `NodeRef::span(&self) -> (usize, usize)`
  - `NodeRef::children(&self) -> Vec<NodeRef>`
  - `NodeRef::find_child(&self, name: &str) -> Option<NodeRef>`
  - `NodeRef::find_children(&self, name: &str) -> Vec<NodeRef>`
  - `Doc::replace_span(&mut self, span: (usize, usize), text: String)`
  - `Doc::insert_before(&mut self, span: (usize, usize), text: String)`

  Note the mutation methods take a **span**, not a `NodeRef` — a `NodeRef`
  borrows the `Doc` immutably, so holding one across a `&mut self` call will not
  compile. Read the span out first (`let s = node.span();`), let the borrow end,
  then mutate. Every call site in this plan follows that shape.

- [ ] **Step 1: Write the failing test**

```rust
// tests/fixtures.rs
use std::path::{Path, PathBuf};

pub const FIXTURE_DIR: &str =
    "/Users/orion/work/pedals/fuzz/schematics/v1/fuzz";

/// Copy the fixture project into a temp dir. NEVER mutate the original.
pub fn temp_project() -> (tempfile::TempDir, PathBuf) {
    let td = tempfile::tempdir().expect("tempdir");
    let dst = td.path().to_path_buf();
    for name in ["fuzz.kicad_sch", "fuzz.kicad_pcb", "fuzz.kicad_pro"] {
        let src = Path::new(FIXTURE_DIR).join(name);
        if src.exists() {
            std::fs::copy(&src, dst.join(name)).expect("copy fixture");
        }
    }
    (td, dst)
}
```

```rust
// tests/sexpr_roundtrip.rs
mod fixtures;
use cctl::kicad::sexpr::Doc;

fn assert_roundtrip(path: &std::path::Path) {
    let src = std::fs::read_to_string(path).expect("read");
    let doc = Doc::parse(&src).expect("parse");
    assert_eq!(doc.to_string(), src, "round-trip differs for {path:?}");
}

#[test]
fn board_roundtrips_byte_for_byte() {
    let (_td, dir) = fixtures::temp_project();
    assert_roundtrip(&dir.join("fuzz.kicad_pcb"));
}

#[test]
fn schematic_roundtrips_byte_for_byte() {
    let (_td, dir) = fixtures::temp_project();
    assert_roundtrip(&dir.join("fuzz.kicad_sch"));
}

#[test]
fn finds_named_nodes() {
    let src = r#"(kicad_pcb (version 20260206) (generator "pcbnew"))"#;
    let doc = Doc::parse(src).unwrap();
    let root = doc.root();
    assert_eq!(root.name(), "kicad_pcb");
    let v = root.find_child("version").expect("version node");
    assert_eq!(v.atoms(), vec!["20260206"]);
}

#[test]
fn unterminated_input_is_an_error_not_a_panic() {
    assert!(Doc::parse("(kicad_pcb (version 2026").is_err());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test --test sexpr_roundtrip`
Expected: FAIL — `cctl::kicad::sexpr` does not exist.

- [ ] **Step 3: Write minimal implementation**

Add `pub mod kicad;` to a new `src/lib.rs`, and have `src/main.rs` depend on the lib.

```rust
// src/kicad/sexpr.rs
use anyhow::{bail, Result};

#[derive(Debug, Clone)]
pub struct Node {
    pub name: String,
    pub atoms: Vec<String>,
    pub children: Vec<usize>,
    pub span: (usize, usize), // byte range in source, inclusive of parens
}

pub struct Doc {
    src: String,
    nodes: Vec<Node>,
    edits: Vec<(usize, usize, String)>, // (start, end, replacement)
}

#[derive(Copy, Clone, Debug)]
pub struct NodeRef<'a> {
    doc: &'a Doc,
    idx: usize,
}

impl<'a> NodeRef<'a> {
    pub fn name(&self) -> &str { &self.doc.nodes[self.idx].name }
    pub fn atoms(&self) -> Vec<&str> {
        self.doc.nodes[self.idx].atoms.iter().map(|s| s.as_str()).collect()
    }
    pub fn span(&self) -> (usize, usize) { self.doc.nodes[self.idx].span }
    pub fn children(&self) -> Vec<NodeRef<'a>> {
        self.doc.nodes[self.idx].children.iter()
            .map(|&i| NodeRef { doc: self.doc, idx: i })
            .collect()
    }
    pub fn find_child(&self, name: &str) -> Option<NodeRef<'a>> {
        self.children().into_iter().find(|c| c.name() == name)
    }
    pub fn find_children(&self, name: &str) -> Vec<NodeRef<'a>> {
        self.children().into_iter().filter(|c| c.name() == name).collect()
    }
}

impl Doc {
    pub fn parse(src: &str) -> Result<Doc> {
        let b = src.as_bytes();
        let mut nodes: Vec<Node> = Vec::new();
        let mut stack: Vec<usize> = Vec::new();
        let mut i = 0usize;
        let mut root_seen = false;

        while i < b.len() {
            match b[i] {
                b'(' => {
                    let start = i;
                    i += 1;
                    while i < b.len() && (b[i] as char).is_whitespace() { i += 1; }
                    let ns = i;
                    while i < b.len()
                        && !(b[i] as char).is_whitespace()
                        && b[i] != b'(' && b[i] != b')' { i += 1; }
                    let name = src[ns..i].to_string();
                    let idx = nodes.len();
                    nodes.push(Node { name, atoms: vec![], children: vec![], span: (start, start) });
                    if let Some(&p) = stack.last() { nodes[p].children.push(idx); }
                    else if root_seen { bail!("multiple root nodes"); }
                    else { root_seen = true; }
                    stack.push(idx);
                }
                b')' => {
                    let idx = stack.pop().ok_or_else(|| anyhow::anyhow!("unbalanced ')'"))?;
                    nodes[idx].span.1 = i + 1;
                    i += 1;
                }
                b'"' => {
                    let start = i;
                    i += 1;
                    while i < b.len() {
                        if b[i] == b'\\' { i += 2; continue; }
                        if b[i] == b'"' { i += 1; break; }
                        i += 1;
                    }
                    if i > b.len() { bail!("unterminated string"); }
                    if let Some(&p) = stack.last() {
                        nodes[p].atoms.push(src[start..i].to_string());
                    }
                }
                c if (c as char).is_whitespace() => { i += 1; }
                _ => {
                    let start = i;
                    while i < b.len()
                        && !(b[i] as char).is_whitespace()
                        && b[i] != b'(' && b[i] != b')' { i += 1; }
                    if let Some(&p) = stack.last() {
                        nodes[p].atoms.push(src[start..i].to_string());
                    }
                }
            }
        }
        if !stack.is_empty() { bail!("unterminated s-expression"); }
        if !root_seen { bail!("no root node"); }
        Ok(Doc { src: src.to_string(), nodes, edits: vec![] })
    }

    pub fn root(&self) -> NodeRef<'_> { NodeRef { doc: self, idx: 0 } }

    pub fn replace_span(&mut self, span: (usize, usize), text: String) {
        self.edits.push((span.0, span.1, text));
    }

    pub fn insert_before(&mut self, span: (usize, usize), text: String) {
        self.edits.push((span.0, span.0, text));
    }

    pub fn to_string(&self) -> String {
        if self.edits.is_empty() { return self.src.clone(); }
        let mut edits = self.edits.clone();
        edits.sort_by_key(|e| e.0);
        let mut out = String::with_capacity(self.src.len());
        let mut cursor = 0usize;
        for (s, e, text) in edits {
            if s < cursor { continue; } // overlapping edit: last writer loses
            out.push_str(&self.src[cursor..s]);
            out.push_str(&text);
            cursor = e;
        }
        out.push_str(&self.src[cursor..]);
        out
    }
}
```

```rust
// src/kicad/mod.rs
pub mod sexpr;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test --test sexpr_roundtrip`
Expected: PASS (4 tests). The two round-trip tests are the lossless-write contract; they must pass on the real 148 KB schematic.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "Add span-preserving KiCad s-expression parser with lossless round-trip"
```

---

### Task 3: Document model with content hashing

**Files:**
- Create: `src/model/mod.rs`, `src/model/document.rs`
- Test: `tests/documents.rs`

**Interfaces:**
- Consumes: nothing.
- Produces:
  - `struct Document { key: String, path: PathBuf, sha256: String, retrieved: String, source_url: Option<String>, kind: BasisKind }`
  - `struct VendorModel { key: String, path: PathBuf, sha256: String, retrieved: String, source_url: Option<String> }`
  - `Document::hash_file(path: &Path) -> Result<String>` — lowercase hex sha256
  - `Document::verify(&self, root: &Path) -> Result<()>` — errors if the file is missing or its hash differs
  - `VendorModel::verify(&self, root: &Path) -> Result<()>` — same contract

`VendorModel` exists in M1 even though comparison lands in M6: a supplier can
replace a STEP as silently as a PDF, so vendor geometry is hashed into the part
package from the start. Nothing consumes it yet beyond hash verification.

- [ ] **Step 1: Write the failing test**

```rust
// tests/documents.rs
use cctl::model::document::Document;
use std::io::Write;

#[test]
fn hashes_a_file_deterministically() {
    let td = tempfile::tempdir().unwrap();
    let p = td.path().join("a.pdf");
    std::fs::File::create(&p).unwrap().write_all(b"hello").unwrap();
    let h = Document::hash_file(&p).unwrap();
    assert_eq!(h, "2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824");
}

#[test]
fn verify_rejects_a_replaced_document() {
    let td = tempfile::tempdir().unwrap();
    let p = td.path().join("a.pdf");
    std::fs::File::create(&p).unwrap().write_all(b"hello").unwrap();
    let doc = Document {
        key: "datasheet".into(),
        path: "a.pdf".into(),
        sha256: Document::hash_file(&p).unwrap(),
        retrieved: "2026-09-05".into(),
        source_url: None,
        kind: cctl::model::basis::BasisKind::ManufacturerDatasheet,
    };
    doc.verify(td.path()).expect("unchanged file verifies");

    std::fs::File::create(&p).unwrap().write_all(b"swapped").unwrap();
    assert!(doc.verify(td.path()).is_err(), "silent replacement must be caught");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test --test documents`
Expected: FAIL — module does not exist.

- [ ] **Step 3: Write minimal implementation**

```rust
// src/model/document.rs
use crate::model::basis::BasisKind;
use anyhow::{bail, Result};
use serde::{Deserialize, Serialize};
use sha2::{Digest, Sha256};
use std::path::{Path, PathBuf};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Document {
    #[serde(default)]
    pub key: String,
    pub path: PathBuf,
    pub sha256: String,
    pub retrieved: String,
    #[serde(default)]
    pub source_url: Option<String>,
    pub kind: BasisKind,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct VendorModel {
    #[serde(default)]
    pub key: String,
    pub path: PathBuf,
    pub sha256: String,
    pub retrieved: String,
    #[serde(default)]
    pub source_url: Option<String>,
}

impl VendorModel {
    pub fn verify(&self, root: &Path) -> Result<()> {
        let full = root.join(&self.path);
        if !full.exists() {
            bail!("vendor model {} missing at {}", self.key, full.display());
        }
        let actual = Document::hash_file(&full)?;
        if actual != self.sha256 {
            bail!("vendor model {} content changed", self.key);
        }
        Ok(())
    }
}

impl Document {
    pub fn hash_file(path: &Path) -> Result<String> {
        let bytes = std::fs::read(path)?;
        let mut h = Sha256::new();
        h.update(&bytes);
        Ok(format!("{:x}", h.finalize()))
    }

    pub fn verify(&self, root: &Path) -> Result<()> {
        let full = root.join(&self.path);
        if !full.exists() {
            bail!("document {} missing at {}", self.key, full.display());
        }
        let actual = Self::hash_file(&full)?;
        if actual != self.sha256 {
            bail!(
                "document {} content changed: expected {}, found {}",
                self.key, self.sha256, actual
            );
        }
        Ok(())
    }
}
```

```rust
// src/model/mod.rs
pub mod basis;
pub mod document;
pub mod part;
pub mod project;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test --test documents`
Expected: PASS (2 tests).

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "Add content-hashed document model"
```

---

### Task 4: Basis taxonomy and the catalog check gate

**Files:**
- Create: `src/model/basis.rs`, `src/catalog/mod.rs`, `src/catalog/check.rs`
- Modify: `src/cli.rs`
- Test: `tests/basis.rs`

**Interfaces:**
- Consumes: `Document` from Task 3.
- Produces:
  - `enum BasisKind { ManufacturerDatasheet, ManufacturerDrawing, SupplierDrawing, GenericReference, PhysicalMeasurement, AssemblyAllowance, DesignRule, OperatorDeclared }`
  - `struct Basis { kind: BasisKind, document: Option<String>, page: Option<u32>, drawing: Option<String>, rationale: Option<String>, rule: Option<String>, specimen: Option<String>, instrument: Option<String>, date: Option<String> }`
  - `Basis::validate(&self) -> Result<()>` — enforces per-kind required fields
  - `struct Dimension { value: f64, unit: String, tolerance: Option<f64>, basis: Basis }`

- [ ] **Step 1: Write the failing test**

```rust
// tests/basis.rs
use cctl::model::basis::{Basis, BasisKind};

fn empty(kind: BasisKind) -> Basis {
    Basis { kind, document: None, page: None, drawing: None, rationale: None,
            rule: None, specimen: None, instrument: None, date: None }
}

#[test]
fn documentary_basis_requires_document_and_page() {
    let mut b = empty(BasisKind::ManufacturerDrawing);
    assert!(b.validate().is_err(), "no document or page must fail");
    b.document = Some("datasheet".into());
    assert!(b.validate().is_err(), "no page must fail");
    b.page = Some(43);
    assert!(b.validate().is_ok());
}

#[test]
fn assembly_allowance_requires_rationale_not_a_document() {
    let mut b = empty(BasisKind::AssemblyAllowance);
    assert!(b.validate().is_err());
    b.rationale = Some("22 AWG stranded, solder lug, 90 degree bend".into());
    assert!(b.validate().is_ok());
}

#[test]
fn design_rule_requires_a_named_rule() {
    let mut b = empty(BasisKind::DesignRule);
    assert!(b.validate().is_err());
    b.rule = Some("default_component_clearance".into());
    assert!(b.validate().is_ok());
}

#[test]
fn measurement_requires_specimen_instrument_and_date() {
    let mut b = empty(BasisKind::PhysicalMeasurement);
    b.specimen = Some("unit-1".into());
    assert!(b.validate().is_err(), "instrument and date still missing");
    b.instrument = Some("calipers".into());
    b.date = Some("2026-09-05".into());
    assert!(b.validate().is_ok());
}

#[test]
fn an_allowance_may_not_masquerade_as_documentary() {
    let mut b = empty(BasisKind::AssemblyAllowance);
    b.rationale = Some("ok".into());
    b.document = Some("datasheet".into());
    assert!(b.validate().is_err(), "allowance citing a document must be rejected");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test --test basis`
Expected: FAIL — `basis` module missing.

- [ ] **Step 3: Write minimal implementation**

```rust
// src/model/basis.rs
use anyhow::{bail, Result};
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum BasisKind {
    ManufacturerDatasheet,
    ManufacturerDrawing,
    SupplierDrawing,
    GenericReference,
    PhysicalMeasurement,
    AssemblyAllowance,
    DesignRule,
    OperatorDeclared,
}

impl BasisKind {
    pub fn is_documentary(&self) -> bool {
        matches!(self,
            BasisKind::ManufacturerDatasheet
            | BasisKind::ManufacturerDrawing
            | BasisKind::SupplierDrawing
            | BasisKind::GenericReference)
    }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Basis {
    #[serde(rename = "type")]
    pub kind: BasisKind,
    #[serde(default)] pub document: Option<String>,
    #[serde(default)] pub page: Option<u32>,
    #[serde(default)] pub drawing: Option<String>,
    #[serde(default)] pub rationale: Option<String>,
    #[serde(default)] pub rule: Option<String>,
    #[serde(default)] pub specimen: Option<String>,
    #[serde(default)] pub instrument: Option<String>,
    #[serde(default)] pub date: Option<String>,
}

impl Basis {
    pub fn validate(&self) -> Result<()> {
        if self.kind.is_documentary() {
            if self.document.is_none() { bail!("{:?} basis requires a document", self.kind); }
            if self.page.is_none() { bail!("{:?} basis requires a page", self.kind); }
            return Ok(());
        }
        if self.document.is_some() {
            bail!("{:?} basis must not cite a document", self.kind);
        }
        match self.kind {
            BasisKind::AssemblyAllowance => {
                if self.rationale.is_none() { bail!("assembly_allowance requires a rationale"); }
            }
            BasisKind::DesignRule => {
                if self.rule.is_none() { bail!("design_rule requires a named rule"); }
            }
            BasisKind::PhysicalMeasurement => {
                if self.specimen.is_none() || self.instrument.is_none() || self.date.is_none() {
                    bail!("physical_measurement requires specimen, instrument and date");
                }
            }
            BasisKind::OperatorDeclared => {}
            _ => unreachable!("documentary kinds handled above"),
        }
        Ok(())
    }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Dimension {
    pub value: f64,
    #[serde(default = "mm")]
    pub unit: String,
    #[serde(default)]
    pub tolerance: Option<f64>,
    pub basis: Basis,
}

fn mm() -> String { "mm".into() }
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test --test basis`
Expected: PASS (5 tests).

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "Add basis taxonomy with per-kind validation"
```

---

### Task 5: Part model and the authored Alpha RV16AF-41 definition

**IMPORTANT:** Do not copy dimensions from this plan into `part.yaml` without checking them. Open `catalog/potentiometers/alpha-rv16af-41/taiwan-alpha-16mm-metal-shaft-series.pdf` and read **page 43** (RV16AF-41 outline drawing) and **page 45** (PCB mounting hole detail and terminal dimensions). Record what the drawing says. If a value below disagrees with the drawing, the drawing wins and you must correct the plan's value in your commit message.

Values transcribed from those pages during design, for cross-checking:

| Quantity | Value (mm) | Page |
|---|---|---|
| Body diameter, front view | Ø17 | 43 |
| Body diameter, side view | Ø16.5 | 43 |
| Body depth behind mounting face | 9.1 | 43 |
| Bushing thread | M7×0.75 | 43 |
| Bushing length | 6.5 | 43 |
| Shaft diameter | Ø6 (18 teeth) | 43 |
| Terminal hole diameter | Ø1.20 +0.2 | 43/45 |
| Terminal pitch | 5.0 | 43/45 |
| Locating boss hole | Ø7.5 +0.2 | 43/45 |
| Boss centre to terminal row | 16 | 43/45 |
| Type-41 size table | D 10.7, H 16 | 45 |
| Total rotation | 300° ±5° | 45 |

**Files:**
- Create: `src/model/part.rs`
- Create: `catalog/potentiometers/alpha-rv16af-41/part.yaml`
- Modify: `src/catalog/check.rs`, `src/cli.rs`
- Test: `tests/catalog_check.rs`

**Interfaces:**
- Consumes: `Basis`, `Dimension`, `Document`.
- Produces:
  - `struct Part { id, mpn, documents: Vec<Document>, pads: Vec<Pad>, geometry: Vec<Primitive>, interfaces: Vec<Interface>, panel: Option<PanelSpec> }`
  - `enum Primitive { Box { id, size: [Dimension;3], at: [f64;3] }, Cylinder { id, diameter: Dimension, length: Dimension, axis: Axis, at: [f64;3] } }`
  - `struct Pad { number: String, at: [f64;2], drill: Dimension, size: Dimension }`
  - `struct Interface { id: String, kind: InterfaceKind, geometry: String }`
  - `enum InterfaceKind { CylindricalAxis, PlanarInterface }`
  - `Part::load(path: &Path) -> Result<Part>`
  - `catalog::check::check_part(part: &Part, root: &Path) -> Result<()>`

- [ ] **Step 1: Write the failing test**

```rust
// tests/catalog_check.rs
use cctl::catalog::check::check_part;
use cctl::model::part::Part;
use std::path::Path;

fn part_dir() -> &'static Path {
    Path::new("catalog/potentiometers/alpha-rv16af-41")
}

#[test]
fn the_alpha_pot_loads_and_passes_the_gate() {
    let p = Part::load(&part_dir().join("part.yaml")).expect("loads");
    assert_eq!(p.id, "potentiometers/alpha-rv16af-41");
    check_part(&p, part_dir()).expect("basis gate passes");
}

#[test]
fn it_declares_three_pads_on_five_millimetre_pitch() {
    let p = Part::load(&part_dir().join("part.yaml")).unwrap();
    assert_eq!(p.pads.len(), 3, "three terminals");
    let mut xs: Vec<f64> = p.pads.iter().map(|pad| pad.at[0]).collect();
    xs.sort_by(|a, b| a.partial_cmp(b).unwrap());
    assert!((xs[1] - xs[0] - 5.0).abs() < 1e-9, "5 mm pitch");
    assert!((xs[2] - xs[1] - 5.0).abs() < 1e-9, "5 mm pitch");
}

#[test]
fn it_exposes_a_shaft_axis_interface() {
    let p = Part::load(&part_dir().join("part.yaml")).unwrap();
    assert!(p.interfaces.iter().any(|i| i.id == "shaft_axis"));
}

#[test]
fn a_dimension_with_a_bad_basis_fails_the_gate() {
    let yaml = r#"
id: test/bad
documents: []
pads: []
geometry:
  - cylinder:
      id: shaft
      diameter: {value: 6.0, basis: {type: manufacturer_drawing}}
      length: {value: 15.0, basis: {type: manufacturer_drawing, document: d, page: 1}}
      axis: z
      at: [0.0, 0.0, 0.0]
interfaces: []
"#;
    let p: Part = serde_yaml_ng::from_str(yaml).expect("parses");
    assert!(check_part(&p, Path::new(".")).is_err(),
        "documentary basis without a page must fail");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test --test catalog_check`
Expected: FAIL — `part` module and `part.yaml` do not exist.

- [ ] **Step 3: Write the model, then author the part**

```rust
// src/model/part.rs
use crate::model::basis::Dimension;
use crate::model::document::Document;
use anyhow::Result;
use serde::{Deserialize, Serialize};
use std::path::Path;

#[derive(Debug, Clone, Copy, Serialize, Deserialize, PartialEq, Eq)]
#[serde(rename_all = "lowercase")]
pub enum Axis { X, Y, Z }

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum Primitive {
    Box { id: String, size: [Dimension; 3], #[serde(default)] at: [f64; 3] },
    Cylinder { id: String, diameter: Dimension, length: Dimension, axis: Axis,
               #[serde(default)] at: [f64; 3] },
}

impl Primitive {
    pub fn id(&self) -> &str {
        match self { Primitive::Box { id, .. } | Primitive::Cylinder { id, .. } => id }
    }
    pub fn dimensions(&self) -> Vec<&Dimension> {
        match self {
            Primitive::Box { size, .. } => size.iter().collect(),
            Primitive::Cylinder { diameter, length, .. } => vec![diameter, length],
        }
    }
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize, PartialEq, Eq)]
#[serde(rename_all = "snake_case")]
pub enum InterfaceKind { CylindricalAxis, PlanarInterface }

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Interface {
    pub id: String,
    #[serde(rename = "type")]
    pub kind: InterfaceKind,
    pub geometry: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Pad {
    pub number: String,
    pub at: [f64; 2],
    pub drill: Dimension,
    pub size: Dimension,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PanelSpec { pub hole_diameter: Dimension }

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Part {
    pub id: String,
    #[serde(default)] pub mpn: Option<String>,
    #[serde(default)] pub documents: Vec<Document>,
    #[serde(default)] pub pads: Vec<Pad>,
    #[serde(default)] pub geometry: Vec<Primitive>,
    #[serde(default)] pub interfaces: Vec<Interface>,
    #[serde(default)] pub panel: Option<PanelSpec>,
}

impl Part {
    pub fn load(path: &Path) -> Result<Part> {
        let text = std::fs::read_to_string(path)?;
        Ok(serde_yaml_ng::from_str(&text)?)
    }
    pub fn interface(&self, id: &str) -> Option<&Interface> {
        self.interfaces.iter().find(|i| i.id == id)
    }
    pub fn primitive(&self, id: &str) -> Option<&Primitive> {
        self.geometry.iter().find(|g| g.id() == id)
    }
}
```

```rust
// src/catalog/check.rs
use crate::model::part::Part;
use anyhow::{Context, Result};
use std::path::Path;

pub fn check_part(part: &Part, root: &Path) -> Result<()> {
    for d in &part.documents {
        d.verify(root).with_context(|| format!("document {}", d.key))?;
    }
    let keys: Vec<&str> = part.documents.iter().map(|d| d.key.as_str()).collect();
    let mut check = |d: &crate::model::basis::Dimension, what: &str| -> Result<()> {
        d.basis.validate().with_context(|| format!("{what}"))?;
        if let Some(doc) = &d.basis.document {
            anyhow::ensure!(keys.contains(&doc.as_str()),
                "{what}: basis cites unknown document '{doc}'");
        }
        Ok(())
    };
    for g in &part.geometry {
        for d in g.dimensions() { check(d, &format!("geometry {}", g.id()))?; }
    }
    for p in &part.pads {
        check(&p.drill, &format!("pad {} drill", p.number))?;
        check(&p.size, &format!("pad {} size", p.number))?;
    }
    if let Some(panel) = &part.panel { check(&panel.hole_diameter, "panel hole")?; }
    Ok(())
}
```

Now author `catalog/potentiometers/alpha-rv16af-41/part.yaml`. Compute the
document hash first:

```bash
shasum -a 256 catalog/potentiometers/alpha-rv16af-41/taiwan-alpha-16mm-metal-shaft-series.pdf
```

Use the Write tool (not a heredoc — the file contains no `#`, but keep the habit)
to create exactly this, substituting the real hash and correcting any dimension
that pages 43 and 45 contradict. Pad 2 sits at the origin so the middle terminal
anchors the footprint; the shaft axis is on the same centre, and the locating
boss is 16 mm away along +Y in footprint space.

```yaml
id: potentiometers/alpha-rv16af-41
mpn: RV16AF-41

documents:
  - key: datasheet
    path: taiwan-alpha-16mm-metal-shaft-series.pdf
    sha256: PUT_THE_REAL_HASH_HERE
    retrieved: "2026-09-05"
    source_url: https://electricdruid.net/wp-content/uploads/2016/12/Taiwan-Alpha-16mm-pots-datasheet.pdf
    kind: manufacturer_datasheet

models: []

physical_validation:
  status: unmeasured

pads:
  - number: "1"
    at: [-5.0, 0.0]
    drill: {value: 1.2, tolerance: 0.2, basis: {type: manufacturer_datasheet, document: datasheet, page: 45, drawing: PCB Mounting Hole Detail}}
    size:  {value: 1.8, basis: {type: design_rule, rule: thru_hole_annular_ring}}
  - number: "2"
    at: [0.0, 0.0]
    drill: {value: 1.2, tolerance: 0.2, basis: {type: manufacturer_datasheet, document: datasheet, page: 45, drawing: PCB Mounting Hole Detail}}
    size:  {value: 1.8, basis: {type: design_rule, rule: thru_hole_annular_ring}}
  - number: "3"
    at: [5.0, 0.0]
    drill: {value: 1.2, tolerance: 0.2, basis: {type: manufacturer_datasheet, document: datasheet, page: 45, drawing: PCB Mounting Hole Detail}}
    size:  {value: 1.8, basis: {type: design_rule, rule: thru_hole_annular_ring}}

geometry:
  - box:
      id: body
      at: [0.0, 16.0, 0.0]
      size:
        - {value: 17.0, basis: {type: manufacturer_datasheet, document: datasheet, page: 43, drawing: RV16AF-41 Outline}}
        - {value: 17.0, basis: {type: manufacturer_datasheet, document: datasheet, page: 43, drawing: RV16AF-41 Outline}}
        - {value: 9.1,  basis: {type: manufacturer_datasheet, document: datasheet, page: 43, drawing: RV16AF-41 Outline}}
  - cylinder:
      id: bushing
      at: [0.0, 16.0, 9.1]
      axis: z
      diameter: {value: 7.0,  basis: {type: manufacturer_datasheet, document: datasheet, page: 43, drawing: RV16AF-41 Outline}}
      length:   {value: 6.5,  basis: {type: manufacturer_datasheet, document: datasheet, page: 43, drawing: RV16AF-41 Outline}}
  - cylinder:
      id: shaft
      at: [0.0, 16.0, 15.6]
      axis: z
      diameter: {value: 6.0,  basis: {type: manufacturer_datasheet, document: datasheet, page: 43, drawing: RV16AF-41 Outline}}
      length:   {value: 15.0, basis: {type: manufacturer_datasheet, document: datasheet, page: 43, drawing: RV16AF-41 Outline}}

interfaces:
  - id: shaft_axis
    type: cylindrical_axis
    geometry: shaft
  - id: pcb_seating_plane
    type: planar_interface
    geometry: body

panel:
  hole_diameter:
    value: 7.5
    tolerance: 0.2
    basis: {type: manufacturer_datasheet, document: datasheet, page: 45, drawing: PCB Mounting Hole Detail}
```

Note the `design_rule` basis on pad *size*: annular ring is our choice, not
Alpha's, and typing it honestly is the point of the basis taxonomy. Note also
that the `at` Z values chain — bushing starts where the body ends — so if you
correct a body depth the shaft moves with it.

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test --test catalog_check`
Expected: PASS (4 tests).

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "Add part model and author Alpha RV16AF-41 from manufacturer drawing"
```

---

### Task 6: Footprint emitter validated by kicad-cli

**Files:**
- Create: `src/kicad/footprint.rs`, `tests/kicad_oracle.rs`
- Test: `tests/footprint_emit.rs`

**Interfaces:**
- Consumes: `Part`.
- Produces:
  - `footprint::emit(part: &Part) -> String` — complete `.kicad_mod` text
  - `kicad_oracle::kicad_cli() -> PathBuf` (test helper)
  - `kicad_oracle::require_ok(cmd: &mut Command)` (test helper)

- [ ] **Step 1: Write the failing test**

```rust
// tests/kicad_oracle.rs
use std::path::PathBuf;
use std::process::Command;

pub fn kicad_cli() -> PathBuf {
    PathBuf::from("/Applications/KiCad/KiCad.app/Contents/MacOS/kicad-cli")
}

pub fn require_ok(cmd: &mut Command) -> String {
    let out = cmd.output().expect("kicad-cli runs");
    let stdout = String::from_utf8_lossy(&out.stdout).to_string();
    let stderr = String::from_utf8_lossy(&out.stderr).to_string();
    assert!(out.status.success(), "kicad-cli failed:\n{stdout}\n{stderr}");
    stdout
}
```

```rust
// tests/footprint_emit.rs
mod kicad_oracle;
use cctl::kicad::footprint;
use cctl::model::part::Part;
use std::path::Path;

#[test]
fn emitted_footprint_is_accepted_by_kicad() {
    let part = Part::load(
        Path::new("catalog/potentiometers/alpha-rv16af-41/part.yaml")).unwrap();
    let text = footprint::emit(&part);

    let td = tempfile::tempdir().unwrap();
    let lib = td.path().join("generated.pretty");
    std::fs::create_dir_all(&lib).unwrap();
    std::fs::write(lib.join("alpha-rv16af-41.kicad_mod"), &text).unwrap();

    // KiCad parses and normalises the library; failure means we emitted garbage.
    kicad_oracle::require_ok(
        std::process::Command::new(kicad_oracle::kicad_cli())
            .args(["fp", "upgrade"])
            .arg(&lib));
}

#[test]
fn emitted_footprint_identifies_us_as_the_generator() {
    let part = Part::load(
        Path::new("catalog/potentiometers/alpha-rv16af-41/part.yaml")).unwrap();
    let text = footprint::emit(&part);
    assert!(text.contains("(generator \"circuit-control\")"),
        "third-party writers must not impersonate KiCad");
}

#[test]
fn emits_one_pad_per_declared_terminal() {
    let part = Part::load(
        Path::new("catalog/potentiometers/alpha-rv16af-41/part.yaml")).unwrap();
    let text = footprint::emit(&part);
    assert_eq!(text.matches("(pad ").count(), part.pads.len());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test --test footprint_emit`
Expected: FAIL — `footprint` module missing.

- [ ] **Step 3: Write minimal implementation**

```rust
// src/kicad/footprint.rs
use crate::model::part::Part;

fn short_name(id: &str) -> &str {
    id.rsplit('/').next().unwrap_or(id)
}

pub fn emit(part: &Part) -> String {
    let name = short_name(&part.id);
    let mut s = String::new();
    s.push_str(&format!(
        "(footprint \"{name}\"\n\t(version 20260206)\n\t(generator \"circuit-control\")\n\t(generator_version \"1\")\n\t(layer \"F.Cu\")\n"
    ));
    s.push_str(&format!(
        "\t(property \"Reference\" \"REF**\"\n\t\t(at 0 -2 0)\n\t\t(layer \"F.SilkS\")\n\t\t(effects (font (size 1 1) (thickness 0.15)))\n\t)\n"
    ));
    s.push_str(&format!(
        "\t(property \"Value\" \"{name}\"\n\t\t(at 0 2 0)\n\t\t(layer \"F.Fab\")\n\t\t(effects (font (size 1 1) (thickness 0.15)))\n\t)\n"
    ));
    for pad in &part.pads {
        s.push_str(&format!(
            "\t(pad \"{}\" thru_hole circle\n\t\t(at {} {})\n\t\t(size {} {})\n\t\t(drill {})\n\t\t(layers \"*.Cu\" \"*.Mask\")\n\t)\n",
            pad.number, pad.at[0], pad.at[1],
            pad.size.value, pad.size.value, pad.drill.value
        ));
    }
    s.push_str(")\n");
    s
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test --test footprint_emit`
Expected: PASS (3 tests). If `fp upgrade` rejects the file, read its error and fix the emitted syntax — KiCad is the authority here, not this plan.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "Emit KiCad footprints from part definitions, validated by kicad-cli"
```

---

### Task 7: Assign the footprint to RV1 in the schematic

**Files:**
- Create: `src/kicad/files.rs`
- Modify: `src/kicad/mod.rs`
- Test: `tests/schematic_assign.rs`

**Interfaces:**
- Consumes: `sexpr::Doc`.
- Produces:
  - `trait KiCadBackend { fn set_footprint(&self, sch: &Path, reference: &str, value: &str) -> Result<()>; fn place_footprint(&self, pcb: &Path, reference: &str, lib: &str, at: Placement) -> Result<()>; }`
  - `struct SexprBackend;` implementing it
  - `struct Placement { pub x: f64, pub y: f64, pub rotation: f64 }`

- [ ] **Step 1: Write the failing test**

```rust
// tests/schematic_assign.rs
mod fixtures;
mod kicad_oracle;
use cctl::kicad::{KiCadBackend, SexprBackend};

#[test]
fn sets_the_footprint_property_for_rv1_only() {
    let (_td, dir) = fixtures::temp_project();
    let sch = dir.join("fuzz.kicad_sch");
    let before = std::fs::read_to_string(&sch).unwrap();
    assert_eq!(before.matches(r#""Footprint" """#).count(), 76,
        "fixture starts with 76 empty footprint fields");

    SexprBackend.set_footprint(&sch, "RV1", "generated:alpha-rv16af-41").unwrap();

    let after = std::fs::read_to_string(&sch).unwrap();
    assert!(after.contains(r#""Footprint" "generated:alpha-rv16af-41""#));
    assert_eq!(after.matches(r#""Footprint" """#).count(), 75,
        "exactly one field filled");
}

#[test]
fn kicad_agrees_the_assignment_landed() {
    let (_td, dir) = fixtures::temp_project();
    let sch = dir.join("fuzz.kicad_sch");
    SexprBackend.set_footprint(&sch, "RV1", "generated:alpha-rv16af-41").unwrap();

    let bom = dir.join("bom.csv");
    kicad_oracle::require_ok(
        std::process::Command::new(kicad_oracle::kicad_cli())
            .args(["sch", "export", "bom", "--fields", "Reference,Footprint",
                   "--output"])
            .arg(&bom)
            .arg(&sch));
    let csv = std::fs::read_to_string(&bom).unwrap();
    assert!(csv.contains("alpha-rv16af-41"), "KiCad's own BOM must show it:\n{csv}");
}

#[test]
fn unknown_reference_is_an_error_and_writes_nothing() {
    let (_td, dir) = fixtures::temp_project();
    let sch = dir.join("fuzz.kicad_sch");
    let before = std::fs::read_to_string(&sch).unwrap();
    assert!(SexprBackend.set_footprint(&sch, "RV99", "x:y").is_err());
    assert_eq!(std::fs::read_to_string(&sch).unwrap(), before, "no partial write");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test --test schematic_assign`
Expected: FAIL — `SexprBackend` missing.

- [ ] **Step 3: Write minimal implementation**

```rust
// src/kicad/files.rs
use super::sexpr::Doc;
use anyhow::{bail, Result};
use std::path::Path;

#[derive(Debug, Clone, Copy)]
pub struct Placement { pub x: f64, pub y: f64, pub rotation: f64 }

pub trait KiCadBackend {
    fn set_footprint(&self, sch: &Path, reference: &str, value: &str) -> Result<()>;
}

pub struct SexprBackend;

impl KiCadBackend for SexprBackend {
    fn set_footprint(&self, sch: &Path, reference: &str, value: &str) -> Result<()> {
        let src = std::fs::read_to_string(sch)?;
        let mut doc = Doc::parse(&src)?;

        let target = {
            let root = doc.root();
            let mut found = None;
            for sym in root.find_children("symbol") {
                let is_ours = sym.find_children("property").iter().any(|p| {
                    let a = p.atoms();
                    a.first().map(|s| s.trim_matches('"')) == Some("Reference")
                        && a.get(1).map(|s| s.trim_matches('"')) == Some(reference)
                });
                if !is_ours { continue; }
                let fp = sym.find_children("property").into_iter().find(|p| {
                    p.atoms().first().map(|s| s.trim_matches('"')) == Some("Footprint")
                });
                found = fp.map(|p| p.span());
                break;
            }
            found
        };

        let Some(span) = target else {
            bail!("no symbol {reference} with a Footprint property in {}", sch.display());
        };

        let original = &src[span.0..span.1];
        let replacement = original.replacen(r#""Footprint" """#,
            &format!(r#""Footprint" "{value}""#), 1);
        if replacement == original {
            bail!("Footprint property for {reference} was not empty; refusing to overwrite");
        }
        doc.replace_span(span, replacement);
        std::fs::write(sch, doc.to_string())?;
        Ok(())
    }
}
```

Add to `src/kicad/mod.rs`:

```rust
pub mod files;
pub mod footprint;
pub mod sexpr;
pub use files::{KiCadBackend, Placement, SexprBackend};
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test --test schematic_assign`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "Assign generated footprints to schematic symbols behind KiCadBackend"
```

---

### Task 8: Coordinate frames and the board-to-world flip

**Files:**
- Create: `src/geometry/mod.rs`, `src/geometry/frames.rs`
- Test: `tests/frames.rs`

**Interfaces:**
- Consumes: nothing.
- Produces:
  - `struct PcbOrigin { pub at: [f64; 3] }`
  - `frames::board_to_world(origin: &PcbOrigin, board: [f64; 2]) -> [f64; 3]`
  - `frames::rotate_z(v: [f64; 3], degrees: f64) -> [f64; 3]`

- [ ] **Step 1: Write the failing test**

```rust
// tests/frames.rs
use cctl::geometry::frames::{board_to_world, PcbOrigin};

/// Deliberately asymmetric: a mirrored transform cannot satisfy both points.
#[test]
fn board_y_is_inverted_into_world() {
    let o = PcbOrigin { at: [10.0, 20.0, 5.0] };
    let a = board_to_world(&o, [3.0, 7.0]);
    assert!((a[0] - 13.0).abs() < 1e-9, "x adds: {a:?}");
    assert!((a[1] - 13.0).abs() < 1e-9, "y subtracts: {a:?}");
    assert!((a[2] - 5.0).abs() < 1e-9);
}

#[test]
fn asymmetric_pair_rejects_a_mirrored_transform() {
    let o = PcbOrigin { at: [0.0, 0.0, 0.0] };
    let p = board_to_world(&o, [1.0, 4.0]);
    let q = board_to_world(&o, [4.0, 1.0]);
    // Under the correct flip these are distinct and not each other's mirror.
    assert!((p[0] - 1.0).abs() < 1e-9 && (p[1] + 4.0).abs() < 1e-9, "{p:?}");
    assert!((q[0] - 4.0).abs() < 1e-9 && (q[1] + 1.0).abs() < 1e-9, "{q:?}");
    assert_ne!(p, q);
}

#[test]
fn rotation_is_about_world_z() {
    let v = cctl::geometry::frames::rotate_z([1.0, 0.0, 0.0], 90.0);
    assert!((v[0]).abs() < 1e-9, "{v:?}");
    assert!((v[1] - 1.0).abs() < 1e-9, "{v:?}");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test --test frames`
Expected: FAIL — module missing.

- [ ] **Step 3: Write minimal implementation**

```rust
// src/geometry/frames.rs
use serde::{Deserialize, Serialize};

/// Where the KiCad board origin sits in world coordinates.
/// World is Z-up, right-handed, millimetres.
#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub struct PcbOrigin { pub at: [f64; 3] }

/// KiCad board space is millimetres with Y increasing downward.
/// The board plane maps to world XY, so Y is negated.
pub fn board_to_world(origin: &PcbOrigin, board: [f64; 2]) -> [f64; 3] {
    [origin.at[0] + board[0], origin.at[1] - board[1], origin.at[2]]
}

pub fn rotate_z(v: [f64; 3], degrees: f64) -> [f64; 3] {
    let r = degrees.to_radians();
    let (s, c) = r.sin_cos();
    [v[0] * c - v[1] * s, v[0] * s + v[1] * c, v[2]]
}
```

```rust
// src/geometry/mod.rs
pub mod frames;
pub mod primitives;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test --test frames`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "Add coordinate frames with tested board-to-world handedness flip"
```

---

### Task 9: Project configuration and the 1590BBS enclosure part

Read `catalog/mechanical/hammond-1590bbs/hammond-1590bbs.pdf` page 1 and record the enclosure's dimensions. Values transcribed during design, for cross-checking: inside 113.48 L × 87.98 W × 37.85 H, wall 2.25, lid 2.00 thick, external 119.50 × 94.00 × 42.10, drawing REV 25.05.2020.

**Files:**
- Create: `src/model/project.rs`
- Create: `catalog/mechanical/hammond-1590bbs/part.yaml`
- Create: `~/work/pedals/fuzz/mech/{parts,panel,enclosure,checks}.yaml` — but write them into `tests/data/mech/` for tests; the operator's copy is created by hand later.
- Test: `tests/project_config.rs`

**Interfaces:**
- Consumes: `Part`, `PcbOrigin`.
- Produces:
  - `struct ProjectConfig { bindings: Vec<Binding>, panel: PanelLayout, enclosure: EnclosureConfig, checks: Vec<CheckSpec> }`
  - `struct Binding { refs: Vec<String>, package: String }`
  - `struct PanelLayout { controls: Vec<PanelControl> }`
  - `struct PanelControl { id: String, reference: String, at: [f64; 2], rotation: f64 }`
  - `struct EnclosureConfig { part: String, pcb_origin: PcbOrigin }`
  - `struct CheckSpec { id: String, kind: String, a: String, b: String, max_offset: f64 }`
  - `ProjectConfig::load(dir: &Path) -> Result<ProjectConfig>`

- [ ] **Step 1: Write the failing test**

```rust
// tests/project_config.rs
use cctl::model::project::ProjectConfig;
use std::path::Path;

#[test]
fn loads_the_fuzz_mechanical_configuration() {
    let cfg = ProjectConfig::load(Path::new("tests/data/mech")).expect("loads");
    assert!(cfg.bindings.iter().any(|b| b.refs.contains(&"RV1".to_string())));
    assert!(cfg.panel.controls.iter().any(|c| c.reference == "RV1"));
    assert_eq!(cfg.enclosure.part, "mechanical/hammond-1590bbs");
    assert_eq!(cfg.checks.len(), 1);
    assert_eq!(cfg.checks[0].id, "rv1-shaft-to-panel-hole");
}

#[test]
fn the_pcb_origin_is_operator_declared_not_inferred() {
    let cfg = ProjectConfig::load(Path::new("tests/data/mech")).unwrap();
    // The tool must never compute this; it only reads it.
    assert!(cfg.enclosure.pcb_origin.at[2] != 0.0,
        "fixture declares a non-trivial Z so a missing transform cannot pass by accident");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test --test project_config`
Expected: FAIL — module and test data missing.

- [ ] **Step 3: Write the model and the config files**

```rust
// src/model/project.rs
use crate::geometry::frames::PcbOrigin;
use anyhow::Result;
use serde::{Deserialize, Serialize};
use std::path::Path;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Binding {
    #[serde(default)] pub refs: Vec<String>,
    pub package: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PanelControl {
    pub id: String,
    pub reference: String,
    pub at: [f64; 2],
    #[serde(default)] pub rotation: f64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PanelLayout { pub controls: Vec<PanelControl> }

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EnclosureConfig { pub part: String, pub pcb_origin: PcbOrigin }

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CheckSpec {
    pub id: String,
    #[serde(rename = "type")] pub kind: String,
    pub a: String,
    pub b: String,
    pub max_offset: f64,
}

#[derive(Debug, Clone)]
pub struct ProjectConfig {
    pub bindings: Vec<Binding>,
    pub panel: PanelLayout,
    pub enclosure: EnclosureConfig,
    pub checks: Vec<CheckSpec>,
}

#[derive(Deserialize)] struct BindingsFile { bindings: Vec<Binding> }
#[derive(Deserialize)] struct ChecksFile { checks: Vec<CheckSpec> }

impl ProjectConfig {
    pub fn load(dir: &Path) -> Result<ProjectConfig> {
        let read = |n: &str| -> Result<String> {
            Ok(std::fs::read_to_string(dir.join(n))?)
        };
        let b: BindingsFile = serde_yaml_ng::from_str(&read("parts.yaml")?)?;
        let panel: PanelLayout = serde_yaml_ng::from_str(&read("panel.yaml")?)?;
        let enclosure: EnclosureConfig = serde_yaml_ng::from_str(&read("enclosure.yaml")?)?;
        let c: ChecksFile = serde_yaml_ng::from_str(&read("checks.yaml")?)?;
        Ok(ProjectConfig { bindings: b.bindings, panel, enclosure: enclosure, checks: c.checks })
    }
}
```

Create the four test-data files with the Write tool. `tests/data/mech/parts.yaml`:

```yaml
bindings:
  - refs: [RV1]
    package: potentiometers/alpha-rv16af-41
```

`tests/data/mech/panel.yaml`:

```yaml
controls:
  - id: vol
    reference: RV1
    at: [30.0, 25.0]
    rotation: 0.0
```

`tests/data/mech/enclosure.yaml` — the PCB origin is operator-declared; pick a
value inside the 1590BBS interior and state that it is declared, not derived:

```yaml
part: mechanical/hammond-1590bbs
pcb_origin:
  at: [10.0, -10.0, 12.0]
```

`tests/data/mech/checks.yaml`:

```yaml
checks:
  - id: rv1-shaft-to-panel-hole
    type: axis_alignment
    a: RV1.shaft_axis
    b: panel.vol
    max_offset: 0.25
```

Also author `catalog/mechanical/hammond-1590bbs/part.yaml`. Hash the PDF first
(`shasum -a 256 catalog/mechanical/hammond-1590bbs/hammond-1590bbs.pdf`) and
correct any dimension the drawing contradicts:

```yaml
id: mechanical/hammond-1590bbs
mpn: 1590BBS

documents:
  - key: drawing
    path: hammond-1590bbs.pdf
    sha256: PUT_THE_REAL_HASH_HERE
    retrieved: "2026-09-04"
    source_url: https://www.hammfg.com/files/parts/pdf/1590BBS.pdf
    kind: manufacturer_drawing

models: []

physical_validation:
  status: unmeasured

pads: []

geometry:
  - box:
      id: interior
      at: [0.0, 0.0, 0.0]
      size:
        - {value: 113.48, basis: {type: manufacturer_drawing, document: drawing, page: 1, drawing: Section B-B Side View}}
        - {value: 87.98,  basis: {type: manufacturer_drawing, document: drawing, page: 1, drawing: Section A-A End View}}
        - {value: 37.85,  basis: {type: manufacturer_drawing, document: drawing, page: 1, drawing: Section A-A End View}}
  - cylinder:
      id: vol_hole
      at: [30.0, 25.0, 37.85]
      axis: z
      diameter: {value: 7.5, tolerance: 0.2, basis: {type: design_rule, rule: panel_hole_from_bushing}}
      length:   {value: 2.25, basis: {type: manufacturer_drawing, document: drawing, page: 1, drawing: Section B-B Side View}}

interfaces:
  - id: vol
    type: cylindrical_axis
    geometry: vol_hole
  - id: interior_volume
    type: planar_interface
    geometry: interior
```

Two things to notice. The panel hole's **position** repeats `panel.yaml`'s
control coordinate — in M2 the enclosure's holes will be generated from
`panel.yaml` rather than restated, but for one control at n=1 restating it keeps
the task small; leave a comment saying so. Its **diameter** carries a
`design_rule` basis, not a documentary one, because Hammond's drawing says
nothing about a hole you will drill yourself — the size follows from the pot's
M7 bushing, and typing it as a rule rather than inventing a citation is exactly
what the basis taxonomy is for.

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test --test project_config`
Expected: PASS (2 tests).

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "Add project configuration model and Hammond 1590BBS enclosure part"
```

---

### Task 10: Derive class-A placement and write it to the board

**Files:**
- Create: `src/placement.rs`
- Modify: `src/kicad/files.rs`
- Test: `tests/placement.rs`

**Interfaces:**
- Consumes: `ProjectConfig`, `Part`, `frames`, `KiCadBackend`.
- Produces:
  - `placement::derive(cfg: &ProjectConfig, control: &PanelControl) -> Placement` — board coordinates for a panel-declared control
  - `KiCadBackend::place_footprint(&self, pcb: &Path, reference: &str, lib_id: &str, at: Placement) -> Result<()>`

- [ ] **Step 1: Write the failing test**

```rust
// tests/placement.rs
mod fixtures;
mod kicad_oracle;
use cctl::kicad::{KiCadBackend, SexprBackend};
use cctl::model::project::ProjectConfig;
use std::path::Path;

#[test]
fn derived_placement_inverts_the_declared_pcb_origin() {
    let cfg = ProjectConfig::load(Path::new("tests/data/mech")).unwrap();
    let ctl = &cfg.panel.controls[0];
    let p = cctl::placement::derive(&cfg, ctl);
    // world.x = origin.x + board.x  =>  board.x = panel.x - origin.x
    assert!((p.x - (ctl.at[0] - cfg.enclosure.pcb_origin.at[0])).abs() < 1e-9);
    // world.y = origin.y - board.y  =>  board.y = origin.y - panel.y
    assert!((p.y - (cfg.enclosure.pcb_origin.at[1] - ctl.at[1])).abs() < 1e-9);
}

#[test]
fn kicad_agrees_where_we_placed_rv1() {
    let (_td, dir) = fixtures::temp_project();
    let pcb = dir.join("fuzz.kicad_pcb");
    let cfg = ProjectConfig::load(Path::new("tests/data/mech")).unwrap();
    let ctl = &cfg.panel.controls[0];
    let at = cctl::placement::derive(&cfg, ctl);

    SexprBackend.place_footprint(&pcb, "RV1", "generated:alpha-rv16af-41", at).unwrap();

    let pos = dir.join("pos.csv");
    kicad_oracle::require_ok(
        std::process::Command::new(kicad_oracle::kicad_cli())
            .args(["pcb", "export", "pos", "--format", "csv", "--units", "mm",
                   "--side", "both", "--output"])
            .arg(&pos)
            .arg(&pcb));
    let csv = std::fs::read_to_string(&pos).unwrap();
    assert!(csv.contains("RV1"), "KiCad must see RV1:\n{csv}");
    let line = csv.lines().find(|l| l.starts_with("RV1") || l.contains("\"RV1\""))
        .expect("RV1 row");
    assert!(line.contains(&format!("{:.4}", at.x)) || line.contains(&format!("{:.3}", at.x)),
        "KiCad's X must match ours: {line}");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test --test placement`
Expected: FAIL — `placement` module and `place_footprint` missing.

- [ ] **Step 3: Write minimal implementation**

```rust
// src/placement.rs
use crate::kicad::Placement;
use crate::model::project::{PanelControl, ProjectConfig};

/// Placement runs backwards from the enclosure: the operator declares where a
/// control sits on the panel, and we solve for the board coordinate that puts
/// the part there. Class-A placement is derived, never authored in KiCad.
pub fn derive(cfg: &ProjectConfig, control: &PanelControl) -> Placement {
    let o = cfg.enclosure.pcb_origin.at;
    Placement {
        x: control.at[0] - o[0],
        y: o[1] - control.at[1],
        rotation: control.rotation,
    }
}
```

Extend the trait and implementation in `src/kicad/files.rs`:

```rust
pub trait KiCadBackend {
    fn set_footprint(&self, sch: &Path, reference: &str, value: &str) -> Result<()>;
    fn place_footprint(&self, pcb: &Path, reference: &str, lib_id: &str,
                       at: Placement) -> Result<()>;
}
```

```rust
    fn place_footprint(&self, pcb: &Path, reference: &str, lib_id: &str,
                       at: Placement) -> Result<()> {
        let src = std::fs::read_to_string(pcb)?;
        let doc = Doc::parse(&src)?;
        let root_span = doc.root().span();
        // Insert before the board's closing paren.
        let insert_at = root_span.1 - 1;

        let block = format!(
            "\t(footprint \"{lib_id}\"\n\t\t(layer \"F.Cu\")\n\t\t(uuid \"{uuid}\")\n\t\t(at {x} {y} {r})\n\t\t(property \"Reference\" \"{reference}\"\n\t\t\t(at 0 -2 0)\n\t\t\t(layer \"F.SilkS\")\n\t\t\t(effects (font (size 1 1) (thickness 0.15)))\n\t\t)\n\t)\n",
            uuid = deterministic_uuid(reference),
            x = at.x, y = at.y, r = at.rotation
        );

        let mut doc = Doc::parse(&src)?;
        doc.replace_span((insert_at, insert_at), block);
        std::fs::write(pcb, doc.to_string())?;
        Ok(())
    }
```

```rust
/// Deterministic so that repeated sync runs do not churn the board file.
fn deterministic_uuid(seed: &str) -> String {
    use sha2::{Digest, Sha256};
    let mut h = Sha256::new();
    h.update(b"circuit-control:");
    h.update(seed.as_bytes());
    let d = format!("{:x}", h.finalize());
    format!("{}-{}-{}-{}-{}", &d[0..8], &d[8..12], &d[12..16], &d[16..20], &d[20..32])
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test --test placement`
Expected: PASS (2 tests). If KiCad rejects the emitted footprint block, read its error and correct the syntax against `dev-docs.kicad.org` board format.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "Derive class-A placement from panel coordinates and write it to the board"
```

---

### Task 11: Geometry backend and assembly construction

**Files:**
- Create: `src/geometry/primitives.rs`, `src/geometry/convex.rs`, `src/assembly.rs`
- Test: `tests/assembly.rs`

**Interfaces:**
- Consumes: `Part`, `ProjectConfig`, `frames`.
- Produces:
  - `struct Solid { pub id: String, pub kind: SolidKind, pub origin: [f64;3], pub axis: [f64;3] }`
  - `enum SolidKind { Box { size: [f64;3] }, Cylinder { diameter: f64, length: f64 } }`
  - `trait GeometryBackend { fn axis_offset(&self, a: &Solid, b: &Solid) -> f64; }`
  - `struct ConvexBackend;`
  - `struct Assembly { pub solids: Vec<Solid>, pub provenance: HashMap<String, Vec<Dependency>> }`
  - `assembly::build(cfg: &ProjectConfig, parts: &HashMap<String, Part>, board_placements: &HashMap<String, Placement>) -> Result<Assembly>`
  - `assembly::build_for_test(cfg: &ProjectConfig) -> Result<Assembly>` — loads parts from the repo catalog and derives placements via `placement::derive`, so constraint tests need no KiCad round trip
  - `Assembly::interface(&self, path: &str) -> Option<&Solid>` — resolves `"RV1.shaft_axis"` and `"panel.vol"`
  - `Assembly::provenance_for(&self, path: &str) -> Vec<Dependency>` — one entry per dimension backing that interface's primitive
  - `assembly::build_from_board(project: &Path, cfg: &ProjectConfig) -> Result<Assembly>` — added in Task 14; reads each class-A footprint's **actual** placement out of the board rather than deriving it, which is what makes drift detectable

- [ ] **Step 1: Write the failing test**

```rust
// tests/assembly.rs
use cctl::geometry::convex::ConvexBackend;
use cctl::geometry::{GeometryBackend, Solid, SolidKind};

#[test]
fn coaxial_cylinders_have_zero_axis_offset() {
    let a = Solid { id: "a".into(), kind: SolidKind::Cylinder { diameter: 6.0, length: 15.0 },
                    origin: [10.0, 20.0, 0.0], axis: [0.0, 0.0, 1.0] };
    let b = Solid { id: "b".into(), kind: SolidKind::Cylinder { diameter: 12.5, length: 2.25 },
                    origin: [10.0, 20.0, 30.0], axis: [0.0, 0.0, 1.0] };
    assert!(ConvexBackend.axis_offset(&a, &b).abs() < 1e-9);
}

#[test]
fn lateral_displacement_is_measured_perpendicular_to_the_axis() {
    let a = Solid { id: "a".into(), kind: SolidKind::Cylinder { diameter: 6.0, length: 15.0 },
                    origin: [10.0, 20.0, 0.0], axis: [0.0, 0.0, 1.0] };
    let b = Solid { id: "b".into(), kind: SolidKind::Cylinder { diameter: 12.5, length: 2.25 },
                    origin: [13.0, 24.0, 30.0], axis: [0.0, 0.0, 1.0] };
    // 3-4-5 triangle: offset is 5, and Z separation must not contribute.
    assert!((ConvexBackend.axis_offset(&a, &b) - 5.0).abs() < 1e-9);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test --test assembly`
Expected: FAIL — geometry backend missing.

- [ ] **Step 3: Write minimal implementation**

```rust
// src/geometry/primitives.rs
#[derive(Debug, Clone)]
pub enum SolidKind {
    Box { size: [f64; 3] },
    Cylinder { diameter: f64, length: f64 },
}

#[derive(Debug, Clone)]
pub struct Solid {
    pub id: String,
    pub kind: SolidKind,
    pub origin: [f64; 3],
    pub axis: [f64; 3],
}
```

```rust
// src/geometry/convex.rs
use super::{GeometryBackend, Solid};

pub struct ConvexBackend;

fn sub(a: [f64; 3], b: [f64; 3]) -> [f64; 3] { [a[0]-b[0], a[1]-b[1], a[2]-b[2]] }
fn dot(a: [f64; 3], b: [f64; 3]) -> f64 { a[0]*b[0] + a[1]*b[1] + a[2]*b[2] }
fn norm(a: [f64; 3]) -> f64 { dot(a, a).sqrt() }

impl GeometryBackend for ConvexBackend {
    /// Perpendicular distance between two parallel axes. Displacement along the
    /// shared axis is not misalignment, so it is projected out.
    fn axis_offset(&self, a: &Solid, b: &Solid) -> f64 {
        let d = sub(b.origin, a.origin);
        let n = norm(a.axis);
        let u = [a.axis[0]/n, a.axis[1]/n, a.axis[2]/n];
        let along = dot(d, u);
        let perp = [d[0] - along*u[0], d[1] - along*u[1], d[2] - along*u[2]];
        norm(perp)
    }
}
```

Add to `src/geometry/mod.rs`:

```rust
pub mod convex;
pub mod frames;
pub mod primitives;
pub use primitives::{Solid, SolidKind};

pub trait GeometryBackend {
    fn axis_offset(&self, a: &Solid, b: &Solid) -> f64;
}
```

Then write `src/assembly.rs`:

```rust
// src/assembly.rs
use crate::constraints::Dependency;
use crate::geometry::frames::board_to_world;
use crate::geometry::{Solid, SolidKind};
use crate::kicad::Placement;
use crate::model::part::{Part, Primitive};
use crate::model::project::ProjectConfig;
use crate::placement;
use anyhow::{anyhow, Result};
use std::collections::HashMap;
use std::path::Path;

pub struct Assembly {
    pub solids: Vec<Solid>,
    pub provenance: HashMap<String, Vec<Dependency>>,
}

fn solid_from(prim: &Primitive, world_at: [f64; 3]) -> Solid {
    match prim {
        Primitive::Box { id, size, .. } => Solid {
            id: id.clone(),
            kind: SolidKind::Box { size: [size[0].value, size[1].value, size[2].value] },
            origin: world_at,
            axis: [0.0, 0.0, 1.0],
        },
        Primitive::Cylinder { id, diameter, length, .. } => Solid {
            id: id.clone(),
            kind: SolidKind::Cylinder { diameter: diameter.value, length: length.value },
            origin: world_at,
            axis: [0.0, 0.0, 1.0],
        },
    }
}

fn deps_for(part: &Part, prim: &Primitive, label: &str) -> Vec<Dependency> {
    prim.dimensions().into_iter().map(|d| {
        let source = match (&d.basis.document, &d.basis.page) {
            (Some(doc), Some(page)) => {
                let path = part.documents.iter()
                    .find(|x| &x.key == doc)
                    .map(|x| x.path.display().to_string())
                    .unwrap_or_else(|| doc.clone());
                format!("{path} p.{page}")
            }
            _ => d.basis.rule.clone()
                .or_else(|| d.basis.rationale.clone())
                .unwrap_or_else(|| "declared".into()),
        };
        Dependency { what: format!("{label} {}", prim.id()), value: Some(d.value),
                     basis: d.basis.kind, source }
    }).collect()
}

pub fn build(cfg: &ProjectConfig, parts: &HashMap<String, Part>,
             board_placements: &HashMap<String, Placement>) -> Result<Assembly> {
    let mut solids = Vec::new();
    let mut provenance: HashMap<String, Vec<Dependency>> = HashMap::new();

    // Enclosure sits in world coordinates directly.
    let enc = parts.get(&cfg.enclosure.part)
        .ok_or_else(|| anyhow!("enclosure part {} not loaded", cfg.enclosure.part))?;
    for iface in &enc.interfaces {
        let prim = enc.primitive(&iface.geometry)
            .ok_or_else(|| anyhow!("interface {} names unknown geometry", iface.id))?;
        let at = match prim { Primitive::Box { at, .. } | Primitive::Cylinder { at, .. } => *at };
        let mut s = solid_from(prim, at);
        s.id = format!("panel.{}", iface.id);
        provenance.insert(s.id.clone(), deps_for(enc, prim, "enclosure"));
        solids.push(s);
    }

    // Board-mounted parts cross the board-to-world flip.
    for binding in &cfg.bindings {
        let part = parts.get(&binding.package)
            .ok_or_else(|| anyhow!("part {} not loaded", binding.package))?;
        for reference in &binding.refs {
            let Some(pl) = board_placements.get(reference) else { continue };
            for iface in &part.interfaces {
                let prim = part.primitive(&iface.geometry)
                    .ok_or_else(|| anyhow!("interface {} names unknown geometry", iface.id))?;
                let local = match prim {
                    Primitive::Box { at, .. } | Primitive::Cylinder { at, .. } => *at
                };
                let w = board_to_world(&cfg.enclosure.pcb_origin,
                                       [pl.x + local[0], pl.y + local[1]]);
                let mut s = solid_from(prim, [w[0], w[1], w[2] + local[2]]);
                s.id = format!("{reference}.{}", iface.id);
                provenance.insert(s.id.clone(), deps_for(part, prim, reference));
                solids.push(s);
            }
        }
    }
    Ok(Assembly { solids, provenance })
}

pub fn build_for_test(cfg: &ProjectConfig) -> Result<Assembly> {
    let mut parts = HashMap::new();
    let mut load = |id: &str| -> Result<()> {
        let p = Path::new("catalog").join(id).join("part.yaml");
        parts.insert(id.to_string(), Part::load(&p)?);
        Ok(())
    };
    load(&cfg.enclosure.part)?;
    for b in &cfg.bindings { load(&b.package)?; }

    let mut placements = HashMap::new();
    for c in &cfg.panel.controls {
        placements.insert(c.reference.clone(), placement::derive(cfg, c));
    }
    build(cfg, &parts, &placements)
}

impl Assembly {
    pub fn interface(&self, path: &str) -> Option<&Solid> {
        self.solids.iter().find(|s| s.id == path)
    }
    pub fn provenance_for(&self, path: &str) -> Vec<Dependency> {
        self.provenance.get(path).cloned().unwrap_or_default()
    }
}
```

Note `Dependency` lives in `constraints`, which is written in Task 12. Write the
`Dependency` struct into `src/constraints/mod.rs` as part of *this* task so
`assembly.rs` compiles, then Task 12 adds the rest of that module.

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test --test assembly`
Expected: PASS (2 tests).

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "Add convex geometry backend and world-frame assembly construction"
```

---

### Task 12: Alignment constraint with dependency provenance

**Files:**
- Create: `src/constraints/mod.rs`, `src/constraints/alignment.rs`
- Test: `tests/constraints.rs`

**Interfaces:**
- Consumes: `Assembly`, `GeometryBackend`, `CheckSpec`, `Basis`.
- Produces:
  - `struct Dependency { pub what: String, pub value: Option<f64>, pub basis: BasisKind, pub source: String }`
  - `struct CheckResult { pub id: String, pub status: Status, pub required: f64, pub measured: f64, pub dependencies: Vec<Dependency> }`
  - `enum Status { Pass, Fail }`
  - `alignment::evaluate(spec: &CheckSpec, asm: &Assembly, g: &dyn GeometryBackend) -> Result<CheckResult>`

- [ ] **Step 1: Write the failing test**

```rust
// tests/constraints.rs
use cctl::constraints::{alignment, Status};
use cctl::geometry::convex::ConvexBackend;
use cctl::model::project::ProjectConfig;
use std::path::Path;

fn built() -> (ProjectConfig, cctl::assembly::Assembly) {
    let cfg = ProjectConfig::load(Path::new("tests/data/mech")).unwrap();
    let asm = cctl::assembly::build_for_test(&cfg).expect("assembly builds");
    (cfg, asm)
}

#[test]
fn aligned_shaft_and_panel_hole_pass() {
    let (cfg, asm) = built();
    let r = alignment::evaluate(&cfg.checks[0], &asm, &ConvexBackend).unwrap();
    assert_eq!(r.status, Status::Pass, "measured {} required {}", r.measured, r.required);
    assert!(r.measured <= r.required);
}

#[test]
fn results_carry_dependency_provenance() {
    let (cfg, asm) = built();
    let r = alignment::evaluate(&cfg.checks[0], &asm, &ConvexBackend).unwrap();
    assert!(!r.dependencies.is_empty(), "a result must say what it depends on");
    assert!(r.dependencies.iter().any(|d| d.what.contains("shaft")));
    assert!(r.dependencies.iter().any(|d| d.what.contains("panel")));
    assert!(r.dependencies.iter().any(
        |d| matches!(d.basis, cctl::model::basis::BasisKind::OperatorDeclared)),
        "the PCB-to-world transform is operator-declared and must appear");
}

#[test]
fn displacing_the_control_fails_the_check() {
    let (mut cfg, _) = built();
    cfg.panel.controls[0].at[0] += 5.0; // well beyond the 0.25 mm tolerance
    let asm = cctl::assembly::build_for_test(&cfg).unwrap();
    let r = alignment::evaluate(&cfg.checks[0], &asm, &ConvexBackend).unwrap();
    assert_eq!(r.status, Status::Fail);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test --test constraints`
Expected: FAIL — constraints module missing.

- [ ] **Step 3: Write minimal implementation**

```rust
// src/constraints/mod.rs
pub mod alignment;

use crate::model::basis::BasisKind;
use serde::Serialize;

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize)]
#[serde(rename_all = "UPPERCASE")]
pub enum Status { Pass, Fail }

#[derive(Debug, Clone, Serialize)]
pub struct Dependency {
    pub what: String,
    pub value: Option<f64>,
    pub basis: BasisKind,
    pub source: String,
}

#[derive(Debug, Clone, Serialize)]
pub struct CheckResult {
    pub id: String,
    pub status: Status,
    pub required: f64,
    pub measured: f64,
    pub dependencies: Vec<Dependency>,
}
```

```rust
// src/constraints/alignment.rs
use super::{CheckResult, Dependency, Status};
use crate::assembly::Assembly;
use crate::geometry::GeometryBackend;
use crate::model::basis::BasisKind;
use crate::model::project::CheckSpec;
use anyhow::{anyhow, Result};

pub fn evaluate(spec: &CheckSpec, asm: &Assembly, g: &dyn GeometryBackend)
    -> Result<CheckResult>
{
    let a = asm.interface(&spec.a)
        .ok_or_else(|| anyhow!("unknown interface {}", spec.a))?;
    let b = asm.interface(&spec.b)
        .ok_or_else(|| anyhow!("unknown interface {}", spec.b))?;
    let measured = g.axis_offset(a, b);
    let status = if measured <= spec.max_offset { Status::Pass } else { Status::Fail };

    let mut dependencies = asm.provenance_for(&spec.a);
    dependencies.extend(asm.provenance_for(&spec.b));
    dependencies.push(Dependency {
        what: "PCB to world transform".into(),
        value: None,
        basis: BasisKind::OperatorDeclared,
        source: "enclosure.yaml".into(),
    });

    Ok(CheckResult { id: spec.id.clone(), status, required: spec.max_offset,
                     measured, dependencies })
}
```

Add `Assembly::provenance_for(&self, path: &str) -> Vec<Dependency>`, returning one `Dependency` per dimension backing that interface's primitive, carrying the dimension's `BasisKind` and a source string naming the document and page (or the rule/rationale for non-documentary bases).

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test --test constraints`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "Evaluate axis alignment constraints with dependency provenance"
```

---

### Task 13: Reporters — checks.json, text, exit code, GLB

**Files:**
- Create: `src/report/mod.rs`, `src/report/json.rs`, `src/report/text.rs`, `src/report/glb.rs`
- Modify: `src/cli.rs`
- Test: `tests/reporting.rs`

**Interfaces:**
- Consumes: `CheckResult`, `Assembly`.
- Produces:
  - `report::json::write(results: &[CheckResult], path: &Path) -> Result<()>`
  - `report::text::render(results: &[CheckResult]) -> String`
  - `report::glb::write(asm: &Assembly, path: &Path) -> Result<()>`
  - `report::exit_code(results: &[CheckResult]) -> i32` — `0` all pass, `1` any fail

- [ ] **Step 1: Write the failing test**

```rust
// tests/reporting.rs
use cctl::constraints::{CheckResult, Dependency, Status};
use cctl::model::basis::BasisKind;

fn pass() -> CheckResult {
    CheckResult { id: "rv1-shaft-to-panel-hole".into(), status: Status::Pass,
        required: 0.25, measured: 0.0,
        dependencies: vec![Dependency { what: "shaft diameter".into(), value: Some(6.0),
            basis: BasisKind::ManufacturerDrawing,
            source: "taiwan-alpha-16mm-metal-shaft-series.pdf p.43".into() }] }
}
fn fail() -> CheckResult { CheckResult { measured: 5.0, status: Status::Fail, ..pass() } }

#[test]
fn exit_code_is_zero_only_when_everything_passes() {
    assert_eq!(cctl::report::exit_code(&[pass()]), 0);
    assert_eq!(cctl::report::exit_code(&[pass(), fail()]), 1);
}

#[test]
fn json_records_dependency_provenance() {
    let td = tempfile::tempdir().unwrap();
    let p = td.path().join("checks.json");
    cctl::report::json::write(&[pass()], &p).unwrap();
    let v: serde_json::Value =
        serde_json::from_str(&std::fs::read_to_string(&p).unwrap()).unwrap();
    let dep = &v["checks"][0]["dependencies"][0];
    assert_eq!(dep["basis"], "manufacturer_drawing");
    assert!(dep["source"].as_str().unwrap().contains("p.43"));
}

#[test]
fn text_report_names_the_failing_constraint_and_its_requirement() {
    let s = cctl::report::text::render(&[fail()]);
    assert!(s.contains("FAIL"));
    assert!(s.contains("rv1-shaft-to-panel-hole"));
    assert!(s.contains("0.25"));
}

#[test]
fn glb_has_a_valid_header() {
    let td = tempfile::tempdir().unwrap();
    let p = td.path().join("assembly.glb");
    let asm = cctl::assembly::Assembly { solids: vec![
        cctl::geometry::Solid { id: "b".into(),
            kind: cctl::geometry::SolidKind::Box { size: [1.0, 2.0, 3.0] },
            origin: [0.0, 0.0, 0.0], axis: [0.0, 0.0, 1.0] }] };
    cctl::report::glb::write(&asm, &p).unwrap();
    let bytes = std::fs::read(&p).unwrap();
    assert_eq!(&bytes[0..4], b"glTF");
    assert_eq!(u32::from_le_bytes(bytes[4..8].try_into().unwrap()), 2);
    assert_eq!(u32::from_le_bytes(bytes[8..12].try_into().unwrap()) as usize, bytes.len());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test --test reporting`
Expected: FAIL — report module missing. Add `cargo add serde_json` and `cargo add serde_json --dev` as needed.

- [ ] **Step 3: Write minimal implementation**

```rust
// src/report/mod.rs
pub mod glb;
pub mod json;
pub mod text;

use crate::constraints::{CheckResult, Status};

pub fn exit_code(results: &[CheckResult]) -> i32 {
    if results.iter().any(|r| r.status == Status::Fail) { 1 } else { 0 }
}
```

```rust
// src/report/json.rs
use crate::constraints::CheckResult;
use anyhow::Result;
use std::path::Path;

pub fn write(results: &[CheckResult], path: &Path) -> Result<()> {
    let doc = serde_json::json!({
        "generator": "circuit-control",
        "checks": results,
    });
    std::fs::write(path, serde_json::to_string_pretty(&doc)?)?;
    Ok(())
}
```

```rust
// src/report/text.rs
use crate::constraints::{CheckResult, Status};

pub fn render(results: &[CheckResult]) -> String {
    let mut s = String::new();
    for r in results {
        let tag = match r.status { Status::Pass => "PASS", Status::Fail => "FAIL" };
        s.push_str(&format!("  {:<32} {:>8.2} mm  {tag}\n", r.id, r.measured));
        if r.status == Status::Fail {
            s.push_str(&format!("      required <= {:.2} mm\n", r.required));
        }
    }
    let failed = results.iter().filter(|r| r.status == Status::Fail).count();
    if failed > 0 {
        s.push_str(&format!("\nFAILED: {failed} mechanical constraint(s)\n"));
    }
    s
}
```

For `src/report/glb.rs`, write a minimal binary glTF by hand: a 12-byte header (`glTF`, version `2`, total length), a JSON chunk (padded to 4 bytes with spaces), and a BIN chunk (padded with zeros) holding interleaved vertex positions and indices. Tessellate each `Solid`: a box becomes 12 triangles, a cylinder an n-sided prism with n = 24. Keep this file under 300 lines.

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test --test reporting`
Expected: PASS (4 tests).

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "Add checks.json, text report, exit codes and GLB export"
```

---

### Task 14: Transactional sync with intent marker, and the verify verb

**Files:**
- Create: `src/sync.rs`, `src/verify.rs`
- Modify: `src/cli.rs`, `src/lib.rs`
- Test: `tests/sync.rs`

**Interfaces:**
- Consumes: everything above.
- Produces:
  - `sync::run(project_dir: &Path, mech_dir: &Path, opts: SyncOpts) -> Result<()>`
  - `struct SyncOpts { pub yes: bool }`
  - `verify::run(project_dir: &Path, mech_dir: &Path, out_dir: &Path) -> Result<Vec<CheckResult>>`

- [ ] **Step 1: Write the failing test**

```rust
// tests/sync.rs
mod fixtures;
use std::path::Path;

#[test]
fn refuses_to_write_while_kicad_holds_a_lock() {
    let (_td, dir) = fixtures::temp_project();
    std::fs::write(dir.join("~fuzz.kicad_pcb.lck"),
        r#"{"hostname":"test","username":"test"}"#).unwrap();
    let err = cctl::sync::run(&dir, Path::new("tests/data/mech"),
        cctl::sync::SyncOpts { yes: true }).unwrap_err();
    let msg = format!("{err:#}");
    assert!(msg.contains("open"), "must name the open-editor hazard: {msg}");
}

#[test]
fn a_failed_precondition_leaves_the_project_untouched() {
    let (_td, dir) = fixtures::temp_project();
    let before_sch = std::fs::read(dir.join("fuzz.kicad_sch")).unwrap();
    let before_pcb = std::fs::read(dir.join("fuzz.kicad_pcb")).unwrap();
    // A mech dir that does not exist fails validation in stage 1.
    let _ = cctl::sync::run(&dir, Path::new("tests/data/does-not-exist"),
        cctl::sync::SyncOpts { yes: true });
    assert_eq!(std::fs::read(dir.join("fuzz.kicad_sch")).unwrap(), before_sch);
    assert_eq!(std::fs::read(dir.join("fuzz.kicad_pcb")).unwrap(), before_pcb);
}

#[test]
fn a_stale_intent_marker_is_detected_and_reported() {
    let (_td, dir) = fixtures::temp_project();
    std::fs::write(dir.join(".cctl-sync-intent"),
        "fuzz.kicad_sch\nfuzz.kicad_pcb\n").unwrap();
    let err = cctl::sync::run(&dir, Path::new("tests/data/mech"),
        cctl::sync::SyncOpts { yes: true }).unwrap_err();
    assert!(format!("{err:#}").contains("did not complete"));
}

#[test]
fn verify_never_mutates_the_project() {
    let (_td, dir) = fixtures::temp_project();
    cctl::sync::run(&dir, Path::new("tests/data/mech"),
        cctl::sync::SyncOpts { yes: true }).unwrap();
    let sch = std::fs::read(dir.join("fuzz.kicad_sch")).unwrap();
    let pcb = std::fs::read(dir.join("fuzz.kicad_pcb")).unwrap();

    let out = dir.join("out");
    std::fs::create_dir_all(&out).unwrap();
    for _ in 0..2 {
        cctl::verify::run(&dir, Path::new("tests/data/mech"), &out).unwrap();
        assert_eq!(std::fs::read(dir.join("fuzz.kicad_sch")).unwrap(), sch);
        assert_eq!(std::fs::read(dir.join("fuzz.kicad_pcb")).unwrap(), pcb);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test --test sync`
Expected: FAIL — sync module missing.

- [ ] **Step 3: Write minimal implementation**

Implement `sync::run` in the ordered stages from spec Section 9:

1. Validate inputs: load `ProjectConfig`, load every bound `Part`, run `check_part` on each. Any failure returns before touching the project.
2. Check preconditions: refuse if any `~*.lck` exists (name the file and say KiCad has it open); refuse if `.cctl-sync-intent` exists (a previous sync did not complete — list the files it names); warn on dirty target files unless `opts.yes`.
3. Compute the full mutation set in memory: the `.kicad_mod` text, the schematic edit, the board edit.
4. Write every output to `<target>.cctl-tmp` alongside its target.
5. Validate the temporaries with `kicad-cli` where a verb exists for them.
6. Write `.cctl-sync-intent` listing every target, then rename each temporary over its target, then delete the marker.
7. Any failure before stage 6 returns with the project unmodified.

Skeleton for the stages that need exact behaviour:

```rust
// src/sync.rs (excerpt)
pub struct SyncOpts { pub yes: bool }

const INTENT: &str = ".cctl-sync-intent";

fn preconditions(project: &Path, opts: &SyncOpts) -> Result<()> {
    for e in std::fs::read_dir(project)? {
        let p = e?.path();
        let name = p.file_name().and_then(|s| s.to_str()).unwrap_or("");
        if name.starts_with('~') && name.ends_with(".lck") {
            bail!("KiCad has {} open. Close it, or your changes will be overwritten.",
                  name.trim_start_matches('~').trim_end_matches(".lck"));
        }
    }
    let marker = project.join(INTENT);
    if marker.exists() {
        let listed = std::fs::read_to_string(&marker)?;
        bail!("a previous sync did not complete; these targets were being replaced:\n{listed}\
               \nInspect with git, then remove {INTENT} to proceed.");
    }
    let _ = opts.yes; // dirty-file warning is wired in the CLI layer
    Ok(())
}

/// Stage 6: write the marker, rename each temporary over its target, clear it.
fn commit(project: &Path, staged: &[(PathBuf, PathBuf)]) -> Result<()> {
    let marker = project.join(INTENT);
    let listing = staged.iter()
        .map(|(_, dst)| dst.display().to_string())
        .collect::<Vec<_>>().join("\n");
    std::fs::write(&marker, format!("{listing}\n"))?;
    for (tmp, dst) in staged {
        std::fs::rename(tmp, dst)?;
    }
    std::fs::remove_file(&marker)?;
    Ok(())
}
```

`verify::run` loads config and parts, reads the board, builds the assembly, evaluates every check, writes `out/checks.json` and `out/assembly.glb`, and returns the results:

```rust
// src/verify.rs
use crate::constraints::{alignment, CheckResult};
use crate::geometry::convex::ConvexBackend;
use crate::model::project::ProjectConfig;
use anyhow::Result;
use std::path::Path;

/// Never writes inside `project`. Outputs go to `out_dir` only.
pub fn run(project: &Path, mech: &Path, out_dir: &Path) -> Result<Vec<CheckResult>> {
    let cfg = ProjectConfig::load(mech)?;
    let asm = crate::assembly::build_from_board(project, &cfg)?;
    let mut results = Vec::new();
    for spec in &cfg.checks {
        results.push(alignment::evaluate(spec, &asm, &ConvexBackend)?);
    }
    std::fs::create_dir_all(out_dir)?;
    crate::report::json::write(&results, &out_dir.join("checks.json"))?;
    crate::report::glb::write(&asm, &out_dir.join("assembly.glb"))?;
    Ok(results)
}
```

Add `assembly::build_from_board(project: &Path, cfg: &ProjectConfig) -> Result<Assembly>`, which differs from `build_for_test` in one way that matters: it reads each class-A footprint's **actual** `(at ...)` out of `fuzz.kicad_pcb` rather than deriving it. That is what makes drift detectable — `verify` compares reality against the derived expectation instead of recomputing the expectation and agreeing with itself.

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test --test sync`
Expected: PASS (4 tests).

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "Add transactional sync with intent marker and non-mutating verify"
```

---

### Task 15: M1 acceptance — prove the negative path

This is the task that makes M1 mean something. A pipeline that only produces agreeable answers has not demonstrated the product thesis.

**Files:**
- Create: `tests/m1_acceptance.rs`
- Modify: `src/cli.rs` (wire `sync`, `verify`, `explain`)
- Test: itself

**Interfaces:**
- Consumes: everything.
- Produces: `cctl sync`, `cctl verify`, `cctl explain <check-id>` as working verbs.

- [ ] **Step 1: Write the failing test**

```rust
// tests/m1_acceptance.rs
mod fixtures;
use std::path::Path;
use std::process::Command;

fn run(dir: &Path, args: &[&str]) -> (i32, String) {
    let out = Command::new(env!("CARGO_BIN_EXE_cctl"))
        .args(args)
        .arg("--project").arg(dir)
        .arg("--mech").arg("tests/data/mech")
        .output().expect("cctl runs");
    (out.status.code().unwrap_or(-1),
     format!("{}{}", String::from_utf8_lossy(&out.stdout),
                     String::from_utf8_lossy(&out.stderr)))
}

#[test]
fn source_to_pass_to_drift_to_fail_to_restore_to_pass() {
    let (_td, dir) = fixtures::temp_project();

    // 1. Materialise generated state.
    let (code, out) = run(&dir, &["sync"]);
    assert_eq!(code, 0, "sync must succeed:\n{out}");

    // 2. Verification passes on the synced project.
    let (code, out) = run(&dir, &["verify"]);
    assert_eq!(code, 0, "verify must pass after sync:\n{out}");
    assert!(out.contains("PASS"), "{out}");

    // 3. Introduce drift: move RV1 far beyond the 0.25 mm tolerance.
    let pcb = dir.join("fuzz.kicad_pcb");
    let text = std::fs::read_to_string(&pcb).unwrap();
    let drifted = shift_rv1_x(&text, 5.0);
    assert_ne!(drifted, text, "drift must actually change the board");
    std::fs::write(&pcb, &drifted).unwrap();

    // 4. Verification fails, with a non-zero exit.
    let (code, out) = run(&dir, &["verify"]);
    assert_eq!(code, 1, "drift must fail verification:\n{out}");
    assert!(out.contains("FAIL"), "{out}");

    // 5. The failure explains what it depends on.
    let (code, out) = run(&dir, &["explain", "rv1-shaft-to-panel-hole"]);
    assert_eq!(code, 0);
    assert!(out.contains("manufacturer_drawing"), "must name the evidence:\n{out}");
    assert!(out.contains("operator_declared"), "must name the declared transform:\n{out}");
    assert!(out.contains("fuzz.kicad_pcb"), "must name the changed input:\n{out}");

    // 6. Restore and pass again.
    std::fs::write(&pcb, &text).unwrap();
    let (code, out) = run(&dir, &["verify"]);
    assert_eq!(code, 0, "restoring must pass:\n{out}");
}

/// Shift RV1's placement X by `dx` mm, editing only the (at ...) inside its
/// footprint block.
fn shift_rv1_x(src: &str, dx: f64) -> String {
    let anchor = src.find("\"RV1\"").expect("RV1 present");
    let block_start = src[..anchor].rfind("(footprint").expect("footprint block");
    let at_rel = src[block_start..].find("(at ").expect("at node");
    let at_abs = block_start + at_rel + 4;
    let end = at_abs + src[at_abs..].find(')').expect("close paren");
    let parts: Vec<&str> = src[at_abs..end].split_whitespace().collect();
    let x: f64 = parts[0].parse().expect("x");
    let rest = parts[1..].join(" ");
    format!("{}{} {}{}", &src[..at_abs], x + dx, rest, &src[end..])
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test --test m1_acceptance`
Expected: FAIL — `sync`, `verify` and `explain` are not wired into the CLI yet.

- [ ] **Step 3: Wire the verbs**

Extend `src/cli.rs` with `Sync`, `Verify` and `Explain { id }` variants, each taking `--project <dir>` and `--mech <dir>`. `Verify` prints `report::text::render`, writes `out/checks.json` and `out/assembly.glb`, and returns `report::exit_code`. `Explain` reads `out/checks.json`, finds the named check, and prints its full dependency list — one line per dependency showing what, value, basis type and source.

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test`
Expected: PASS, all suites. This is M1 complete.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "Wire sync, verify and explain; prove the M1 negative path end to end"
```

---

## Definition of Done

M1 is complete when `cargo test` passes and the acceptance test in Task 15 demonstrates, on a temp copy of the real fuzz project:

```
source -> model -> generated ECAD -> assembly -> PASS
                                        |
                                  introduce drift
                                        |
                    FAIL + non-zero exit + dependency provenance
                                        |
                                     restore
                                        |
                                      PASS
```

Plus these invariants, each covered by a test:

- `fuzz.kicad_sch` and `fuzz.kicad_pcb` round-trip byte-for-byte with no mutation requested.
- `verify` run twice leaves the project byte-identical.
- A failed precondition leaves the project unmodified.
- An open KiCad lock refuses the write by name.
- A documentary basis missing its page fails `catalog check`.
- A silently replaced document fails its hash check.
- KiCad's own `sch export bom` and `pcb export pos` agree with what we wrote.

## Out of scope for M1

Deliberately deferred, with the milestone that owns each: remaining 75 footprints and the board outline (M2); **catalog resolution over ordered roots** — M1 loads parts by direct path from the repo `catalog/`, which is the bundled root only (M2); content hashing of the whole package and lock enforcement (M2); sourcing verbs and `cctl bom` (M2.5); class-C parts, envelopes and clearance checks (M3); catalog update and freeze (M4); the GLB viewer page and CI wiring (M5); vendor model cross-check — `VendorModel` is hashed from M1 but nothing compares against it (M6).

Two spec features appear in M1 as **schema only**, carrying no behaviour: `physical_validation.status` and `models:`. Both are recorded and hash-verified so the part package is right from the first commit, but neither is consumed until later milestones. That is deliberate — adding them later would mean a schema migration across every authored part.
