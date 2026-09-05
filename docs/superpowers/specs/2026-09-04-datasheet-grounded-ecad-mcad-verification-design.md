---
title: Datasheet-Grounded ECAD/MCAD Verification — Design
status: Draft (awaiting review)
date: 2026-09-04
prd: PRD — Datasheet-Grounded ECAD↔MCAD Verification (2026-09-04)
---

# Datasheet-Grounded ECAD/MCAD Verification — Design

Design for `circuit-control`, derived from the PRD of the same date. Section
references in the form `§n` refer to the PRD.

## 1. What this is

A tool that makes the mechanical consequences of a PCB design testable from
source.

Three sources of truth feed it: KiCad owns the board, manufacturer datasheets
own component dimensions, and hand-written mechanical definitions own the
enclosure. From those it generates a populated-PCB assembly plus surrounding
structure, evaluates numeric constraints between named mechanical interfaces,
and exits non-zero when a required constraint fails.

The core invariant, quoted from §23:

> No mechanically significant relationship that can be derived from an
> authoritative source should require manual synchronization between ECAD and
> MCAD.

The 3D model is evidence. **The verification contract is the product.**

## 2. What the user is buying

The operator's stated pain, in their words, is managing component libraries,
searching for footprints, hunting for parts, and lining up components on a PCB
against real parts in the real world. They are happy to do PCB layout and
routing.

This sets the product boundary precisely:

- The computer owns **every footprint**, the placement of parts whose position
  is physically determined by the enclosure, and **finding parts and their
  datasheets** (Section 16).
- The human owns **layout, routing, and circuit design**.

Design decisions below are resolved in favour of that split wherever the PRD
leaves room.

## 3. Acceptance fixture

The fixture is the fuzz pedal at `~/work/pedals/fuzz`, schematic revision `v1`.
Verified state as of this writing:

| Fact | Value |
|---|---|
| Schematic | `schematics/v1/fuzz/fuzz.kicad_sch`, 148 KB, 76 symbols |
| Footprints assigned | **0** — all 76 `Footprint` properties are empty strings |
| Board | `fuzz.kicad_pcb`, 79 bytes — empty, no footprints, no outline |
| KiCad | 10.0.5, board file format `20260206` |
| Enclosure | Hammond 1590BBS |

Part inventory from the schematic, which the operator has designated the source
of truth for the circuit (`DESIGN-RECOMMENDATIONS.md` is stale and must not be
used as a requirements input):

- 23 × `Device:R`
- 14 × `Device:C`, 1 × `Device:C_Polarized`
- 6 × `Device:D`, 1 × `Device:D_Schottky`
- 4 × `Simulation_SPICE:NPN` (Q1–Q4, value `BC548`)
- 1 × `Device:R_Potentiometer` (RV1, `A500K`)
- 1 × `Switch:SW_SPDT` (SW1, `STAGE SELECTOR`)

Reference designators skip R17, R24, C3, C6, C13 — deleted during design, not
an error.

**Q1–Q4 use a simulation symbol.** `Simulation_SPICE:NPN` declares pins
`1=C, 2=B, 3=E`, which matches BC548 TO-92 datasheet numbering, so binding a
TO-92 footprint is correct. But the symbol carries no MPN or footprint filter,
so KiCad's own assignment tooling offers no help. The binding is explicit in
project configuration and is covered by a test, because a silent error here
produces physically backwards transistors.

**Datasheets already on hand** in `~/work/pedals/assets`: Alpha RV16AF 16 mm
potentiometer, Taiway 100DP1T8B13M2QEH toggle, Neutrik NMJ6HCD2 TRS jack,
KC-301339 DC power jack.

**Enclosure**, from Hammond's official drawing (REV 25.05.2020), vendored at
`catalog/mechanical/hammond-1590bbs/`, all mm:

| | |
|---|---|
| External | 119.50 × 94.00 × 42.10 |
| **Inside** | **113.48 L × 87.98 W × 37.85 H** |
| Wall thickness | 2.25 |
| Lid | 2.00 thick, 4 × Ø4.00 holes |
| Fixings | 4 × 6-32 UNC, 11.00 deep |
| Screw centres | 109.50 × 84.00 |

**Footswitch**, sourced and selected by the operator: StompBox Parts 3PDT
Footswitch PRO, SKU FSW-1032, solder lug, therefore **class C**. Dimensions
from the vendored drawing at `catalog/switches/3pdt-stomp/`; panel hole Ø12.5,
M12×0.75 bushing 11.5 long, and **18.7 mm of interior depth consumed** behind
the panel.

**Assembly geometry is operator-specified, not inferred.** How the PCB sits in
the enclosure — its plane, offset, and mounting method — is declared in
`enclosure.yaml` by the operator. The tool consumes that transform and checks
its consequences; it does not derive the arrangement. Dimensions above are
sourced facts from vendored drawings; conclusions drawn from combining them are
verification output, not spec content.

## 4. Part classes

The fixture forces a three-way split that the PRD does not name explicitly.

| Class | Members in fixture | Footprint | Placement | Verified by |
|---|---|---|---|---|
| **A — panel-mounted, on board** | RV1, SW1 | generated from datasheet | **computed from enclosure** | alignment |
| **B — free, on board** | R, C, D, Q (74 symbols) | generated from family | **operator's** | clearance |
| **C — enclosure-only** | TRS jacks, DC jack, footswitch, LED | none — not on the board | declared in enclosure | obstacles the PCB must clear |

Class C exists because pedal jacks and footswitches are hand-wired and never
appear in the netlist, yet they occupy significant interior volume in a 1590BBS
and are among the most likely things for a board to collide with. Goal G5
covers them as non-PCB component placement.

## 5. Placement runs backwards from the enclosure

The central inversion of this design. Goal G1 assumes KiCad is authoritative
for placement; for class A that is false, because a hole in a physical box
determines where the part must be.

```
1590BBS interior  →  panel hole for VOL  →  pot shaft axis
                                                  │
                          PCB mounting position    │
                                       └───────────┤
                                                   ▼
                                        pot footprint origin
                                                   │
                                                   ▼
                                        where RV1's pins land
```

The operator expresses position in **panel language** — the language they would
use with a drill — and the tool solves for the board coordinates.

Class A placement is therefore a *derived* quantity, and G1 is honoured for
classes B and C only. This is a deliberate, documented departure from the PRD.

### Derived state is enforced, not merely produced

Class-A placement now exists in two writable places: the mechanical definition
that derives it, and the KiCad board, where the operator can drag the footprint.
Two writable representations of one derived value require an explicit contract:

> **Class-A placement, generated footprint geometry, and the generated board
> outline are governed state.** `cctl sync` materialises them from the
> mechanical definition. `cctl verify` fails when the checked-in KiCad state
> differs from the derived expectation beyond tolerance.

And the invariant that makes this usable in CI:

> **`cctl verify` never repairs or synchronises project state.** It evaluates
> the checked-in state against derived expectations and reports. Mutation is
> `sync`'s job alone.

`sync` materialises and repairs; `verify` detects. Verification must never
require mutation first, or CI cannot distinguish "correct" from "just fixed".

### Board outline ownership

Enclosure-derived outlines are the fixture's case, not an architectural
invariant — real boards need notches, mounting-hole avoidance, and irregular
shapes. Ownership is declared by strategy:

```yaml
board:
  outline:
    strategy: enclosure_inset
    clearance: 2.0
```

V1 implements `enclosure_inset` only. `explicit` (operator draws it, tool
verifies fit) and `polygon` can be added later without changing ownership
semantics, because the strategy names who owns the outline rather than assuming
the tool does.

## 6. Repository layout and ownership

`circuit-control` is the tool and stays project-agnostic. It also carries the
catalog, because §7 requires component definitions to be reusable across
projects and §20 lists library sharing as a goal.

```
circuit-control/
├── catalog/                          canonical, versioned with the tool
│   ├── potentiometers/alpha-rv16af/  part.yaml + datasheet PDF
│   ├── switches/taiway-100dp1t8.../  part.yaml + datasheet PDF
│   ├── connectors/neutrik-nmj6hcd2/
│   ├── connectors/kc-301339/
│   ├── semiconductors/bc548-to92/
│   ├── passives/                     generic families
│   └── mechanical/hammond-1590bbs/
└── docs/

~/work/pedals/fuzz/
├── schematics/v1/fuzz/               KiCad — see the ownership rule below
└── mech/
    ├── parts.yaml                    project parts manifest (see below)
    ├── enclosure.yaml                1590BBS + class-C placement
    ├── panel.yaml                    control positions in panel coordinates
    ├── checks.yaml                   constraints that must hold
    ├── parts/                        project-local one-offs (shadow catalog)
    └── circuit-control.lock
```

Category names follow §7's tree verbatim.

**Datasheet PDFs are vendored into the catalog** beside their `part.yaml`. §8
requires "why does the model believe this dimension is 6.0 mm" to be
answerable; if the YAML ships in `circuit-control` and the PDF lives in a
different repository, provenance breaks for anyone else cloning the catalog.
Total cost is a few megabytes.

### Catalog resolution over ordered roots

Adding a part must never require a commit to the tool repository, or library
management returns as friction. But a project-local shadow that promotes into
the tool's source tree is an odd lifecycle: the catalog is *data*, not tool
source, and this operator intends to build many kinds of electronics projects.

So the bundled catalog is **one root among several, not a privileged
singleton**. Resolution walks ordered roots with deterministic precedence:

```yaml
catalogs:
  - ./mech/parts          # project-local
  - ~/work/hardware-catalog   # personal or team, optional
  - builtin               # bundled with circuit-control
```

V1 ships the project root and the bundled root. A shared root needs no new
architecture, only configuration, and the catalog can later move out of this
repository without a redesign.

### The project parts manifest

`mech/parts.yaml` is the union of everything the project is built from, and is
deliberately **not** derived from the schematic.

A hardware project's real bill of materials is larger than its netlist. KiCad
knows about RV1 and Q1–Q4; it will never know about the enclosure, the
footswitch, the jacks, knobs, the LED bezel, standoffs, or hardware. Those are
class-C parts and pure mechanical items, and they are the ones the operator
must actually order.

**Mechanical package and purchasing identity are separate bindings.** The 23
resistors share one mechanical package but are electrically distinct — 10K,
100R, 8.2K, 470K, 100K, 1M. `Device:C` is likewise insufficient to determine
ceramic disc, film, or radial electrolytic, and those differ in height, which
is exactly what clearance checking depends on. Package classification is a real
decision and must be represented rather than assumed.

```yaml
bindings:
  - refs: [R1, R2, R3, R4, R5, R6, R7, R8]
    package: passives/resistor-axial-din0207-p10.16
  - refs: [Q1, Q2, Q3, Q4]
    package: semiconductors/to92-inline-p2.54
    part: semiconductors/bc548          # optional purchasing identity
  - ref: RV1
    package: potentiometers/alpha-rv16af-vertical

hardware:
  - id: enclosure
    part: mechanical/hammond-1590bbs
  - id: footswitch
    part: switches/3pdt-stomp
    qty: 1
```

**Electrical values are never restated here.** The schematic owns them, and
duplicating `10K` or `47uF` into the manifest would create a second writable
copy of ECAD-owned data — precisely the manual synchronisation §23 forbids.
`cctl bom` joins against the schematic's values at build time; any
part-number mapping is keyed on `(symbol, value)` rather than repeating the
value.

The manifest owns **what**; `enclosure.yaml` owns **where**, referring to
`hardware` entries by id. This preserves the ownership split in PRD §6.

Three things fall out:

- **`cctl bom`** emits a complete project bill of materials, covering
  non-netlist items no KiCad BOM export can produce. It claims **orderability
  only for entries carrying sourcing identity**; everything else is listed as
  mechanical description. This keeps a mechanical verification system from
  quietly becoming an electrical component-management system.
- **Reconciliation** — every schematic reference must appear under `bindings`,
  and every binding must correspond to a real symbol. Unbound references are an
  error, which states the fixture's "76 unassigned footprints" problem as a
  check.
- **Package assignment is explicit**, so a decision like "these caps are radial
  electrolytic and 11 mm tall" is recorded and reviewable rather than implied
  by a symbol name.

**A package definition must completely determine generated copper.** `to92`
does not: it fixes neither lead forming nor pad arrangement, and BC548 ships
with different lead-forming assumptions. Since this system *generates*
footprints and Section 3 treats pin ordering as safety-critical, an
under-specified package is a silent hazard.

The fix is enforcement rather than nomenclature — an under-specified
`land_pattern:` would be exactly as broken as an under-specified `package:`.
So `cctl catalog check` **rejects any package definition that does not fully
determine pad geometry**: `semiconductors/to92` fails, `semiconductors/to92-inline-p2.54`
passes. Package ids are expected to carry the distinguishing detail in their
name for the same reason.

**Ownership rule.** The tool writes exactly four things into a KiCad project:

1. `Footprint` properties on schematic symbols (all classes);
2. a generated `.pretty` footprint library;
3. X/Y/rotation of **class A footprints only**;
4. the board outline.

Everything else in those files belongs to the operator and is not modified.

## 7. Architecture

```
  fuzz.kicad_sch ──┐                          ┌──> fuzz.kicad_sch  (footprint fields)
  fuzz.kicad_pcb ──┤                          ├──> fuzz.pretty/    (.kicad_mod)
                   │      ┌────────────┐      ├──> fuzz.kicad_pcb  (class-A placement,
   catalog/ ───────┼─────>│  KiCad I/O │─────>┤                     board outline)
   mech/*.yaml ────┘      └─────┬──────┘      │
                                │             ├──> out/assembly.glb
                          ┌─────▼──────┐      ├──> out/checks.json
                          │  assembly  │─────>┤
                          │   model    │      └──> stdout report
                          └─────┬──────┘
                                │
                          ┌─────▼──────┐
                          │ constraints│
                          └────────────┘
```

Components, each independently testable:

- **KiCad I/O** — reads and writes KiCad documents. Behind a trait (Section 8).
- **Catalog** — YAML to part model: pads, geometry, envelopes, keepouts,
  interfaces, provenance. Resolves over ordered catalog roots.
- **Footprint emitter** — part model to `.kicad_mod`.
- **Geometry** — primitives, transforms, coordinate frames, tessellation.
  Behind a trait (Section 8).
- **Placement** — panel coordinates to class-A footprint transforms.
- **Constraints** — alignment, clearance, collision, containment over named
  interfaces.
- **Reporters** — GLB, `checks.json`, text report.

### Implementation language

Rust, per §12 and §13. A single binary with no runtime is the most direct
satisfaction of the operator's requirement to clone a repository and start
running. `rust-toolchain.toml` pins the toolchain; `Cargo.lock` pins the
dependency graph. **No Python at any layer**, including the geometry backend —
this is a hard constraint from the operator, and §13's "the user must never be
expected to manually create Python virtual environments" points the same way.

The 3D viewer is a static HTML page consuming the emitted GLB. No build step,
no second toolchain.

### CLI

Binary name `cctl`. (`cc` was used during design discussion and is rejected —
it collides with the system C compiler.) §12 leaves naming open.

```
cctl sync              assign footprints, emit library, place class A, outline
cctl build             read board, construct assembly, emit GLB
cctl verify            evaluate constraints; non-zero exit on required failure
cctl inspect <part>    dump a part model with provenance
cctl bom               emit an orderable bill of materials
cctl catalog new       scaffold a part definition
cctl catalog datasheet vendor a datasheet PDF into the catalog
cctl catalog check     validate a part; fails on missing or malformed basis
cctl catalog measure   record a physical cross-check against a specimen
cctl catalog update    upgrade a pinned part, with dimensional diff
cctl explain <check>   full dependency provenance for one constraint result
cctl freeze            tag the lock against a fabrication release
```

## 8. Two backend seams

§14 asks for one swappable boundary, around geometry. This design has two,
because the KiCad side has an equally clear and better-dated migration ahead of
it.

```
                        ┌──────────────────────┐
                        │  product model       │
                        │  parts, interfaces,  │
                        │  constraints, checks │
                        └──┬────────────────┬──┘
                           │                │
                  KiCadBackend      GeometryBackend
                           │                │
          ┌────────────────┴───┐    ┌───────┴────────────┐
   now    │ s-expression files │    │ convex primitives  │
          ├────────────────────┤    ├────────────────────┤
  later   │ kicad-cli          │    │ OCCT               │
          │ api-server (K11)   │    │ (extrusions, STEP) │
          └────────────────────┘    └────────────────────┘
```

### GeometryBackend

V1 needs no BREP kernel. §19 scopes geometry to "boxes and cylinders at
minimum"; §11 requires only an inspectable 3D assembly, machine-readable
results, and a human-readable summary, with STEP "preferred where supported";
§20 defers STEP import and richer STEP export.

Boxes and cylinders are convex. Distance and interference between convex
primitives is GJK/EPA plus linear algebra. Tessellation to GLB is direct.
Crucially, §7 and §10 express constraints between **named interfaces** rather
than by boolean geometry — "is this shaft concentric with that opening" is a
dot product, not a CSG operation — so the hard non-convex cases do not arise in
v1. Non-convex bodies, where needed, decompose into convex parts.

Consequence: the operator's `164_1590BBS.stp` is **unused in v1**. The
enclosure is modelled from Hammond's published dimensional drawing instead,
which NG5 and §15 both prefer, and the STEP becomes a future cross-check.

OCCT becomes necessary for real extrusions, STEP export, and imported-versus-
datasheet comparison — all §20 deferrals. Because no Python is permitted, that
future backend must bind OCCT natively rather than via build123d or CadQuery.

### KiCadBackend

Verified against KiCad documentation, not merely the local install:

- The **IPC API** (Protocol Buffers over nng) in KiCad 9 and 10 communicates
  only with a **running pcbnew GUI**. CI use today requires xvfb.
- **SWIG `pcbnew`** is deprecated in 9, present in 10, and **removed in 11**.
- **KiCad 11 adds `kicad-cli api-server`**, a headless IPC server. That is the
  supported, stable, language-agnostic path, with a documented compatibility
  promise.
- **`kicad-cli` today** can export, render, run DRC, and upgrade. It **cannot
  edit**.

Therefore the only headless write path in KiCad 10 is the s-expression file
format, and the v1 backend hand-rolls it. Because the IPC replacement is on a
published roadmap rather than hypothetical, the s-expression implementation is
scoped as a **bridge**: it must handle the fixture's documents correctly, not
every KiCad file ever written. Investment is bounded accordingly.

**Rust clients for the IPC API already exist.** The `.proto` definitions live
in the KiCad repository as a submodule; `gitlab.com/kicad/code/kicad-rs` is an
official KiCad-org Rust binding, and `kicad-ipc-rs` is an MIT-licensed
alternative shipping checked-in generated protobuf so consumers need no KiCad
checkout. The migration target is therefore concrete rather than speculative,
which is what justifies bounding investment in the bridge.

### The lossless-write invariant

A generic s-expression parser and re-emitter can produce enormous diffs through
whitespace, quoting, numeric formatting, node ordering, or version
normalisation. Claiming the operator's data is preserved requires more than a
fixture test:

> **Any KiCad document accepted by the backend must round-trip byte-for-byte
> when no supported mutation is requested.** Constructs the backend does not
> understand are preserved opaquely, or the backend refuses the write.

That is a backend contract, not a test case. It is what makes the ownership
rule in Section 6 defensible rather than aspirational.

The backend also records `circuit-control` in the `(generator ...)` field.
KiCad's formats carry that field, and third-party writers are expected to
identify themselves rather than impersonate KiCad's own generators.

## 9. KiCad write strategy

The tool writes `.kicad_sch` and `.kicad_pcb` **directly**. No patch-proposal
step, no shadow copies, no region fencing. Git is the journal and the rollback
mechanism, consistent with the operator's standing practice.

Implementation rule: parse to a tree, mutate named nodes, re-emit. Never
regenerate a document wholesale.

Two hazards git cannot cover, which the tool handles:

- **Open-editor race.** KiCad holds `~*.lck` files while a document is open. A
  write underneath a running KiCad is silently discarded on its next save, and
  nothing records that the write happened. `cctl sync` detects the lock and
  refuses with a message naming the file.
- **Uncommitted work in target files.** Git can only restore what is committed.
  If a file the tool is about to rewrite is dirty, warn and name it; `--yes`
  proceeds. Scoped to the files at risk, not the whole tree.

### sync is transactional

`sync` performs several dependent mutations — schematic footprint fields, the
generated library, class-A placement, the board outline, the lock. Git provides
rollback, but partially applied state is still a poor runtime contract: a
failure after step two leaves a project nobody designed.

Ordered contract:

1. Read and validate all inputs.
2. Compute the complete desired mutation set in memory.
3. Evaluate all preconditions — locks, dirty files, stale lockfile.
4. Write every output to temporary files.
5. Validate the generated KiCad artifacts with `kicad-cli` against the
   temporaries.
6. Replace targets by rename.
7. Fail without modifying the project if anything before step 6 fails.

Stated precisely, because overclaiming here would repeat the error the
lossless-write invariant exists to prevent: **each individual file is replaced
atomically by rename; multi-file replacement is validated-before-any-write and
ordered, but is not atomic across files.** POSIX offers no cross-file
atomicity.

Backup-and-restore rollback across the rename phase is deliberately **not**
implemented. Git is the journal, the exposed window is a few renames wide, and
that machinery earns its place only if this actually bites.

But "recoverable via git" is only true if you know to look. So `sync` writes an
**intent marker** naming the full mutation set before the first rename and
clears it after the last. A marker found on a later invocation means a previous
`sync` did not complete: the tool reports exactly which files were and were not
replaced, and points at git. A partial commit is thereby detectable rather than
merely survivable.

## 10. Coordinate frames

Per §9, frames must be documented and tested.

```
world  (1590BBS interior floor, back-left corner, Y-up, right-handed, mm)
 ├── enclosure ── panel ── opening/interface
 ├── PCB ── footprint ── component ── interface
 └── artwork                                    (deferred, §20)
```

The PCB's relationship to world is one explicit transform **declared by the
operator** in `enclosure.yaml`. The tool does not infer how the board sits in
the enclosure; it consumes the declared transform and verifies its
consequences.

**KiCad board space is millimetres with Y increasing downward.** Every
placement crosses that handedness flip. A sign error here yields a model that
looks plausible and is mirrored, so the flip is isolated in one function with
dedicated tests, including a deliberately asymmetric fixture that a mirrored
transform cannot satisfy.

## 11. Verification

Per §10, initial constraint kinds: alignment, clearance, collision,
containment, mounting, PCB fit.

Results identify the constraint, participating objects, required value,
measured value, PASS/FAIL, and diagnostics. `cctl verify` exits non-zero when a
required constraint fails, satisfying §17 and acceptance criterion 10.

```
cctl verify

Panel
  VOL  shaft ↔ 1590BBS hole ......... 0.00 mm  PASS
  SW1  toggle ↔ hole ................ 0.00 mm  PASS
Clearance
  C10 (47uF) ↔ lid .................. 0.41 mm  FAIL
      required >= 1.00 mm

FAILED: 1 mechanical constraint
```

### Results carry their dependency provenance

A result that reports only required and measured values asks to be trusted.
Since the tool holds the whole chain, `checks.json` records **what each number
depends on**, by identity:

```
constraint: C10 ↔ lid clearance          FAIL
 ├─ required        1.00 mm   design_rule            default_component_clearance
 ├─ measured        0.41 mm
 └─ dependencies
     ├─ C10 body height    11.0 mm   manufacturer_datasheet  doc hash, p.3
     ├─ wire allowance     10.0 mm   assembly_allowance      project rule
     ├─ enclosure inside   37.85 mm  manufacturer_drawing    doc hash, p.1
     ├─ PCB placement                fuzz.kicad_pcb @ rev
     └─ PCB→world                    operator_declared       enclosure.yaml
```

Every number carries its basis type, so the report distinguishes a
manufacturer-specified dimension from a project assembly allowance from an
operator-declared transform at a glance. That vocabulary is what makes
`cctl explain` useful rather than merely verbose.

Terminal output stays terse; the dependency identities live in `checks.json`,
and `cctl explain <check>` expands one result into the full chain.

This is what makes §23's claim — *the verification contract is the product* —
concrete rather than rhetorical. A PASS that cannot say why it believes its
inputs is not a verification, and a FAIL whose provenance is opaque cannot be
debugged.

## 12. Catalog versioning

The catalog accretes, so identity must be right early.

**Change severity is computed from a declared field-to-effect table.** An
earlier draft claimed the footprint emitter and geometry core consume disjoint
fields, so severity fell out of the data model for free. That was wrong: pad
positions are mechanically meaningful, courtyards are mechanical, and mounting
holes plainly span both domains. Fields are classified by **effect**, in a
table that is maintained deliberately and reviewed when the schema changes:

| Severity | Fields | Why it matters |
|---|---|---|
| **fab-affecting** | anything reaching copper: pad position, pitch, drill, courtyard | Physically committed once routed or fabricated; must never move silently. Note these are *also* mechanical inputs — the classes are effects, not partitions. |
| **mechanical-only** | body height, shaft diameter, envelope, keepout | Feeds geometry and checks alone. May change; consequences surface as PASS/FAIL. |
| **breaking** | interface renamed or removed | `checks.yaml` references interfaces by name. Hard error naming every dangling check. |

A field may carry both fab-affecting and mechanical effects; severity is the
strongest effect it has.

**Pinning uses a package hash, not a file hash.** Hashing `part.yaml` alone
leaves a hole straight through the provenance premise: the same YAML with a
silently replaced PDF would produce the same lock entry. The lock therefore
records a hash over the whole **mechanically meaningful package**:

- `part.yaml`
- every referenced authoritative document, by content hash
- any declared auxiliary geometry
- the schema/generator version

A deterministic hash over the manifest plus referenced file hashes is
sufficient; a full Merkle structure is not required. Any change within that
package makes the lock stale, and nothing recomputes silently. A human-facing
`version:` field exists in `part.yaml` for communication, but the lock keys on
the hash so no one's diligence is load-bearing.

One deliberate consequence: re-downloading a byte-different but
content-equivalent PDF invalidates the lock. That is the correct default for a
system whose premise is documentary authority, and `catalog update` shows
exactly what changed.

Upgrades are explicit, and the diff is the deliverable:

```
cctl catalog update alpha-rv16af

  shaft_diameter    6.00 → 6.35 mm     mechanical-only
      provenance:   p.4 → p.6 (datasheet rev B)
  pad pitch         unchanged
  interfaces        unchanged

  re-checking fuzz…
  VOL shaft ↔ 1590BBS hole   PASS → FAIL
      hole 6.10 mm, shaft 6.35 mm
```

Semantic version numbers cannot express that; the tool can, because it holds
both definitions and the constraint set.

### Release identity

Because the manifest deliberately joins against schematic-owned values rather
than restating them, **the lock alone cannot reproduce a fabrication state.**
The lock pins mechanical catalog inputs; git pins the project inputs — schematic
values, placement, `mech/*.yaml`; and the tool version determines the generated
geometry itself.

`cctl freeze` therefore records a release identity, not a lock tag:

```
fabrication release
├── project git revision
├── circuit-control version
├── catalog lock hash
└── KiCad version + board format version
```

KiCad's version is recorded rather than optional: emitted documents carry a
format stamp (`20260206` for this fixture), and generated artifacts were
validated by a specific `kicad-cli`. Reproducing what was fabricated means
reproducing all four. §21 criterion 1 and §8 provenance both point here.

**Deferred:** version-range resolution and conflict solving across a dependency
graph. Parts will genuinely compose — a knob on a shaft, a drilled 1590BBS
variant, a jack pre-mounted in an enclosure — so part identity is stable and
referenceable from day one. But there is nothing to resolve with a flat catalog
and one consumer. The first real composite part should force the shape of the
resolver.

## 13. Testing

`kicad-cli` provides **headless validation and export** operations today — not
headless editing, per the distinction in Section 8 — and is used as an
**oracle**: KiCad itself confirms what the hand-rolled bridge writes. This is
what makes a hand-rolled s-expression writer acceptable.

| We write | KiCad verifies |
|---|---|
| class-A placement | `pcb export pos` — does KiCad agree RV1 is where we put it? |
| board file | `pcb drc` — does KiCad accept and check it? |
| generated `.kicad_mod` | `fp upgrade`, `fp export svg` |
| footprint assignments | `sch export bom` — all 76 fields populated? |
| whole board | `pcb render`, `pcb export glb` |

`kicad-cli` is therefore a required development and CI dependency, though not a
runtime one.

Additional required coverage:

- Y-flip transform, with an asymmetric fixture (Section 10).
- Q1–Q4 pin mapping: `Simulation_SPICE:NPN` 1/2/3 to TO-92 C/B/E.
- **Lossless-write invariant** (Section 8): byte-for-byte round-trip on every
  fixture document with no mutation requested, and refusal-or-preservation on
  constructs the backend does not model. This is a contract test, not a
  fixture check.
- `verify` does not mutate: run `verify` twice against a dirty project and
  confirm the working tree is byte-identical after each.
- Transaction behaviour, stated to match the contract in Section 9 rather than
  overstate it: induce failure at **every pre-commit stage** and confirm the
  project is unmodified; induce failure **during ordered replacement** and
  confirm the partial state is detected and reported, and recoverable via git.

### Additional acceptance criterion: drift detection

PRD §21 criterion 9 covers moving a footprint until verification fails. That
generalises into a criterion the design needs in its own right:

> **Any manual modification to state governed by its configured generation
> strategy must be detected by `cctl verify`.**

Phrased against the strategy rather than against a fixed list, because
ownership is configurable: under `outline.strategy: enclosure_inset` the board
outline is governed state and drift is a failure, while under a future
`explicit` strategy the outline is operator-owned and editing it is not drift
at all. The ownership model decides what counts.

Without it, `sync` is authoritative only at the instant it runs, and every
guarantee in Section 6's ownership rule decays silently from that moment.

## 14. Milestones

**M1 — one part, end to end, including the failure.** RV1's pot only:
`part.yaml` with typed bases and hashed documents → emitted `.kicad_mod` →
assigned to RV1 → placed from an operator-declared panel coordinate → 3D built
→ one alignment check against a 1590BBS panel hole → GLB, `checks.json`, text
report.

**Completion requires proving the negative path.** A happy path proves a
pipeline can produce an agreeable answer; it does not prove the thesis. M1 is
done when the fixture demonstrates:

```
source → model → generated ECAD → assembly → PASS
                                      ↓
                            introduce drift
                                      ↓
                     FAIL + non-zero exit + dependency provenance
```

Perturb the panel position or the board state, require FAIL with a non-zero
exit, and require `cctl explain` to name the changed input. Restore, require
PASS. At that point §23's claim holds at n=1 rather than being asserted.

Exercises every component at n=1 and settles the coordinate-frame flip.
Acceptance criteria 2, 3, 4, 5, 7, 9, 10, 11 hold in miniature.

**M1 is blocked on open question 3.** It is written above as "RV1's pot" rather
than "the Alpha RV16AF" deliberately: the fixture's pot library is an Adafruit
5283 while the adjacent datasheet is an Alpha RV16AF, so which part RV1 *is*
has not been established. Resolving that is agent legwork and precedes M1.

**M2 — the whole board gets footprints, under lock.** All 76 symbols bound via
`bindings`: axial resistor, disc and radial capacitor, TO-92, DO-35 packages
plus SW1. Board outline via `enclosure_inset`. **The operator is unblocked
here** — the pedal can be laid out.

M2 also carries **content identity and lock enforcement**: package hashes,
`circuit-control.lock`, and refusal to build against a stale lock. M2 is where
catalog data starts generating fab-affecting geometry, and that is precisely
where reproducibility becomes load-bearing — deferring it to M4 would mean the
first board is laid out against unpinned inputs. The upgrade *experience* stays
in M4; only identity and enforcement move up.

**M2.5 — parts sourcing.** The catalog verbs from Section 16: `catalog new`,
`catalog datasheet` with document hashing, `catalog check` with the provenance
and evidence-class gates, `catalog measure`, the sourcing block, and
`cctl bom`. Exercised by sourcing the 3PDT footswitch end to end, which is the
part M3 is blocked on.

**M3 — class C and clearance.** Neutrik TRS, KC-301339, footswitch, LED as
enclosure objects, using envelopes rather than nominal geometry. Component
height versus lid, and board versus enclosure-mounted parts. Acceptance
criterion 8. Depends on M2.5 for the footswitch.

**M4 — catalog upgrade experience.** `cctl catalog update` with the semantic
dimensional diff and constraint re-evaluation, `cctl freeze`, release history.
Identity and locking already exist from M2.

**M5 — viewer and CI.** GLB viewer page, report polish, `cctl verify` as a
repository check (§17).

**Why M1 precedes M2**, despite M2 being what unblocks the operator: footprint
pads and 3D geometry derive from the same `part.yaml`. Building the emitter
across 76 symbols before anything consumes the geometry side would produce a
schema serving copper and fighting mechanics. M1 is small at n=1 and forces
both halves to agree.

## 15. V1 scope

In, per §19: one KiCad PCB; board outline and holes; footprint placement and
rotation; top-side components; datasheet-derived simplified models; boxes and
cylinders; explicit component frames; component interfaces; one enclosure/panel
assembly; alignment constraints; clearance and collision; populated-board 3D;
machine-readable verification; CLI; reproducible setup.

Out, per §5 and §20: general-purpose CAD; photorealistic modelling; automatic
datasheet extraction; PCB electrical design; bottom-side components; complex
extrusions; STEP import and export; DXF generation; artwork and SVG
verification; worst-case tolerance analysis; automatic footprint association;
multiple or flex PCBs; fastener modelling.

Tolerances (G8) are **recorded in `part.yaml` from v1** but not yet reasoned
over — worst-case analysis is deferred. Recording them early avoids a schema
migration later.

**Additions beyond PRD §19**, both driven by operator requirements stated after
the PRD was written: the project parts manifest with `cctl bom` (Section 6),
and agent-driven parts sourcing with its provenance gate (Section 16). Neither
appears in §19; both are in v1 scope.

## 16. Parts sourcing

The operator's third stated pain, alongside library management and footprint
hunting, is **parts hunting**: they want to say "find the footswitch everyone
uses, put it in the library, add it to the project" and have that happen.

NG3 permits this. It defers *automatic* datasheet extraction while explicitly
allowing "an agent or human" to translate dimensional drawings into component
definitions, and §16 of the PRD describes precisely that authoring loop. This
section fills in the step before it: how the datasheet is acquired.

### Division of labour

**The agent does the judgment.** Identifying the canonical part, searching
distributors, and reading a product page are knowledge and web tasks, not API
queries. Amplified Parts and StompBox Parts have no API at all, and a
hard-coded scraper would be fragile, unmaintainable, and — per the operator's
standing instruction against fallbacks — a bug factory.

**The tool provides verbs and the guardrail.** No scrapers ship in the binary.
The agent uses `cctl catalog new`, `cctl catalog datasheet`, and edits
`part.yaml`; the tool validates the result and records sourcing in a block
carrying distributor, manufacturer part number, and URL, so reordering is one
lookup.

**The manifest, never the schematic.** A sourced part is added to
`mech/parts.yaml`, not to `fuzz.kicad_sch`. Wiring a bypass footswitch is
circuit design and belongs to the operator under NG4.

### Interaction model: the operator decides, the agent does the legwork

A standing operator constraint, in their words: "You can ask me to choose a
part from a list of approved suppliers, but you can't just make me do all the
legwork. Legwork is for computers and agents."

This is binding on the tool and on any agent operating it:

- **Never ask the operator for part data.** Not dimensions, not manufacturer
  part numbers, not datasheet links, not stock or pricing. Every one of those
  is findable.
- **Ask the operator to choose**, and present a short list of real candidates
  with the data already gathered — supplier, part number, price, availability,
  whether a dimensioned datasheet exists — so the decision is a judgment call,
  not an errand.
- **The operator makes the final call on parts**, and updates the schematic
  with off-board parts when they choose to.
- **The provenance gate binds the agent, never the operator.** A missing
  dimension is the agent's problem to solve by finding the document. It must
  never surface as a prompt asking the operator to supply the number.

Approved suppliers are configured, not improvised, so searches stay inside
sources the operator actually buys from. For this project: Amplified Parts,
StompBox Parts, Love My Switches, Tayda, Digi-Key, Mouser.

The same rule applies to open questions in this document. An unresolved
question is an agent research task producing a recommendation, not a
questionnaire.

### Provenance is a hard gate

If an agent authors catalog entries, an agent can hallucinate a dimension, and
a hallucinated dimension produces a confidently wrong PASS. That would destroy
the premise of the whole system — §23's chain is worth nothing if any link is
invented.

But "every dimension must cite a datasheet" is the wrong gate, and the `~` in
the worked example below is what exposes it. A wiring or service envelope
generally *cannot* come from a manufacturer: they specify the solder lug, not
how much room your wire gauge, strip length, solder joint, and bend radius
consume. A design clearance of 1.0 mm is not a documentary fact either — it is
engineering policy.

An unsatisfiable gate is worse than no gate. It does not prevent invention; it
pressures the author into attributing a legitimate engineering number to a page
that says nothing about it. That is a *more* corrosive failure than an
uncited number, because it looks sourced.

So the gate is on **basis**, not on documents:

> **Every mechanically significant numeric input must declare an explicit
> basis.** Source-derived dimensions require resolvable documentary provenance.
> Measurements require specimen provenance. Engineering allowances and design
> rules must be explicitly typed and attributable. No basis may masquerade as
> another.

`cctl catalog check` enforces that a basis exists and is well-formed for its
type. A missing or malformed basis is a hard failure, not a lint warning.

### Documents are first-class, and identified by content

A filename is not an identity. A supplier can silently replace a PDF while
keeping its name, and the catalog would never notice — a hole straight through
the premise. Documents are therefore catalog objects with content hashes:

```yaml
documents:
  datasheet:
    path: aionfx-3pdt-stomp-switch.pdf
    sha256: <hash>
    retrieved: 2026-09-04
    source_url: https://aionfx.com/app/files/datasheets/aionfx-3pdt-stomp-switch.pdf
    kind: supplier_drawing
```

Dimensions then reference the document by key, closing the chain:

```
dimension → document identity → exact bytes → external origin
```

This lands in **M1**, not later. It is cheap, and it is the difference between
provenance that is checkable and provenance that is decorative.

### Three distinct claims, three distinct names

An earlier draft used `verified` for "checked against a physical specimen",
while the whole system uses *verification* to mean evaluating an assembly
against authoritative dimensions. Those are different claims and now have
different names:

| Claim | Question it answers |
|---|---|
| **provenance validation** | Is every dimension grounded in an identified document? |
| **physical validation** | Has a specimen been measured, and does it agree? |
| **assembly verification** | Do the constraints hold? |

### Measurement is a cross-check, not a promotion in authority

The earlier design had calipers outranking the datasheet. That is wrong, and
wrong in a way that would make verification *less* sound.

A specimen tells you what **one specimen** measures. A manufacturer's drawing
tells you what the part is **specified to guarantee**. For clearance design the
specification is the more authoritative input: if the datasheet says
`15.5 ±0.5` and your specimen measures `15.2`, replacing 15.5 with 15.2 designs
against a sample rather than against the production contract, and the next unit
may be 16.0.

So physical measurement records agreement; it does not replace the source:

```yaml
physical_validation:
  status: unmeasured        # | cross_checked | discrepant
```

`discrepant` — a specimen outside the document's stated tolerance — is a
finding in its own right. It means the document is wrong, the part is not what
was ordered, or the specimen is counterfeit. All three are worth failing over.

### Basis types are the evidence taxonomy

Basis and "evidence class" are not two mechanisms. A number's basis *is* what
kind of evidence stands behind it, and maintaining two parallel vocabularies
over the same values would guarantee they drift apart. One taxonomy:

| `basis.type` | Requires | Example |
|---|---|---|
| `manufacturer_datasheet` | document + page | body height |
| `manufacturer_drawing` | document + page | the Hammond 1590BBS drawing |
| `supplier_drawing` | document + page | the generic 3PDT drawing |
| `generic_reference` | document + page | a family convention |
| `physical_measurement` | specimen id, instrument, date | measured shaft diameter |
| `assembly_allowance` | rationale | wiring envelope |
| `design_rule` | named rule | default component clearance |
| `operator_declared` | file + field | the PCB-to-world transform |

**Categorical, not a total order.** A default strength ordering exists for
reporting, but it is not baked into the model, because prestige is not
fitness. A manufacturer datasheet that omits the dimension is worthless *for
that dimension*; a manufacturer's dimensional drawing often beats their general
datasheet; a distributor-hosted manufacturer drawing is still authoritative;
and an enclosure drawing is not a "datasheet" at all.

Checks therefore declare **acceptable bases** rather than a minimum rank:

```yaml
- id: shaft-to-panel-hole
  evidence:
    require: [manufacturer_datasheet, manufacturer_drawing, physical_measurement]
```

Under that policy the current footswitch entry — backed by a generic supplier
drawing with no tolerances — **fails**, naming the weak link rather than
silently reporting PASS. Under a looser policy for a non-critical clearance, it
passes. Fitness for the claim, decided per claim.

### Worked example

The first sourced part, selected by the operator from a candidate list and
vendored at `catalog/switches/3pdt-stomp/`.

```yaml
id: switches/3pdt-stomp
mpn: FSW-1032

documents:
  datasheet:
    path: aionfx-3pdt-stomp-switch.pdf
    sha256: <hash>
    retrieved: 2026-09-04
    kind: supplier_drawing        # weaker than manufacturer_datasheet

physical_validation:
  status: unmeasured

sourcing:
  - supplier: StompBox Parts
    sku: FSW-1032
    url: https://stompboxparts.com/switches/3pdt-footswitch-pro-latching-solder-lug/
    price_usd: 4.30

mount: panel                      # class C — not on the board
termination: solder_lug

geometry:                         # nominal — what the object is
  - cylinder: {id: bushing,   diameter: 12.0, length: 11.5, axis: z}
  - cylinder: {id: actuator,  diameter: 10.0, length: 5.0,  axis: z}
  - box:      {id: body,      size: [19.6, 17.0, 15.5]}
  - box:      {id: terminals, size: [19.6, 17.0, 3.2]}

envelope:                         # space that must remain available
  - box:
      id: wiring
      of: terminals
      size: [19.6, 17.0, ~]       # not yet established — see below
      basis:
        type: assembly_allowance
        rationale: "22 AWG stranded, solder lug, 90° bend"

interfaces:
  - {id: panel_axis,       type: cylindrical_axis, geometry: bushing}
  - {id: panel_mount_face, type: planar_interface}

panel:
  hole_diameter:
    value: 12.5
    unit: mm
    basis: {type: supplier_drawing, document: datasheet, page: 1,
            drawing: Drill Dimensions}

keepouts:
  clearance:
    value: 1.0
    basis: {type: design_rule, rule: default_component_clearance}
```

Three different bases appear there, and none pretends to be another: the panel
hole is documentary, the wiring envelope is an assembly allowance with a stated
rationale, and the clearance is project policy. Forcing the latter two through
documentary provenance would have produced fiction.

### Nominal geometry, envelope, and keepout are distinct

Collision checking exposes the difference immediately. Nine solder lugs are not
merely a 3.2 mm box: attached wire and solder fillets consume real additional
space, and a jack needs room to insert a plug or turn a nut. Encoding "what the
object looks like" and "space that must remain available" as the same field
guarantees a later schema migration.

So the schema separates them from the start:

- **`geometry`** — nominal form. Drives GLB rendering.
- **`envelope`** — maximum manufacturing, installation, and service volume.
  Drives collision and clearance.
- **`keepouts`** — declared design clearance on top of the envelope.

V1 does not implement rich envelope semantics — an envelope may simply be a
scaled or offset primitive — but the distinction exists now so that collision
operates on envelopes while rendering shows nominal form.

The `~` above is deliberate: the wiring envelope for this switch has **no
established allowance yet**. Its basis type is known — an assembly allowance,
not a documentary fact — but the number itself requires a decision about wire
gauge and assembly practice that has not been made. Writing a plausible value
would be exactly the invention the gate exists to prevent, and typing its basis
honestly is what makes the gap visible rather than papered over.

## 17. Open questions

Per Section 16, these are **agent research tasks producing recommendations**,
not requests for the operator to supply information. Each is resolved by
finding a document, then putting a choice with its consequences to the
operator.

**Resolved.**

- *Footswitch selection* — the operator selected the StompBox Parts 3PDT PRO
  (FSW-1032), solder lug, class C, from a sourced candidate list.
- *KiCad `.proto` publication* — published in the KiCad repository as a
  submodule, with `gitlab.com/kicad/code/kicad-rs` an official Rust binding and
  `kicad-ipc-rs` an MIT alternative. The Section 8 migration target is
  concrete.
- *Assembly geometry* — **out of scope for inference.** The operator specifies
  the PCB-to-world transform and mounting arrangement in `enclosure.yaml`. The
  tool verifies consequences; it does not propose the arrangement.
- *RV1's identity* — the operator specified Alpha, and selected the
  **RV16AF-41**, vertical PC-board terminals. RV1 is therefore **class A**. The
  Adafruit 5283 library in `assets/` is discarded rather than reconciled. The
  Taiwan Alpha 16 mm Metal Shaft Series catalog is vendored at
  `catalog/potentiometers/alpha-rv16af-41/`, and its page 45 carries the PCB
  mounting hole detail that fully determines generated copper — satisfying the
  package-completeness rule in Section 6.

**Open, with the legwork identified:**

1. **SW1 mounting.** The Taiway 100DP1T8B13M2QEH datasheet is already in the
   operator's `assets/toggle/`, with a vendor `.kicad_mod`. Determine which
   mounting variants the part is offered in and present the class A / class C
   choice. Blocks SW1's class assignment and therefore part of M2.
2. **Wiring envelope for solder-lug parts.** The 3PDT, jacks and DC jack all
   terminate in lugs whose wire and solder consume space beyond nominal
   geometry. Needs a sourced or measured figure before M3's clearance checks
   mean anything.
3. **Canonical retrieval for the Alpha catalog.** The vendored copy came from a
   third-party mirror (`electricdruid.net`), not `taiwanalpha.com`. The
   document is a manufacturer datasheet and its basis type says so; the
   `source_url` honestly records a mirror. Re-fetch from Taiwan Alpha before
   this grounds a fabricated board, and re-hash. This is a useful demonstration
   that document *authorship* and document *retrieval* are separate facts, both
   recorded.
4. **Shaft variant for RV1.** Order-code position 9 selects shaft type
   independently of terminals: the catalog drawings show an 18-tooth splined
   Ø6 shaft, while the Mouser-specific drawing for the `-10` showed a Ø6.35
   variant. This affects knobs, not the footprint, so it does not block M1 —
   but it must be settled before a knob is modelled.
