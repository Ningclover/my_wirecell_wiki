---
tags: [algorithm, experiment]
sources: 1
updated: 2026-04-14
---

# PDVD Imaging Configuration

Wire-cell 3D imaging pipeline for ProtoDUNE-VD, configured in `img.jsonnet`. Runs after [[PDVD Signal Processing Configuration]] to convert deconvolved charge into 3D blobs and clusters.

**Entry point:** `wcls-nf-sp-img.jsonnet`

## Pipeline (per anode)

```
SP output (gauss/wiener frame)
    │
    ▼
pre_proc
  ├── CMMModifier           organize dead channel ranges
  ├── FrameMasking          mask gauss/wiener traces by "bad" channel map
  └── ChargeErrorFrameEstimator   gauss→gauss_error (charge uncertainty)
    │
    ▼
slicing (MaskSlices)         gauss+wiener → ISlice stream, tick_span=4
    │
    ▼
tiling                       ISlice → IBlobSet (per-face GridTiling, both faces)
  ├── SliceFanout (2-way)
  ├── GridTiling face 0
  ├── GridTiling face 1
  └── BlobSetSync
    │
    ▼
solving                      IBlobSet → ICluster
  ├── BlobClustering         policy: "uboone"
  ├── ChargeSolving (1st)    weighting: "uniform", solve_config: "uboone"
  ├── LocalGeomClustering
  ├── ChargeSolving (2nd)    weighting: "uboone"
  └── GlobalGeomClustering   policy: "uboone"
    │
    ▼
ClusterFileSink              output: clusters-apa-{name}.tar.gz (json format)
```

## Pre-processing details

### ChargeErrorFrameEstimator
- Uses `microboone-charge-error.json.bz2` (WaveformMap)
- `rebin: 4` — must match the waveform_map choice
- `fudge_factors: [2.31, 2.31, 1.1]` per plane [U, V, W]
- `time_limits: [12, 800]` ticks

### CMMModifier
- Single dead channel region: channels 0–8500
- Organizes contiguous bad channel ranges

## Slicing parameters

| Parameter | Value |
|-----------|-------|
| `tick_span` | 4 ticks |
| `min_tbin` | 0 |
| `max_tbin` | 8500 |
| Active planes | [0, 1, 2] (all three) |
| `nthreshold` | [3.6, 3.6, 3.6] (signal threshold per plane) |

## Tiling

Two faces per anode processed in parallel; results merged via `BlobSetSync`.  
`nudge: 1e-2` applied in GridTiling to avoid degenerate geometry.

## Solving (charge solving pipeline)

Uses simplified pipeline:
```
BlobClustering → ChargeSolving(uniform) → LocalGeomClustering → ChargeSolving(uboone) → GlobalGeomClustering
```
(Full pipeline with deghosting stages — `ProjectionDeghosting`, `InSliceDeghosting` — is commented out.)

- `whiten: true` in both ChargeSolving stages
- `dryrun: false` in LocalGeomClustering

## Multi-slicing options

`img.jsonnet` supports several slicing strategies (not all used in the default `wcls-nf-sp-img.jsonnet`):
- `single` — single pass, 3 active planes, tick_span=4 (default in per_anode)
- `active` — multi-pass with 3/2-plane combinations (tick_span=4)
- `masked` — masked 2-view slicing (tick_span=500)
- combined active+masked fork

## See also

- [[ProtoDUNE-VD WireCell Configuration Overview]]
- [[PDVD Signal Processing Configuration]]
- [[WireCell Clus Pipeline Overview]] — downstream consumer of imaging blobs

## Sources

- [[source-pdvd-wct-config]]
