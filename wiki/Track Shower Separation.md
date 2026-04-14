# Track Shower Separation

`separate_track_shower()` classifies each `PR::Segment` as either a **track** or a **shower** using two independent tests. A segment is a shower if either test fires.

## Test 1: Topology (RMS transverse spread)

Measures how wide the charge deposition is transverse to the segment axis:

- Project all blob centers onto the segment direction vector
- Compute the RMS of the transverse (perpendicular) distances
- **Threshold**: if RMS spread > topology_threshold → `kShowerTopology = true`

Physical basis: EM showers produce broad, diffuse charge; MIP tracks are narrow and linear.

## Test 2: Trajectory (KS-test on dQ/dx profile)

Compares the dQ/dx profile of a segment against a reference MIP distribution:

- Sample dQ/dx at evenly spaced points along the segment using [[Track Fitting and Calorimetry]]
- Run a Kolmogorov-Smirnov test between the segment's dQ/dx CDF and the MIP reference CDF
- **Threshold**: if KS statistic > ks_threshold → `kShowerTrajectory = true`

Physical basis: showers have rising dQ/dx toward the shower maximum; MIPs are flat.

## is_shower predicate

A segment is classified as a shower if:

```
is_shower = kShowerTrajectory OR kShowerTopology OR (parent_segment.is_shower AND angle_to_parent < shower_cone_angle)
```

The third condition propagates shower identity along nearly-collinear children (delta rays, Bremsstrahlung photons).

## Shower clustering: shower_clustering_with_nv()

After per-segment classification, shower segments are grouped into `PR::Shower` objects:

1. Start from the neutrino vertex (or the nearest track endpoint if no NV identified)
2. Breadth-first expansion: collect contiguous shower segments + any isolated blobs within `shower_grouping_radius`
3. Blobs not reachable by tracks or showers are assigned to the nearest shower

## Pi0 identification

Two-shower topology check within the shower grouping:

- If two `PR::Shower` objects are found with an opening angle consistent with π⁰ → γγ decay (~135 MeV invariant mass window)
- Both showers must have `is_shower = true` with no MIP segment between them
- Sets a `kPi0` flag on the neutrino vertex

## Kinematics

After shower grouping, energy is summed:

- **Track energy**: dQ/dx integrated along track length → dE/dx via [[Track Fitting and Calorimetry]] recombination model → calorimetric kinetic energy
- **Shower energy**: `cal_kine_charge()` — sum all blob charges in the shower, apply recombination correction, convert to GeV using W_ion

## See also

- [[WireCell Clus Pipeline Overview]]
- [[Pattern Recognition PR Loop]]
- [[Track Fitting and Calorimetry]]
- [[Particle Identification]]
- [[Neutrino Taggers]]
