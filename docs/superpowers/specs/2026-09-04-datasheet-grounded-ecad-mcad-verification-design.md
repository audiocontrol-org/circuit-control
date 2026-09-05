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
searching for footprints, and lining up components on a PCB against real parts
in the real world. They are happy to do PCB layout and routing.

This sets the product boundary precisely:

- The computer owns **every footprint** and the placement of parts whose
  position is physically determined by the enclosure.
- The human owns **layout and routing** of everything else.

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

**Known gap:** the footswitch has no datasheet — only a photograph. It must be
measured or sourced before it can be modelled. This blocks part of M3.

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
use with a drill — and the tool solves for the board coordinates. The board
outline is likewise generated from the enclosure interior minus clearance, not
drawn.

Class A placement is therefore a *derived* quantity, and G1 is honoured for
classes B and C only. This is a deliberate, documented departure from the PRD.

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
    ├── enclosure.yaml                1590BBS + class-C placement
    ├── panel.yaml                    control positions in panel coordinates
    ├── bindings.yaml                 RV1 → alpha-rv16af; Q1..Q4 → bc548-to92
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

**`mech/parts/` shadows the catalog.** Adding a part must never require a
commit to the tool repository, or library management returns as friction.
Promotion to the catalog is a file move once a part proves out.

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
- **Catalog** — YAML to part model: pads, geometry, interfaces, keepouts,
  provenance. Resolves `mech/parts/` over `catalog/`.
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
cctl catalog update    upgrade a pinned part, with dimensional diff
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

Open item, not blocking v1: confirm whether KiCad publishes `.proto` files for
non-Python clients. If not, the bridge simply lives longer.

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

## 10. Coordinate frames

Per §9, frames must be documented and tested.

```
world  (1590BBS interior floor, back-left corner, Y-up, right-handed, mm)
 ├── enclosure ── panel ── opening/interface
 ├── PCB ── footprint ── component ── interface
 └── artwork                                    (deferred, §20)
```

The PCB's relationship to world is one explicit transform declared in
`enclosure.yaml`.

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

## 12. Catalog versioning

The catalog accretes, so identity must be right early.

**Change severity is computed, not judged.** The footprint emitter and the
geometry core consume disjoint fields of `part.yaml`, so classification falls
out of the data model:

| Severity | Fields | Why it matters |
|---|---|---|
| **fab-affecting** | pad position, pitch, drill, courtyard | Feeds copper. Physically committed once routed or fabricated; must never move silently. |
| **mechanical-only** | body height, shaft diameter, keepout | Feeds geometry and checks. May change; consequences surface as PASS/FAIL. |
| **breaking** | interface renamed or removed | `checks.yaml` references interfaces by name. Hard error naming every dangling check. |

**Pinning uses content hashes.** `circuit-control.lock` records a hash of every
part definition resolved. Any catalog edit makes the lock stale; nothing
recomputes silently. A human-facing `version:` field exists in `part.yaml` for
communication, but the lock keys on the hash so no one's diligence is
load-bearing.

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

**`cctl freeze`** tags the lock against a fabrication release. Once a board is
manufactured its footprints are physical fact, and §21 criterion 1 plus §8
provenance both require reproducing exactly what was fabricated.

**Deferred:** version-range resolution and conflict solving across a dependency
graph. Parts will genuinely compose — a knob on a shaft, a drilled 1590BBS
variant, a jack pre-mounted in an enclosure — so part identity is stable and
referenceable from day one. But there is nothing to resolve with a flat catalog
and one consumer. The first real composite part should force the shape of the
resolver.

## 13. Testing

`kicad-cli` is headless today and is used as an **oracle**: KiCad itself
confirms what the hand-rolled bridge writes. This is what makes a hand-rolled
s-expression writer acceptable.

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
- Round-trip: parse then re-emit an untouched document produces no diff.
- Acceptance criterion 9: moving a footprint far enough fails verification;
  restoring it passes.

## 14. Milestones

**M1 — one part, end to end.** Alpha RV16AF only: `part.yaml` from datasheet →
emitted `.kicad_mod` → assigned to RV1 → placed from a panel coordinate → 3D
built → one alignment check against a 1590BBS hole → GLB, JSON, non-zero exit.

Exercises every component at n=1 and settles the coordinate-frame flip.
Acceptance criteria 2, 3, 4, 5, 7, 9, 10, 11 hold in miniature.

**M2 — the whole board gets footprints.** All 76 symbols bound: `R_Axial`,
`C_Disc`, `C_Radial`, `TO-92`, `DO-35` families plus SW1. Board outline
generated from the 1590BBS interior. **The operator is unblocked here** — the
pedal can be laid out.

**M3 — class C and clearance.** Neutrik TRS, KC-301339, footswitch, LED as
enclosure objects. Tall-capacitor-versus-lid and board-versus-jack collision.
Acceptance criterion 8. Blocked in part by the missing footswitch datasheet.

**M4 — catalog versioning.** Hashes, lock, `cctl catalog update` with
dimensional diff, `cctl freeze`.

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

## 16. Open questions

1. **Footswitch dimensions.** No datasheet exists. Measure or source before M3.
2. **SW1 mounting.** The Taiway 100DP1T8B13M2QEH is typically panel-mounted and
   hand-wired, but SW1 appears in the schematic and so is on the board. Confirm
   whether SW1 is board-mounted through a panel hole (class A) or panel-mounted
   and wired (class C, with the schematic symbol carrying no footprint).
3. **PCB mounting method.** Standoffs to the enclosure floor, or suspended on
   pot bushings? Determines the PCB-to-world transform in `enclosure.yaml`.
4. **KiCad `.proto` publication** for non-Python clients. Affects the timing of
   the IPC backend, not v1.
