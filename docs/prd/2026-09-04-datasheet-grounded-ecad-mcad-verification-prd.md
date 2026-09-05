# PRD — Datasheet-Grounded ECAD↔MCAD Verification

**Status:** Draft  
**Date:** 2026-09-04

## 1. Summary

Build a code-first tool for generating and verifying mechanical models of PCB-based assemblies.

The tool bridges three sources of engineering truth:

1. **KiCad** — authoritative for PCB geometry, footprint placement, pad locations, and component orientation.
2. **Manufacturer datasheets** — authoritative for component dimensions and mechanical interfaces.
3. **Mechanical design definitions** — authoritative for enclosures, panels, brackets, artwork, and other non-PCB geometry.

From these inputs, the tool generates a simplified 3D mechanical assembly and performs deterministic checks such as component-to-enclosure alignment, PCB and component clearances, connector/cutout alignment, shaft/hole alignment, mounting-hole alignment, component collisions, PCB fit, and panel/artwork registration.

The goal is not to replace a general-purpose CAD system. It is to make the mechanical consequences of an ECAD design **reproducible, inspectable, testable, and automatable from source**.

The initial validation project may be an audio-electronics enclosure, but the architecture must make no assumptions specific to guitar pedals or audio equipment.

---

## 2. Problem

PCB assemblies frequently interact with mechanical structures:

- connectors pass through enclosure walls;
- potentiometers and encoders pass through panels;
- switches must align with openings;
- LEDs must align with lenses or artwork;
- mounting holes must align with standoffs;
- tall components must clear enclosure surfaces;
- heatsinks must contact or avoid other structures;
- displays must align with windows;
- PCB edges must fit within chassis constraints.

ECAD tools accurately describe PCB geometry and component placement, but they are not the natural source of truth for the surrounding mechanical assembly.

General-purpose MCAD tools can model the complete assembly, but maintaining component placement independently in MCAD creates a synchronization problem. Moving a footprint in the PCB layout can silently invalidate enclosure holes, artwork, mounting geometry, or clearance assumptions.

Manufacturer-provided 3D models do not completely solve this problem: models may not exist; available models may be inaccurate; model origins may be arbitrary; third-party models may have unclear provenance; and visually detailed models may still have incorrect mechanically important dimensions.

Manufacturer datasheets, by contrast, normally specify the mechanical dimensions required to design around the component. The desired system therefore treats **datasheet dimensions rather than downloaded 3D geometry as the primary mechanical authority for components**.

---

## 3. Product Principle

The system should answer:

> Given the PCB that will actually be manufactured and the dimensions manufacturers specify for the physical parts, will the resulting physical assembly fit and align correctly?

The answer must be derived mechanically rather than by visual inspection. A 3D rendering is useful evidence. A deterministic verification result is the product.

---

## 4. Goals

### G1 — Derive mechanical placement from KiCad

Read a KiCad PCB and obtain, at minimum:

- board outline;
- footprint references and identifiers;
- footprint origins, X/Y placement, and rotation;
- board side;
- pad geometry and locations where required;
- PCB thickness where available or configured; and
- PCB holes and cutouts.

The user must not manually duplicate PCB component coordinates in the mechanical model.

### G2 — Support datasheet-derived component models

Allow components to be represented using simplified parametric geometry derived from manufacturer dimensional drawings. Typical primitives should include boxes, cylinders, cones, extrusions, holes, planes, axes, and simple polygons.

A component model should contain only as much geometry as necessary for mechanical reasoning. Visual fidelity is explicitly not a goal.

### G3 — Establish an explicit footprint↔component coordinate contract

Every component model must have a defined relationship to its KiCad footprint. The relationship must be deterministic and reviewable.

Examples:

- model origin corresponds to footprint origin;
- model origin corresponds to a particular pad;
- seating plane corresponds to the PCB surface; and
- component axes and offsets are expressed in footprint-local coordinates.

The system must never depend on manually positioning a component by eye.

### G4 — Generate the populated PCB mechanically

Given PCB geometry, KiCad footprint placements, and component mechanical definitions, generate a 3D representation of the populated PCB.

Moving or rotating a footprint in KiCad must automatically move or rotate its mechanical representation on the next build.

### G5 — Model external mechanical structures

Support code/data-defined geometry for enclosures, panels, chassis, brackets, standoffs, fixtures, mounting plates, cutouts, and external mechanical components.

Imported CAD may be supported, but generated models must not depend on imported CAD being available.

### G6 — Model mechanical interfaces explicitly

Components and structures should expose named mechanical interfaces, including cylindrical axes, panel pass-throughs, connector openings, mounting axes and planes, seating planes, mating faces, display apertures, and heatsink contact surfaces.

Interfaces provide stable semantic objects against which constraints can be expressed.

### G7 — Perform deterministic mechanical verification

Support machine-evaluable constraints such as:

- axis alignment and concentricity;
- positional alignment;
- minimum clearance and maximum separation;
- coplanarity and containment;
- interference and required overlap; and
- hole alignment.

Verification must produce numeric results rather than merely visual warnings.

```text
J2.panel_interface

allowed axis offset: <= 0.25 mm
actual axis offset:     0.08 mm
result:                 PASS
```

### G8 — Support nominal dimensions, tolerances, and design clearances

The model must distinguish between manufacturer nominal geometry, manufacturer tolerance, fabrication tolerance, assembly tolerance, and user-selected design clearance.

```text
component body:
    nominal width: 17.8 mm
    tolerance: ±0.2 mm

design keepout:
    minimum additional clearance: 1.0 mm
```

Verification should eventually be capable of reasoning about worst-case geometry rather than nominal geometry alone.

### G9 — Support artwork and panel-layout registration

2D artwork and fabrication geometry should be capable of sharing a coordinate system with the mechanical model. This enables verification of relationships such as control center ↔ artwork center, LED ↔ printed indicator, display ↔ artwork window, connector ↔ panel label, and mounting hole ↔ fabrication drawing.

SVG and DXF are initial candidate interchange formats. Artwork itself is not authored by this tool.

### G10 — Be automation-first

The complete model must be buildable and verifiable without interactive CAD operations. A clean checkout should support a workflow equivalent to:

```text
git clone <repo>
cd <repo>
./dev verify
```

No manual CAD manipulation may be required to reproduce the verification result.

---

## 5. Non-Goals

### NG1 — General-purpose CAD

The system is not intended to replace Onshape, SolidWorks, Fusion, FreeCAD, or similar tools for arbitrary mechanical design.

### NG2 — Photorealistic component modeling

Models should represent mechanically relevant geometry, not cosmetic detail. Threads, logos, tiny fillets, surface finishes, lettering, and similar features should normally be omitted.

### NG3 — Automatic datasheet interpretation in v1

An agent or human may translate manufacturer dimensional drawings into component definitions. Automatically extracting dimensions from PDFs is a possible future capability but is not required initially.

### NG4 — PCB electrical design

KiCad remains responsible for schematic capture, footprints, routing, design rules, and manufacturing outputs.

### NG5 — Replacing manufacturer tolerances with inferred geometry

Downloaded STEP files or other CAD assets must not silently override explicitly recorded datasheet dimensions.

---

## 6. Source-of-Truth Model

The system must maintain explicit ownership boundaries.

### KiCad owns

- PCB outline;
- holes and cutouts represented by the PCB design;
- footprint placement and rotation;
- board side;
- pad placement; and
- PCB component identity.

### Component definitions own

- physical component geometry;
- component-local coordinate system;
- seating relationship to PCB;
- mechanically relevant axes and faces;
- datasheet-derived dimensions;
- tolerances; and
- component-specific keepouts.

### Mechanical assembly definitions own

- enclosure and panel geometry;
- brackets and standoffs;
- non-PCB component placement; and
- mechanical relationships.

### Artwork owns

Graphic geometry.

### Verification definitions own

Acceptable relationships between those objects.

No derived artifact should become a competing source of truth.

---

## 7. Component Library

Component definitions should be reusable across projects.

```text
components/
├── connectors/
├── switches/
├── potentiometers/
├── encoders/
├── displays/
├── relays/
├── transformers/
├── heatsinks/
├── semiconductors/
└── mechanical/
```

A component definition should include a stable component identifier, manufacturer, manufacturer part number, datasheet reference, footprint mapping information, mechanical geometry, interfaces, relevant tolerances and keepouts, and provenance for mechanically significant dimensions.

```yaml
id: example-component
manufacturer: Example Corp
mpn: ABC-123

footprint:
  anchor: pad-1

geometry:
  - box:
      id: body
      size: [17.8, 16.0, 9.5]

  - cylinder:
      id: shaft
      diameter: 6.0
      length: 15.0
      axis: z
      center: [7.5, 0, 17.0]

interfaces:
  - id: shaft_axis
    type: cylindrical_axis
    geometry: shaft
  - id: pcb_seating_plane
    type: planar_interface

keepout:
  clearance: 1.0
```

The final schema is a technical-design decision rather than a PRD requirement.

---

## 8. Dimension Provenance

Mechanically significant dimensions should be traceable to their source.

```yaml
shaft_diameter:
  value: 6.0
  unit: mm
  tolerance: 0.05
  source:
    type: manufacturer_datasheet
    document: ABC-123.pdf
    page: 4
    drawing: Mechanical Dimensions
```

The system should make it possible to answer:

> Why does the model believe this dimension is 6.0 mm?

This provenance should survive generation of derived artifacts.

---

## 9. Mechanical Coordinate Model

The system must define stable coordinate frames for PCB, footprints, components, mechanical structures, and artwork where applicable. Transformations between frames must be deterministic.

```text
world
 ├── PCB
 │    └── footprint
 │          └── component
 │                └── interface
 │
 ├── enclosure
 │    └── panel
 │          └── opening/interface
 │
 └── artwork
```

The exact coordinate convention is a technical-design decision, but it must be documented and tested.

---

## 10. Verification

Verification is a first-class output. Initial capabilities should include:

- **Alignment:** compare positions and/or axes of two interfaces.
- **Clearance:** determine minimum separation between selected physical or keepout geometry.
- **Collision:** determine whether two forbidden volumes intersect.
- **Containment:** verify that an object or keepout lies within an allowed volume.
- **Mounting:** verify alignment between mounting features.
- **PCB fit:** verify that the populated PCB and associated keepouts fit within the permitted mechanical envelope.

Results should identify the constraint, participating objects, required value or tolerance, measured value, PASS/FAIL status, and useful diagnostic information.

---

## 11. Outputs

The build should be capable of producing some combination of:

```text
out/
├── assembly.step
├── assembly.glb
├── pcb.step
├── enclosure.step
├── panel.dxf
├── artwork-reference.svg
├── mechanical-checks.json
└── mechanical-report.txt
```

Required v1 outputs are:

1. inspectable 3D assembly;
2. machine-readable verification results; and
3. human-readable verification summary.

STEP is preferred for interoperable mechanical geometry where supported. A lightweight browser-viewable representation such as GLB is desirable.

---

## 12. CLI

The system should expose a stable, public Rust command-line interface.

```text
mcad build
mcad verify
mcad render
mcad inspect <component>
```

Exact naming is TBD. `verify` must return a non-zero process status when required mechanical constraints fail so that it can act as a CI gate.

```text
$ mcad verify

PCB
  board envelope ................ PASS
  enclosure clearance ........... PASS

Interfaces
  J1 connector ↔ opening ........ 0.08 mm   PASS
  SW1 shaft ↔ opening ............ 0.00 mm   PASS
  DS1 display ↔ window ........... 0.14 mm   PASS

Clearance
  J1 ↔ chassis ................... 2.84 mm   PASS
  SW1 ↔ PCB ...................... 0.63 mm   FAIL
      required >= 1.00 mm

Artwork
  DS1 ↔ artwork window ........... 0.05 mm   PASS

FAILED: 1 mechanical constraint
```

---

## 13. Repository and Dependency Requirements

Reproducibility is a product requirement, not merely a development preference.

The public interface should be implemented as a Rust CLI. The geometry backend may use any appropriate implementation language or CAD kernel. However:

> Backend implementation choices must not leak into the user's development environment.

The repository must provide complete dependency management for the Rust toolchain, geometry backend, scripting runtime if present, CAD kernel, KiCad tooling required by the build, and rendering/export dependencies.

The user must never be expected to manually create Python virtual environments, run `pip install`, install arbitrary system libraries, configure CAD-library paths, resolve compiler versions manually, or maintain globally installed backend dependencies.

Nix or an equivalently robust hermetic environment is acceptable as the outer dependency boundary. Language-specific lockfiles should additionally pin internal dependency graphs.

---

## 14. Backend Isolation

The Rust frontend owns the product model and semantics. The CAD backend owns geometric computation.

The boundary should be narrow enough that the geometry backend can eventually be replaced without redefining components, interfaces, constraints, provenance, KiCad integration, or verification semantics.

```text
Rust

KiCad parser
component model
assembly model
constraint model
verification
CLI
       │
       │ geometry operations
       ▼
CAD backend

primitives
transforms
BREP operations
distance/intersection
tessellation
STEP export
```

build123d, CadQuery, OpenCascade, FreeCAD libraries, or another kernel may therefore be evaluated during technical design without changing the product contract.

---

## 15. Imported CAD

The system may support STEP or other imported geometry. Imported geometry is useful for enclosure models, complex mechanical structures, comparison against manufacturer models, and geometry whose authoritative source is legitimately CAD.

However, imported component CAD must not automatically supersede datasheet-derived geometry. Where both exist, the system should eventually support comparing the imported model against the datasheet-grounded model.

---

## 16. Agent-Native Authoring

The system should be straightforward for coding agents to operate. An agent should be able to:

1. inspect a component datasheet;
2. create or update its mechanical definition;
3. build the model;
4. inspect generated measurements;
5. run verification;
6. modify geometry; and
7. repeat.

The canonical representation of components and assemblies must be text-based and suitable for Git, diffs, code review, automated editing, and testing. Binary CAD documents must not be required as canonical project state.

---

## 17. CI

Mechanical verification should eventually be usable as an ordinary repository check.

```text
PCB layout change
        │
        ▼
CI regenerates assembly
        │
        ▼
mechanical verification
        │
   ┌────┴────┐
 PASS       FAIL
```

A footprint move that causes a connector to become misaligned with an enclosure opening should therefore be capable of failing a pull request in exactly the same way an automated software test can fail.

Generated binary artifacts need not necessarily be committed.

---

## 18. Initial User Workflow

### Create project

Define PCB source, enclosure/mechanical structures, and verification rules.

### Add component

Obtain the manufacturer datasheet. Create a simplified mechanical definition from its dimensional drawing. Associate that definition with the appropriate KiCad footprint/component.

### Layout PCB

Place components normally in KiCad.

### Build

```text
./dev build
```

The system reads the PCB and generates the populated mechanical assembly.

### Verify

```text
./dev verify
```

The system checks mechanical constraints.

### Inspect

Open the generated 3D artifact in an appropriate viewer when visual inspection is useful.

### Iterate

Modify PCB, mechanical definitions, or artwork and regenerate. No component positions are manually synchronized between KiCad and MCAD.

---

## 19. V1 Scope

V1 should prove the architecture rather than maximize CAD capability. It should support:

- one KiCad PCB;
- board outline and holes;
- footprint placement and rotation;
- top-side components;
- simplified datasheet-derived component models;
- boxes and cylinders at minimum;
- explicit component coordinate frames;
- component interfaces;
- one mechanically defined enclosure/panel assembly;
- alignment constraints;
- basic collision/clearance checks;
- populated-board 3D generation;
- machine-readable verification;
- CLI execution; and
- hermetic/reproducible repository setup.

A real hardware project should serve as the acceptance fixture.

---

## 20. Deferred Capabilities

Candidates for later increments include:

- bottom-side components;
- complex extrusions;
- STEP import and richer STEP export;
- DXF generation;
- SVG/artwork verification;
- worst-case tolerance analysis;
- automated datasheet ingestion;
- component-library sharing;
- automatic KiCad-footprint association;
- flex PCBs and multiple PCBs;
- cable and wiring envelopes;
- moving assemblies;
- thermal/mechanical relationships;
- fastener modeling;
- fabrication drawing generation; and
- interactive/browser-based inspection.

These should not complicate the v1 data model unnecessarily, but the architecture should not preclude them.

---

## 21. Acceptance Criteria

V1 is successful when a real PCB-based hardware project can demonstrate all of the following:

1. **Clean reproducible build**
   - A fresh checkout can enter the repository-defined environment and build the project without manually installing or configuring backend dependencies.
   - CAD-backend implementation details do not leak into the normal user workflow.
2. **KiCad-derived placement**
   - The tool reads the actual KiCad PCB.
   - Component X/Y position and rotation are derived from KiCad rather than duplicated in the mechanical configuration.
   - Moving a footprint in KiCad causes the corresponding mechanical component to move on the next build.
3. **Datasheet-grounded component**
   - At least one component lacking a trusted manufacturer 3D model is represented using simplified geometry derived from its datasheet.
   - Mechanically significant dimensions have recorded provenance.
4. **PCB/component registration**
   - The relationship between the KiCad footprint and mechanical component model is explicit and deterministic.
   - Pad/component/mechanical-interface relationships do not depend on manual visual positioning.
5. **Generated populated PCB**
   - The tool generates an inspectable 3D representation containing the PCB and its modeled components in their KiCad-defined positions.
6. **External mechanical structure**
   - The project contains at least one enclosure, panel, bracket, or equivalent mechanical structure.
   - Its coordinate relationship to the PCB is explicitly defined.
7. **Interface verification**
   - At least one PCB-mounted component exposes a mechanical interface such as a shaft, connector, switch actuator, display, or mounting feature.
   - The tool numerically verifies its alignment with the corresponding mechanical structure.
8. **Clearance verification**
   - At least one minimum-clearance or collision constraint is evaluated between the populated PCB and surrounding mechanical geometry.
9. **Failure detection**
   - Deliberately moving a footprint in KiCad sufficiently far from its valid position causes mechanical verification to fail.
   - Restoring the valid layout causes verification to pass.
10. **Machine-readable result**
    - Verification produces structured output suitable for CI.
    - Failed required constraints cause a non-zero process exit status.
11. **No interactive CAD dependency**
    - The complete acceptance test can be reproduced without opening or manipulating the design in an interactive MCAD application.

---

## 22. Success Criteria

The project succeeds if mechanical correctness becomes a property that can be **tested from source**, rather than something that must be manually reconstructed and visually inspected in a CAD application.

```text
edit KiCad / component / mechanical source
                  │
                  ▼
             build model
                  │
                  ▼
               verify
                  │
          ┌───────┴───────┐
          │               │
        PASS             FAIL
          │               │
          ▼               ▼
     manufacturable    actionable
       confidence       diagnostics
```

A successful implementation establishes a trustworthy chain:

**manufacturer datasheet → component mechanical definition → KiCad footprint → PCB placement → mechanical assembly → enclosure/interface geometry → verification**

Every transformation in that chain should be reproducible, inspectable, and suitable for automation.

---

## 23. Core Product Invariant

> **No mechanically significant relationship that can be derived from an authoritative source should require manual synchronization between ECAD and MCAD.**

KiCad placement should flow into the mechanical model. Datasheet dimensions should flow into component geometry. Mechanical geometry should flow into verification. A PCB-layout change that invalidates the physical assembly should therefore be detectable automatically.

The 3D model is one representation of that system.

**The verification contract is the product.**
