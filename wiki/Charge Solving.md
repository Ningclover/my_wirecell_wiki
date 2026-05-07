---
tags: [algorithm]
sources: 1
updated: 2026-05-07
---

# Charge Solving

Determines blob charges from wire measurements using LASSO regression.
Part of the [[WireCell Imaging Pipeline Overview]].

## Purpose

After tiling, many blobs are "ghosts" (ambiguous intersections). Charge solving uses
the magnitude of wire signals — not just their existence — to constrain blob charges.
The linear system `m = C·G·b` (measurements = channel-wire-blob associations × charges)
is solved with L1 regularization to drive ghost blobs to zero.

## Mathematical Formulation

Per time slice, given:
- `m` — vector of wire signal measurements (`nmeas` entries)
- `A` — blob-measure association matrix (`nmeas × nblob`), `A[i,j]=1` if blob j covers measure i
- `Σ` — diagonal measurement covariance (`σ_i²`)
- `w` — per-blob regularization weights

Solve (LASSO with Cholesky whitening):
```
minimize ‖U·A·x − U·m‖₂² + λ·Σᵢ wᵢ|xᵢ|
```
where `U` = Cholesky factor of `Σ⁻¹` (whitening transform).

---

## ChargeSolving (`src/ChargeSolving.cxx`)

### Pipeline

```
Input ICluster
  → CS::unpack   — cluster graph → per-slice bipartite subgraphs
  → [Weight → CS::solve → CS::prune] × N_strategies
  → CS::repack   — update cluster graph with solved charges
  → Output ICluster
```

Multiple weighting strategies can be applied in sequence.

### Blob Weighting Strategies

Weight stored in `vtx.value.uncertainty()` (field reuse).

**`uniform`**: all blobs get weight 9.0.

**`simple`**: weight = number of unique slice idents connected via b-b edges + 1.
More connected → lower regularization → harder to zero out.

**`uboone`** (default):
- Base weight 9.0
- If blob connects to a blob with charge > 300 in next slice: ÷3
- If blob connects to a blob with charge > 300 in prev slice: ÷3

| Connectivity | Weight |
|-------------|--------|
| Isolated (no neighbors) | 9.0 |
| One direction only | 3.0 |
| Both prev + next | 1.0 |

Isolated blobs are aggressively regularized to zero → ghost removal.

> **Note**: The 300-charge threshold was previously hardcoded; now configurable as `good_blob_charge_th`.

---

## CSGraph (`src/CSGraph.cxx`)

### CS::unpack

1. Find all blob nodes, group by slice, build per-slice bipartite subgraphs
2. Skip measures with value below threshold or uncertainty above threshold
3. Skip measures with non-positive uncertainty (logs warning)
4. Split each per-slice graph into connected components (each solved independently)

### CS::solve (LASSO core)

1. **Setup**: extract source values, weights, measurements, covariance
2. **Special case** (1 blob): average all measurement values; result ÷ `params.scale`
3. **Build A matrix**: binary association from blob-measure edges
4. **LASSO parameters** (`uboone` config):
   ```
   λ = 1.5 × scale / total_wire_charge
   tolerance = 0.005 × total_wire_charge / (3 × scale × n_blobs)
   max_iter = 100000  (hardcoded; no convergence check)
   ```
5. **Cholesky whitening**: `LLT<mcov>` → triangular solve; whitened: `m̃ = L⁻¹m`, `Ã = scale·L⁻¹·A`
6. **Solve**: `Ress::solve(Ã, m̃, rparams, source, weight)`
7. **Output**: `bvalue.value = solution[i] × scale` (uncertainty not updated from solution)

> **Note**: Uses `L.solve()` (triangular solve) instead of explicit matrix inverse to avoid O(n³) cost.
> Cholesky failure (singular `mcov`) is detected and logged — returns early.

### CS::prune

Copies only blobs above charge threshold; drops measure nodes for removed blobs.

### CS::repack

Reconstructs original cluster graph replacing blob values with solved charges,
omitting pruned blobs and their edges.

---

## BlobSolving (`src/BlobSolving.cxx`)

Simpler standalone variant — same uboone weight scheme (base=9, ÷3 per connection,
checks max 2 slices). Calls `Ress::solve` directly per slice. Does not use CSGraph pipeline.
Uncertainty always set to 0.0 (FIXME noted).

---

## ChargeErrorFrameEstimator (`src/ChargeErrorFrameEstimator.cxx`)

Estimates per-channel charge uncertainty from ROI (Region of Interest) length:
- Pre-computed error waveforms indexed by ROI length
- Applies plane-specific fudge factors
- Three time-position modes (before/between/after `time_limits`)
- Error scales as `sqrt(waveform_value × fudge_factor)`

---

## Known Issues

- LASSO `max_iter = 100000` is hardcoded; no convergence detection
- Blob weight constants (9, 3) and LASSO factors (3/2, 0.005) are undocumented physics choices
- `BlobSolving` always sets uncertainty to 0.0 (FIXME)
- Single-blob path did not apply `params.scale` (fixed), did not initialize `val` (fixed)

## See Also

- [[WireCell Imaging Pipeline Overview]]
- [[Slicing and Tiling]]
- [[Imaging Deghosting]]
- [[Img Bug List]]

## Sources

- [[source-img-examination]]
