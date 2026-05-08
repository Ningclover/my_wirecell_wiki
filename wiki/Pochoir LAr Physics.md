---
tags: [algorithm, concept]
sources: 1
updated: 2026-05-07
---

# Pochoir LAr Physics

Physics models for liquid argon electron transport used by the `pochoir` field response toolkit. Implements the BNL LAr mobility polynomial and the Einstein-relation diffusion coefficients.

**Source:** `pochoir/lar.py`

## Electron mobility model

Based on the BNL polynomial formula from lar.bnl.gov. Maps electric field magnitude `E` (kV/cm) and temperature `T` (K) to drift mobility µ (cm²/(kV·µs)):

```python
def mobility_function(E, T=89):
    a0, a1, a2, a3, a4, a5 = 551.6, 7158.3, 4440.43, 4.29, 43.63, 0.2053
    T0 = 89  # K
    denom = 1 + (a1/a0)*E + a3*E**2 + a5*E**3
    numer = a0 + a1*E + a2*E**2 + a4*E**3
    Tc = T0/T  # temperature correction factor
    return numer/denom * Tc
```

The six coefficients `a0..a5` define the rational polynomial approximation. Temperature correction is linear (`T0/T`). The function is vectorized via NumPy and accepts array inputs.

**Units**: E in kV/cm → µ in cm²/(V·s) equivalent (the µs unit makes velocity = µ·E in mm/µs when E is in kV/cm).

## Diffusion coefficients

Computed from the mobility via the **Einstein relation**: `D = (k_B T / q) × µ`. Pochoir separates longitudinal and transverse components.

### Longitudinal diffusion

```python
def longitudanal_diffusion(E, T=89):
    mu = mobility_function(E, T)
    dmu_dE = gradient(mu, E)   # dµ/dE
    kT = boltzmann * T / eplus  # thermal voltage (V)
    return mu * kT + E * kT * dmu_dE
```

The extra `E × dmu/dE` term arises because longitudinal diffusion along the drift direction couples to the field gradient: `D_L = µ kT (1 + E/µ × dµ/dE)`.

### Transverse diffusion

```python
def transverse_diffusion(E, T=89):
    mu = mobility_function(E, T)
    kT = boltzmann * T / eplus
    return mu * kT
```

Transverse diffusion is the standard Einstein relation: `D_T = µ kT`. No field-gradient term since transverse motion is perpendicular to E.

**Output units**: cm²/s (same convention as WireCell `TrackFitting::Parameters` DL/DT fields).

## Constants (`pochoir/units.py`)

| Symbol | Value | Meaning |
|--------|-------|---------|
| `millimeter` | 1 | Base length unit |
| `nanosecond` | 1 | Base time unit |
| `boltzmann` | 8.617e-5 eV/K | Boltzmann constant |
| `eplus` | 1 | Electron charge (natural units) |
| `volt` | 1e-6 MeV/eplus | Voltage unit |
| `kelvin` | 1 | Temperature unit |

The mm/ns base means velocity in mm/µs = mm/(1000 ns) requires explicit 1e3 factors in some places.

## Velocity field computation (`pochoir velo`)

Pipeline: `potential/drift3d.npz` → velocity field:

1. `Domain.gradient(potential)` → E-field components `[Ex, Ey, Ez]` = −∇φ
2. Zero out boundary-adjacent cells (1-cell border in each direction) to remove edge artifacts
3. `vmag(E)` → scalar field magnitude `|E|`
4. `mobility_function(|E|, T)` → µ (mm²/(kV·µs)) — vectorized over entire domain
5. `velocity = µ × E` component-wise → 3D velocity field in mm/µs

Output: `velocity/drift3d.npz` — shape `(3, Nx, Ny, Nz)`, units mm/µs.

## Comparison to WireCell TrackFitting defaults

The `pochoir lar.py` model and WireCell's `TrackFitting::Parameters` use compatible coefficient sets but different field contexts:

- `pochoir` uses BNL polynomial at runtime (T=89 K, PDVD nominal field)
- `TrackFitting` stores pre-computed DL=6.4 cm²/s, DT=9.8 cm²/s (MicroBooNE defaults at ~0.27 kV/cm)
- PDVD simulation overrides: DL=4.0, DT=8.8 cm²/s (different E-field)

See [[Track Fitting and Calorimetry]] and [[PDVD Detector Parameters]] for the WireCell usage.

## See also

- [[Pochoir FDM Solver]] — potential solution that feeds into velocity computation
- [[Pochoir Drift Integration]] — uses the velocity field for electron path tracing
- [[PDVD Field Response Computation]] — full pipeline context

## Sources

- [[source-pochoir-source]]
