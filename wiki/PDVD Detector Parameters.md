---
tags: [concept, experiment]
sources: 1
updated: 2026-04-14
---

# PDVD Detector Parameters

Key physical and electronics parameters for ProtoDUNE-VD, defined in `params.jsonnet` (data) and `simparams.jsonnet` (simulation overrides).

## Geometry

| Parameter | Value |
|-----------|-------|
| APA-CPA center-to-center | 341.55 cm |
| CPA thickness | 50.8 mm |
| APA wire-to-wire gap | 85.725 mm |
| Wire plane gap | 4.76 mm |
| APA grid-to-grid | 114.3 mm |
| Response plane (from collection wires) | 18.1 cm |
| Detector bounds | x: ±3.15 m, y: ±3.42 m, z: 0–3.04 m |
| Anodes | 8 total (0–3: bottom drift, 4–7: top drift) |

Wire geometry file: `protodunevd-wires-larsoft-v3.json.bz2`

## ADC

| Parameter | Value |
|-----------|-------|
| Resolution | 14 bits |
| Bottom full scale | 0.2–1.6 V (1.4 V range) |
| Top full scale | 0–2 V (2.0 V range) |
| Baselines (U, V, W) | 1003.4 mV, 1003.4 mV, 507.7 mV (bottom; reused from ProtoDUNE-SP) |
| nticks | 6000 |

> **Note:** Top drift ADC baselines and full scale are different from bottom — the comment in `params.jsonnet` says "will rewrite top drift values in sim/digitizer and sigproc". The `sp.jsonnet` handles this by computing `ADC_mV` separately for each anode identity.

## Electronics

### Bottom drift (anodes 0–3)
| Parameter | Value |
|-----------|-------|
| Type | Standard shaping electronics |
| Gain | 7.8 mV/fC |
| Shaping time | 2.2 µs |
| Post-gain | 1.1365 |

### Top drift (anodes 4–7)
| Parameter | Value |
|-----------|-------|
| Type | `JsonElecResponse` |
| Response file | `dunevd-coldbox-elecresp-top-psnorm_400.json.bz2` |
| Post-gain | 1.52 (equivalent to ~11 mV/fC) |

## Field response

Current field response: `protodunevd_FR_norminal_260324.json.bz2`  
*(Updated 2026-03-24 by xning — both bottom and top use the same FR file.)*

Previous/alternative: `protodunevd_FR_3view_speed1d55.json.bz2` (commented out)

> **Note:** `ctoffset = 4 µs` in `sp.jsonnet` is explicitly set to be consistent with this field response file.

## Noise spectra

| Drift | File | Version |
|-------|------|---------|
| Bottom | `pdvd-bottom-noise-spectra-v1.json.bz2` | v1 |
| Top | `pdvd-top-noise-spectra-v2.json.bz2` | v2 |

## Simulation overrides (simparams.jsonnet)

| Parameter | Value |
|-----------|-------|
| Drift speed | 1.473 mm/µs |
| Longitudinal diffusion (DL) | 4.0 cm²/s |
| Transverse diffusion (DT) | 8.8 cm²/s |
| Electron lifetime | 1000 ms |
| Simulation mode | Fixed time (LArSoft-compatible) |
| Readout start time (tick 0) | −250 µs (G4 time) |

## Channel layout (funcs.jsonnet)

Each CRP = 2 CRUs × 3072 channels:
- **CRU0** (even anode index within CRP): U=[0–475, 952–1427], V=[1904–2487]
- **CRU1** (odd anode index within CRP): U=[476–951, 1428–1903], V=[2488–3071]
- CRP N uses channels: `[x + 3072*N for x in CRU0 or CRU1]`

## See also

[[ProtoDUNE-VD WireCell Configuration Overview]], [[PDVD Noise Filtering Configuration]], [[PDVD Signal Processing Configuration]], [[Detector-Specific Signal Processing]]

## Sources

- [[source-pdvd-wct-config]]
