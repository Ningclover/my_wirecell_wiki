---
tags: [concept, component]
sources: 1
updated: 2026-04-29
---

# WireCell Wires Schema

JSON file format describing the wire geometry of a LArTPC for WireCell. Used by the Wires service to instantiate `IAnodePlane` / `IWirePlane` / `IWire` objects from a single `*-wires-larsoft-v*.json.bz2` file.

## Purpose

A single source of truth for the per-detector geometry of every readout wire: its endpoints (in detector coordinates), the channel it maps to, and which anode/face/plane it belongs to. Loaded once at job start and shared across all WireCell components that need wire-pixel correspondences.

## Top-level structure

```json
{
  "Store": {
    "detectors": [],          // optional, often empty
    "anodes":  [{"Anode":  {...}}, ...],
    "faces":   [{"Face":   {...}}, ...],
    "planes":  [{"Plane":  {...}}, ...],
    "wires":   [{"Wire":   {...}}, ...],
    "points":  [{"Point":  {...}}, ...]
  }
}
```

Each list element is wrapped in a single-key dictionary keyed by the type name (`Anode`, `Face`, `Plane`, `Wire`, `Point`). The inner dict is the actual record.

### Per-record fields

| Record | Fields |
|--------|--------|
| `Anode` | `ident`, `faces: [<list-positions>]` |
| `Face`  | `ident`, `planes: [<list-positions>]` |
| `Plane` | `ident`, `wires: [<list-positions>]` |
| `Wire`  | `ident`, `channel`, `segment`, `tail: <point list-position>`, `head: <point list-position>` |
| `Point` | `x`, `y`, `z` (mm, detector coordinates) |

## Critical convention: list position vs. ident

Cross-references between records (`anode.faces`, `face.planes`, `plane.wires`, `wire.tail/head`) are **list positions in the corresponding `Store.<type>` array**, not the inner `ident` field.

The `ident` field is local — for example, `Face.ident` only takes values 0 or 1 (the two faces of an anode), and `Plane.ident` only takes 0/1/2 (U/V/W within a face). These idents repeat across faces/planes; they are not unique. To follow a reference like `anode.faces[i]`, index the global `Store.faces` list at position `i`.

> **Pitfall:** building dicts keyed by `ident` (e.g. `{f['Face']['ident']: f for f in store['faces']}`) collapses 16 VD faces or 8 HD faces into 2 entries and silently breaks all downstream lookups.

## Per-detector record counts

| Detector | anodes | faces | planes | wires |
|----------|--------|-------|--------|-------|
| ProtoDUNE-VD | 8  | 16 | 48 | 13 840 |
| ProtoDUNE-HD | 4  | 8  | 24 | 22 208 |

Pattern: `#faces = 2 × #anodes`, `#planes = 3 × #faces` (U/V/W per face), `#points = 2 × #wires` (each wire has a tail and head endpoint).

## Plane order within a face

`Face.planes[0..2]` correspond to the **U, V, W** planes in that order (induction-1, induction-2, collection).

## Coordinate units and conventions

- All `Point` coordinates are in **mm**, in the global detector coordinate frame.
- A wire is a line segment from `points[wire.tail]` to `points[wire.head]`.
- The wire direction (a unit vector) is `(head − tail) / |head − tail|`.
- The wire pitch direction within a plane is `wire_dir × x̂` (perpendicular to wires, in the y-z drift-anode plane).

## Representative numbers per plane

| Detector | Plane | Wire direction (face 0) | Pitch | Wires per face |
|----------|-------|--------------------------|-------|----------------|
| VD | U | `(0, +0.500, +0.866)` (60° from +y in y-z) | 7.65 mm | 286–287 |
| VD | V | `(0, +0.500, −0.866)` | 7.65 mm | 286–287 |
| VD | W | `(0, +1.000, 0)` (vertical) | 5.10 mm | 292 |
| HD | U | `(0, +0.812, +0.584)` (≈35.7°) | 4.669 mm | 1 148 |
| HD | V | `(0, +0.812, −0.584)` | 4.669 mm | 1 148 |
| HD | W | `(0, +1.000, 0)` | 4.79 mm | 480 |

For VD anodes 4–7 (positive-x side) the U/V handedness flips: `U=(0, +0.5, −0.866)`, `V=(0, +0.5, +0.866)`. For HD, **face 0 and face 1 within the same anode have opposite-sign U/V handedness** — this matters when picking which face's plane to use for a track parallel to the wire plane (see [[WireCell Sim Track Conventions]]).

## Identifying the cathode-facing face (HD)

For HD, the two faces of an anode sit at slightly different x (separated by ≈ 100 mm in the wire-plane stack). The face with the smaller `|x|` (closer to x=0, i.e. closer to the central CPA) is the active readout face for that anode.

| Anode | Cathode-facing face position in `Anode.faces[]` | x of W plane (mm) |
|-------|-------------------------------------------------|---------------------|
| 0 | 0 | −3532 |
| 1 | 1 | +3530 |
| 2 | 0 | −3532 |
| 3 | 1 | +3530 |

For VD, both faces of an anode share the same x (single readout side); face 0 is conventional.

## Tooling note

The `wirecell-util wires-info` and `wirecell-util wires-volumes` Python utilities print human-readable summaries of these files; they are the canonical way to inspect a wires JSON without writing a parser.

## See also

[[PDVD Detector Parameters]], [[pdhd Detector Parameters]], [[WireCell Sim Track Conventions]]

## Sources

- [[source-session-2026-04-29-track-sim-geometry]]
