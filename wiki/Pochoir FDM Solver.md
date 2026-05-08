---
tags: [algorithm, component]
sources: 1
updated: 2026-05-07
---

# Pochoir FDM Solver

Pochoir's finite-difference method (FDM) engine solves the **Laplace equation** (∇²φ = 0) for electric and weighting potentials in LArTPC geometry. Uses iterative Jacobi relaxation on a uniform Cartesian grid.

**Source:** `pochoir/fdm_generic.py`, `pochoir/fdm_numpy.py`, `pochoir/fdm_cumba.py`

## Mathematical basis

Laplace's equation on a uniform N-D grid reduces to the Jacobi stencil:

```
φ(i,j,k) = (1 / 2N) × Σ_{±x,±y,±z} φ(neighbors)
```

where `N` is the number of spatial dimensions. For 3D: each interior point is the mean of its 6 neighbors. Iteration converges when `max(|φ_new − φ_old|) < precision`.

## Edge conditions (`fdm_generic.py`)

Each domain axis face can have one of three boundary conditions:

| Mode | Meaning | Implementation |
|------|---------|----------------|
| `fix` | Dirichlet — boundary value held constant | Copy from `bi` (boundary array) at each iteration |
| `periodic` | Periodic wrap-around | Roll array ±1 and average with wrap |
| `mirror` | Neumann/symmetry — zero gradient | Reflect inner edge outward |

The `edge_condition()` function applies the appropriate BC by padding the working array before computing the stencil. `fix` is used for all PDVD field computations.

## Engines

Five engine backends exist (selected via `--engine` flag):

| Engine | File | Backend | Use case |
|--------|------|---------|---------|
| `numpy` | `fdm_numpy.py` | NumPy (CPU) | Reference, small problems |
| `numba` | `fdm_numba.py` | Numba JIT (CPU) | Fast CPU fallback |
| `torch` | `fdm_torch.py` | PyTorch | GPU or CPU |
| `cupy` | `fdm_cupy.py` | CuPy (GPU) | Simple GPU path |
| `cumba` | `fdm_cumba.py` | CUDA + Numba | Production GPU (PDVD) |

The PDVD field response computation uses `cumba`.

## NumPy engine (`fdm_numpy.py`)

Epoch-based outer loop, inner loop per iteration:

1. Pad working array `arr_pad` with edge-conditioned values
2. Compute stencil: `tmp = (sum of 2N neighbor slices) / (2N)`
3. Restore fixed boundaries: `mutable * tmp + bi` (where `mutable=1` for free cells, `0` for fixed)
4. Track max change: `|arr_new − arr_old|.max()`
5. Stop epoch when `max_change < precision`

Convergence is checked after every iteration.

## cumba engine (`fdm_cumba.py`)

CUDA kernel `stencil_numba3d_jit` (for 3D grids) or `stencil_numba2d_jit` (for 2D):

- **Thread layout**: 3D blocks of 8×8×16 threads
- **In-place update**: `iarr_pad[i,j,k] = bi_pad[i,j,k] + mutable_pad[i,j,k] * tmp_pad[i,j,k]`
  - `bi_pad`: fixed boundary values (0 at free cells)
  - `mutable_pad`: 1 at free cells, 0 at fixed cells
  - `tmp_pad`: stencil average at this iteration
- **Convergence**: sampled every 100 iterations per epoch (vs every iteration in numpy); measured as `cupy.abs(delta).max()`
- **Arrays**: all working arrays are CuPy device arrays; `npz` stores numpy (host-side)

The epoch structure (`--nepochs × --niter`): outer loop runs `nepochs` epochs; each epoch runs up to `niter` Jacobi iterations or stops early if `precision` is reached.

## Key parameters

| Parameter | PDVD 2D/drift | PDVD 3D weighting | Meaning |
|-----------|--------------|-------------------|---------|
| `--nepochs` | 20 | 10 | Number of convergence phases |
| `--niter` | 200,000 | 9,000 | Max iterations per epoch |
| `--precision` | 5×10⁻⁸ | 1×10⁻⁵ | Convergence threshold (max abs change) |
| `--edges` | all `fix` | all `fix` | Boundary condition mode |

## Domain representation (`domain.py`)

`Domain` is an N-D uniform Cartesian grid:

```python
class Domain:
    shape: tuple[int, ...]     # grid dimensions
    spacing: tuple[float, ...]  # cell size per axis (mm)
    origin: tuple[float, ...]   # coordinate of index [0,0,...] (mm)
```

Coordinate of index `i` along axis `k`: `origin[k] + i * spacing[k]`.

`Domain.linspaces()` returns the 1D coordinate arrays per axis. `Domain.gradient(arr)` computes the N-D gradient using `numpy.gradient` with the correct spacing (returns E = −∇φ from potential φ).

Base unit: **mm** (from `pochoir/units.py`: `millimeter = 1`).

## See also

- [[PDVD Field Response Computation]] — how fdm fits into the 10-stage pipeline
- [[Pochoir LAr Physics]] — mobility/diffusion applied to the solved potential
- [[Pochoir Drift Integration]] — drift path ODE integration using the velocity field

## Sources

- [[source-pochoir-source]]
