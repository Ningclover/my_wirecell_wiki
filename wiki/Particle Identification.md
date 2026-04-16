---
tags: [algorithm]
sources: 1
updated: 2026-04-16
---

# Particle Identification

Particle identification (PID) in the clus module assigns PDG codes to `PR::Segment` objects and propagates directions from the neutrino vertex outward.

## is_shower predicate

A segment is classified as a shower if any of three conditions holds:

1. `kShowerTrajectory = true` — KS-test on dQ/dx profile failed MIP hypothesis (see [[Track Shower Separation]])
2. `kShowerTopology = true` — RMS transverse spread exceeds threshold
3. Parent segment is a shower AND the opening angle to the parent < `shower_cone_angle`

Condition 3 propagates shower identity along near-collinear daughters (delta rays, Bremsstrahlung).

## separate_track_shower

See [[Track Shower Separation]] for full details. PID uses the resulting `is_shower` flags to partition segments before direction assignment.

## determine_direction dispatch table

Each segment is assigned a propagation direction (toward or away from the neutrino vertex) by `determine_direction()`, which dispatches based on particle type:

| Hypothesis | Criteria | Direction rule |
|------------|----------|---------------|
| Muon | `!is_shower`, long (> 10 cm), MIP dQ/dx | Away from vertex |
| Proton | `!is_shower`, short (< 10 cm), high dQ/dx near start | Away from vertex |
| Pion | `!is_shower`, medium length, MIP + Bragg peak | Away from vertex |
| Electron | `is_shower`, single segment | Away from vertex |
| Photon | `is_shower`, gap between shower start and vertex | Gap segment assigned inward |
| Unknown | Fallback | Away from vertex |

## fix_maps_* and improve_maps_* consistency passes

Two families of passes enforce global direction consistency across the full `"pr"` graph:

- **fix_maps_***: Detect and correct segments whose `dirsign` contradicts the direction implied by their parent segment at their shared vertex. Hard corrections (flip `dirsign`).
- **improve_maps_***: Softer heuristic improvements — e.g., if three segments meet at a vertex and two agree, the third is flipped.

These passes run after the initial `determine_direction` assignment and before `examine_direction`.

## examine_direction: BFS propagation

The final PID step is a BFS (breadth-first search) propagation starting from the neutrino vertex:

1. Set the neutrino vertex segment directions as "anchored"
2. BFS expands outward along the `"pr"` graph
3. At each visited segment, apply PDG re-assignment rules based on the local context:
   - If a segment starts as MIP and its dQ/dx rises toward the endpoint → reclassify as proton (Bragg peak)
   - If a shower segment has a preceding gap segment → reclassify the gap as photon, set `dirsign` inward
   - If a short segment branches off a long MIP → reclassify as delta ray (sub-threshold shower)
4. PDG codes are written to `PR::Segment` and propagated to associated `PR::Shower` objects

## Shower grouping maps

After BFS propagation:

- `shower_group_map`: segment → `PR::Shower` index
- `shower_blob_map`: blob → `PR::Shower` index

These maps are used by [[Neutrino Taggers]] (especially NuE and Pi0) to count showers, measure shower energies, and identify the primary shower.

## See also

- [[WireCell Clus Pipeline Overview]]
- [[Track Shower Separation]]
- [[Neutrino Vertex Determination]]
- [[Track Fitting and Calorimetry]]
- [[Neutrino Taggers]]

## Sources

- [[source-clus-examination]]
