---
tags: [concept, component]
sources: 1
updated: 2026-04-29
---

# WireCell Sim Track Conventions

Conventions and constraints for placing simulated tracks (line-of-charge depositions) so that they produce expected signals in the WireCell simulation pipeline.

## Track depo construction

The standard simulated-track config (e.g. `pgrapher/experiment/{protodunevd,pdhd}/wct-sim-check.jsonnet`) builds depos via `sim.tracks(tracklist, step=0.1*wc.mm)`:

```jsonnet
local tracklist = [{
    time: 0 * wc.us,
    charge: -500,                     // negative ⇒ N electrons per step
    ray: { tail: wc.point(...), head: wc.point(...) },
}];
local depos = sim.tracks(tracklist, step=0.1 * wc.mm);
```

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `charge: -500` | −500 | When negative, magnitude is electrons per `step`. With `step = 0.1 mm` ⇒ 5000 e/mm, the MIP ionization density. **Do not retune for "stronger" tracks** — this matches MIP. |
| `step: 0.1*wc.mm` | 0.1 mm | Spacing between successive point depositions along the ray. |
| `time: 0*wc.us` | 0 µs | All depos at t = 0 (fixed-time simulation; see `simparams.jsonnet:fixed:true`). |
| `ray.tail` / `ray.head` | 3-vectors in cm | Endpoint positions; `wc.point(x, y, z, wc.cm)` does the unit conversion. |

The number of depos is `length / step`. A 100 cm track has 10 000 depos.

## Active-volume placement constraint

Depos with x outside the **per-anode `[anode_plane, cathode_plane]`** are dropped before drifting. This is enforced inside the drifter using the `volumes` field of `params.jsonnet:det.volumes` (see [[PDVD Detector Parameters]] and [[pdhd Detector Parameters]] for the per-detector geometry).

The wire-plane x in the wires-JSON is **outside** the active drift volume — the anode-cut plane sits at offset `0.5×apa_g2g` (PDVD) or `0.5×apa_g2g − plane_gap` (HD) toward the cathode from the wire plane. A depo placed exactly at the wire-plane x silently produces no signal.

| Detector | Wire plane → anode-cut offset | Wire plane → response plane offset |
|----------|-------------------------------|------------------------------------|
| VD | 5.715 mm (= 0.5×114.3 mm) | 22.39 cm (= 0.5×85.725 + 181 mm) |
| HD | 5.246 mm (= 0.5×114.43 − 4.76 mm) | 14.29 cm (= 0.5×85.87 + 100 mm) |

**Recommended placement:** at or below the response plane (toward the cathode side). Tracks closer to the wires than the response plane may be "backed up" to the response plane in PDVD (no proper field response); in pdhd they are simply discarded.

## Tick-axis offset (reframer trim)

The per-anode pipeline includes a `Reframer` that trims `tbin = elec.fields.nticks = (response_plane / drift_speed) / tick` ticks off the front (`pgrapher/common/params.jsonnet:194`). Effective tick 0 of the readout output is the moment a depo *at the response plane* reaches the wires.

Drift offset → tick conversion:

| Detector | drift_speed | ticks per cm of additional drift |
|----------|-------------|----------------------------------|
| VD | 1.473 mm/µs, 0.5 µs/tick | ≈ 13.6 ticks/cm |
| HD | 1.565 mm/µs, 0.5 µs/tick | ≈ 12.8 ticks/cm |

Empirically measured (VD anode 0): a 27.6 cm increase in drift offset (response-plane → response-plane + 27.6 cm) shifted the peak from tick 745 to tick 1118 (ratio 27.6 cm / 373 ticks / 0.5 µs/tick = 1.48 mm/µs ≈ `drift_speed`).

## Track geometry semantics

For a track parallel to a wire plane (constant x) with direction `d̂` in y-z:

| Plane | "Hot" channel count | Reason |
|-------|---------------------|--------|
| Plane parallel to track | many: `length / pitch` × |cos(d̂, pitch_dir_plane)| | track crosses many wires |
| Plane perpendicular to track | 1–few | track follows a single wire-direction line |

Concretely, for a track perpendicular to U wires (so `d̂ = pitch_dir_U`):

| Detector | Plane | |cos| | Expected hot channels per cm of track |
|----------|-------|-------|---------------------------------------|
| VD | U | 1.0 | 13.07 (= 10 mm/cm ÷ 7.65 mm pitch) |
| VD | V | 0.5 | 6.54 |
| VD | W | 0.5 | 9.80 (= 10 ÷ 5.10 × 0.5) |
| HD | U | 1.0 | 21.42 (= 10 ÷ 4.669) |
| HD | V | 0.5 | 10.71 |
| HD | W | 0.5 | 10.44 (= 10 ÷ 4.79 × 0.5) |

These ratios are the validation oracle: the number of channels with `peak |ADC| > N×RMS` on each plane should match `track_length × 10 × |cos(d̂, pitch_dir_plane)| / pitch_plane` to within a few channels.

## Charge polarity by plane

For a downward-going (or any) MIP track simulated this way:

- **W (collection)** plane: unipolar positive peak. Width set by the FE shaping time (≈ ±10–20 ticks at 0.5 µs/tick).
- **U, V (induction)** planes: bipolar pulse (positive lobe + negative lobe) of opposite total area, peak magnitude similar to W on each plane after deconvolution.

In peak-aligned mean of raw frames, W shows a ~Gaussian-like peak while U/V show their characteristic bipolar S-shape.

## See also

[[WireCell Wires Schema]], [[PDVD Detector Parameters]], [[pdhd Detector Parameters]], [[ProtoDUNE-VD WireCell Configuration Overview]]

## Sources

- [[source-session-2026-04-29-track-sim-geometry]]
