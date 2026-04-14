# Track Fitting and Calorimetry

`TrackFitting` is a utility class (not a WireCell component — it has no `configure()` or `execute()` interface). It is instantiated by algorithms inside the clus pipeline to perform local track fitting and calorimetric energy measurement.

## Parameters struct

```cpp
struct TrackFitting::Parameters {
    double diffusion_L;    // longitudinal diffusion coefficient (cm²/µs)
    double diffusion_T;    // transverse diffusion coefficient (cm²/µs)
    double sigma_x;        // position uncertainty in drift direction (cm)
    double sigma_yz;       // position uncertainty in wire-plane directions (cm)
    double charge_sigma;   // per-blob charge uncertainty (electrons)
};
```

These match the detector simulation parameters (e.g., for PDVD: DL=4.0 cm²/s, DT=8.8 cm²/s from `simparams.jsonnet`).

## Wire → space projection

WireCell imaging blobs carry charge on wire segments, not 3D points. `TrackFitting` projects each wire-segment charge measurement into a 3D space point using:

- Wire geometry (angle, pitch, offset)
- Drift velocity and T0 correction
- The current track direction hypothesis (iterative refinement)

## Fitting modes

### Single fitting mode

Assumes negligible multiple scattering. Minimizes:

```
χ² = Σ_i [ (x_i - x_fit(s_i))² / σ_x² + (yz_i - yz_fit(s_i))² / σ_yz² ]
```

where `s_i` is the arc length parameter along the fitted line. Used for straight muon tracks far from the vertex.

### Multiple fitting mode

Adds a multiple-scattering term between successive measurement points. Used for lower-momentum tracks and near the neutrino vertex where scattering is significant.

The Kalman-filter-like update propagates position and direction uncertainties point by point.

## dQ/dx measurement

At each fitted point along the track:

1. Identify blobs within a cylinder of radius `dqdx_radius` centered on the fitted position
2. Sum their charges weighted by distance (Gaussian weight with σ = `charge_sigma`)
3. Divide by the local path length `ds` between fitted points → dQ/dx in electrons/cm

## 2D charge map caching

For efficiency, `TrackFitting` caches a 2D map: (wire index, tick index) → charge. This avoids re-projecting the same blobs for every dQ/dx evaluation point along a segment. The cache is invalidated when the track direction hypothesis changes by more than `direction_cache_threshold` radians.

## IRecombinationModel

Converts dQ/dx → dE/dx. The default implementation is the **Box model** (used for MicroBooNE and derived detectors):

```
dE/dx = (exp(dQ/dx · W_ion · β' / (ρ · E · α)) - α) / (β' / (ρ · E))
```

Parameters (MicroBooNE defaults):
- `α = 0.93` (A constant)
- `β' = 0.207` (B/ρ constant, cm/kV)
- `ρ = 1.396 g/cm³` (LAr density)
- `E` = electric field (kV/cm)
- `W_ion = 23.6 eV` (ionization work function)

## Calorimetric energy integration

After dE/dx is computed at each point:

```
KE = ∫ dE/dx · ds  (along track)
```

`cal_kine_charge()` sums this integral over all fitted points, returning kinetic energy in MeV. The result feeds [[Track Shower Separation]] kinematics and [[Neutrino Taggers]] energy cuts.

## ParticleDataSet

A detector-agnostic lookup table of dE/dx vs kinetic energy (and range-energy) for each particle hypothesis (µ, π, p, e, γ). Used for:

- Comparing measured dE/dx profile to hypothesis (PID)
- Converting measured range to kinetic energy for stopping tracks

## See also

- [[WireCell Clus Pipeline Overview]]
- [[Clus Data Structures]]
- [[Track Shower Separation]]
- [[Particle Identification]]
- [[PDVD Signal Processing Configuration]]
