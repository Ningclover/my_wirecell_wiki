---
tags: [concept, experiment]
sources: 1
updated: 2026-04-29
---

# pdhd Detector Parameters

Key physical and electronics parameters for ProtoDUNE-HD, defined in `toolkit/cfg/pgrapher/experiment/pdhd/params.jsonnet` and `simparams.jsonnet`.

## Geometry

| Parameter | Value |
|-----------|-------|
| APA-CPA center-to-center | 357.34 cm |
| CPA thickness | 3.175 mm (1/8″) |
| APA wire-to-wire gap (`apa_w2w`) | 85.87 mm |
| Wire plane gap (`plane_gap`) | 4.76 mm |
| APA grid-to-grid (`apa_g2g`) | `apa_w2w + 6×plane_gap` = 114.43 mm |
| Anode-cut plane offset (`apa_plane`) | `0.5×apa_g2g − plane_gap` = 52.46 mm (at first induction wires) |
| Response plane (from collection wires) | 10 cm |
| Anodes | 4 (2 + 2 mirrored across central CPA) |

Wire geometry file: `protodunehd-wires-larsoft-v1.json.bz2`

> **Note:** the comment in `params.jsonnet:32` notes that the anode-cut plane is placed at the first induction wires, not at the grid-wire midplane. Depos closer to the wires than this are simply discarded (no backup).

### Per-plane numbers

Each face has 1148 U + 1148 V + 480 W = **2776 wires** total. Each anode has two faces.

| Plane | Wire direction (face 0, anode 0) | Pitch | Wires per face |
|-------|-----------------------------------|-------|----------------|
| U | `(0, +0.812, +0.584)` (≈ ±35.7° from y-axis) | 4.669 mm | 1148 |
| V | `(0, +0.812, −0.584)` | 4.669 mm | 1148 |
| W | `(0, +1.000, 0)` (vertical) | 4.79 mm | 480 |

Cosines: `cos(35.7°) = 0.8120`, `sin(35.7°) = 0.5836` — appears as the ±0.584 z-component of U/V wire directions.

### Faces

Per anode, the two faces are at slightly different x (separated by ~100 mm in the wire-plane stack), with **opposite-sign U/V handedness**. Within `Anode.faces[0..1]` the cathode-facing face position is:

| Anode | `Anode.faces[i]` to use | x of that face's W plane (mm) |
|-------|--------------------------|-------------------------------|
| 0 | 0 | −3532 |
| 1 | 1 | +3530 |
| 2 | 0 | −3532 |
| 3 | 1 | +3530 |

The non-cathode-facing face sits at the outer wall, e.g. anode 0 face 1 W is at x = −3618 mm.

## Active drift volume / x-coordinate layout

`pdhd/params.jsonnet:55–80` builds per-anode drift volumes. With `n` the anode index and `sign = 2*(n%2)−1`, `centerline = sign × apa_cpa`. For anode 0 (`sign = −1`):

```
centerline   = −357.34 cm
apa_plane offset (toward cathode) = +5.246 cm
res_plane offset                  = +14.29 cm   (= 0.5×apa_w2w + 10 cm)
cpa_plane offset                  = +357.18 cm  (= apa_cpa − 0.1588 cm)
```

| Plane (anode 0) | x (cm) |
|-----------------|--------|
| Wire plane (W collection, cathode-facing face) | −353.20 |
| Anode-cut plane | −352.09 |
| Response plane | −343.05 |
| Cathode plane | −0.16 |

> **Constraint:** depos outside `[anode_plane, cathode_plane]` are discarded before drifting. The response plane is the natural starting point for clean field response (`params.jsonnet:23–32`).

## ADC and electronics

| Parameter | Value |
|-----------|-------|
| ADC resolution | 14 bits |
| ADC baselines (U, V, W) | 1003.4, 1003.4, 507.7 mV (reused from ProtoDUNE-SP) |
| ADC full scale | [0.2, 1.6] V |
| FE shaping time | 2.2 µs |

### `elecGain` external variable

`pdhd/params.jsonnet:114` reads the FE amplifier gain from a jsonnet external variable:

```jsonnet
elecs: [
  super.elec {
    gain : std.parseJson(std.extVar("elecGain")) * wc.mV / wc.fC,
    shaping : 2.2 * wc.us,
  } for n in std.range(0, 3)
],
```

Pass at the wire-cell command line as **a string** (`std.parseJson` parses it as JSON):

```bash
wire-cell -V elecGain=14   # default
wire-cell -V elecGain=7.8  # low-gain runs
```

For raw `jsonnet` evaluation outside `wire-cell`, use `--ext-str elecGain=14` (not `--ext-code`, which would yield a number that `std.parseJson` rejects with `Unexpected type number, expected string`).

The noise spectrum file is auto-selected from this gain in `params.jsonnet:169–170`:

```jsonnet
noise: if $.elec.gain > 8*wc.mV/wc.fC then "protodunehd-noise-spectra-14mVfC-v1.json.bz2"
       else "protodunehd-noise-spectra-7d8mVfC-v1.json.bz2"
```

## Simulation overrides

From `pdhd/wct-sim-check.jsonnet:11–18`:

| Parameter | Value |
|-----------|-------|
| Drift speed | 1.565 mm/µs |
| Longitudinal diffusion (DL) | 6.2 cm²/s |
| Transverse diffusion (DT) | 16.3 cm²/s |
| Electron lifetime | 50 ms |
| Simulation mode | Fixed time (`fixed: true`) |

## NF input/output frame tags

`pdhd/nf.jsonnet` `OmnibusNoiseFilter`:

- `intraces: ''` — wildcard, reads **all traces** in the input frame regardless of trace tag (so no per-anode frame `Retagger` is needed between sim and NF, unlike PDVD where `nf.jsonnet` hardcodes `intraces: 'orig'`).
- `outtraces: 'raw%d' % n` — output trace tag is `raw<n>` where `n` is the anode index.

Sim digitizer output trace tag is `'orig'` (single name, not per-anode); the digitizer also produces per-anode frame tag `'orig%d' % n` (`pdhd/sim.jsonnet:25, 74`).

## Channel layout

Each anode has 2560 channels (2 faces × 1280) in **local 0-based numbering**:

| Plane | Channel range | Count |
|-------|---------------|-------|
| U | 0 – 799 | 800 |
| V | 800 – 1599 | 800 |
| W | 1600 – 2559 | 960 |

(800 = 2 faces × 400 wrapping U wires; 960 = 2 faces × 480 W wires.)

When multiple anodes are stored in one frame, channels are offset by `2560 × anode_index`.

## See also

[[WireCell Wires Schema]], [[WireCell Sim Track Conventions]], [[PDVD Detector Parameters]], [[Detector-Specific Signal Processing]]

## Sources

- [[source-session-2026-04-29-track-sim-geometry]]
