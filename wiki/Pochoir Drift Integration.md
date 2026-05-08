---
tags: [algorithm, component]
sources: 1
updated: 2026-05-07
---

# Pochoir Drift Integration

Electron drift path integration in `pochoir`. Traces electron trajectories through the drift velocity field to compute weighting-potential waveforms for each strip.

**Source:** `pochoir/drift_numpy.py`, `pochoir/arrays.py`, `pochoir/__main__.py`

## Simple class (deterministic drift)

`Simple` in `drift_numpy.py` wraps a `RegularGridInterpolator` over the 3D velocity field:

```python
class Simple:
    def __init__(self, vfield, domain):
        # Build one RGI per velocity component (Vx, Vy, Vz)
        self.rgi_x = RegularGridInterpolator(domain.linspaces(), vfield[0], ...)
        self.rgi_y = RegularGridInterpolator(domain.linspaces(), vfield[1], ...)
        self.rgi_z = RegularGridInterpolator(domain.linspaces(), vfield[2], ...)

    def velocity(self, t, pos):
        # Returns [Vx, Vy, Vz] at position pos = [x, y, z]
```

`RegularGridInterpolator` uses linear interpolation by default (trilinear for 3D grids).

## Deterministic path integration (`solve`)

```python
def solve(simple, start, t_span, t_eval):
    result = scipy.integrate.solve_ivp(
        simple.velocity,
        t_span,
        start,
        method='Radau',
        t_eval=t_eval,
        rtol=1e-10,
        atol=1e-10,
    )
    return result.y  # shape (3, N_timesteps)
```

- **Method**: `Radau` (implicit Runge-Kutta, A-stable, good for stiff ODEs)
- **Tolerances**: `rtol=atol=1e-10` — tight, suitable for precise weighting-potential sampling
- **`t_eval`**: explicit output times at 0.1 µs intervals from 0 to 300 µs
- Output: position array `[x(t), y(t), z(t)]` at each evaluation time

`pochoir drift` calls `solve` once per start point and saves all paths to `paths/drift3d.npz`.

## Stochastic path integration (`solve_sde`)

`solve_sde` adds diffusion noise via **Euler-Maruyama** stepping:

```python
def solve_sde(simple, start, t_span, dt, DL, DT, T=89):
    pos = start.copy()
    positions = [pos.copy()]
    t = t_span[0]
    while t < t_span[1]:
        v = simple.velocity(t, pos)
        drift_direction = v / |v|
        # Anisotropic noise: DL along drift, DT perpendicular
        dW = sqrt(dt) * randn(3)
        noise = DL * drift_dir * dW_longitudinal + DT * perp_dir * dW_transverse
        pos += v * dt + noise
        t += dt
    return positions
```

Diffusion is anisotropic: `DL` along the local drift direction, `DT` perpendicular. Used for diffusion-smeared path ensembles (not used in the PDVD FR production script).

## `rgi` utility (`arrays.py`)

```python
def rgi(domain, arr, method='linear'):
    return RegularGridInterpolator(domain.linspaces(), arr, method=method)
```

Convenience wrapper for building interpolators — used by `Simple` and also by `induce`/`induce-30deg` to sample the weighting potential along each path.

## Induced current calculation (`__main__.py induce`)

After paths are computed, `pochoir induce` and `pochoir induce-30deg` evaluate the induced current:

1. For each path `pos(t)`:
   - Sample weighting potential `W(pos(t))` via `rgi` trilinear interpolation on `weight3dextend`
   - `Q(t) = charge × W(t)` (charge = 1 electron, W in Volts/Volt = dimensionless)
   - `I(t) = dQ/dt` (numpy gradient with dt spacing)
2. Mirror across strips: shift X by `n × dx` for strip `n` (`dx=7.615 mm` for U/V, `5.1 mm` for W)
3. Average `--average 10` paths per output waveform (Y-position averaging within a strip)

### `induce-30deg` specifics

The 30° induction strips have two orientations alternating by strip index:

- Even strip (index 0, 2, 4, ...): Y offset = **−1.45 mm**
- Odd strip (index 1, 3, 5, ...): Y offset = **+1.45 mm**

The 60 start paths are split into two halves (paths 0..29 and 30..59). For even strips the first half is used; for odd strips the second half with the alternated Y offset. Strip spacing: `dx = 7.615 mm`.

## WireCell format conversion (`convertfr`)

`pochoir convertfr` builds the `FieldResponse` schema objects:

- Reads induced current arrays for all three planes
- Truncates each waveform to **1325 samples** (= 132.5 µs at 100 ns period)
- Flips sign: stored current = `−1 × computed_current` (sign convention for Ramo-theorem induction)
- Builds `PathResponse(current, pitchpos, wirepos)` per path per strip
- Aggregates into `PlaneResponse(paths, planeid, location, pitch)` per plane
- Wraps in `FieldResponse(planes, axis, origin, tstart, period, speed)`:
  - `origin=181.0 mm`, `speed=0.00153 mm/ns`, `period=100 ns`
- Serializes to bz2-compressed JSON via `persist.dumpfr`

## Path start point layout

Two start-point grids, both at Z = 198 mm (just inside drift volume):

**U/V paths** (`starts/drift3d`): 120 points = 6 X-positions × 10 Y-positions × 2 orientations
- X (mm): 0.05, 0.52, 0.81, 1.28, 1.56, 2.32 (covers one 30° half-strip unit cell)
- Y (mm): 0.05 to 1.31, 10 points spaced 0.14 mm

**W paths** (`starts/drift3d_w`): 60 points = 6 X-positions × 10 Y-positions
- X (mm): 0.05, 0.51, 1.02, 1.53, 2.04, 2.54 (covers one 90° half-strip unit cell)

## See also

- [[Pochoir FDM Solver]] — provides the drift potential field
- [[Pochoir LAr Physics]] — provides the drift velocity from potential
- [[PDVD Field Response Computation]] — full 10-stage pipeline context

## Sources

- [[source-pochoir-source]]
