---
tags: [algorithm, experiment]
sources: 1
updated: 2026-04-14
---

# PDVD Noise Filtering Configuration

Noise filtering pipeline for ProtoDUNE-VD, configured in `nf.jsonnet`. One `OmnibusNoiseFilter` instance per anode.

## Filter assembly

```
OmnibusNoiseFilter
  ├── channel_filters:
  │     └── PDVDOneChannelNoise        (per-channel, all anodes)
  ├── multigroup_chanfilters:
  │     ├── PDVDCoherentNoiseSub        (grouped, all anodes, uses chndb.groups)
  │     └── PDVDShieldCouplingSub       (grouped, TOP ONLY: anode.ident > 3, uses chndb.top_u_groups)
  ├── grouped_filters: []              (empty — grouped handled via multigroup)
  └── channel_status_filters: []       (empty)
```

## Per-channel filter: PDVDOneChannelNoise

Standard per-channel cleanup — RC correction, spectral masking, baseline subtraction. See [[Detector-Specific Signal Processing]] for what this does in C++.

## Grouped filter: PDVDCoherentNoiseSub

Coherent noise subtraction using median across electronics groups. Applied to **all anodes** using `chndb.groups` from [[PDVD chndb-base]].

- `rms_threshold: 0.0` — no RMS gating; all channels contribute to the group median

## Grouped filter: PDVDShieldCouplingSub (top drift only)

Shield coupling subtraction applied only to **top drift anodes** (anode.ident > 3), using `chndb.top_u_groups`. Implements the capacitive coupling subtraction described in [[Detector-Specific Signal Processing]]:

1. Scale down by strip length (from `PDVD_strip_length.json.bz2`)
2. Compute median (excluding signal-protected and outlier bins)
3. Subtract median
4. Scale back up

- `rms_threshold: 0.0`

## Channel mask mapping

```
maskmap: { sticky: "bad", ledge: "bad", noisy: "bad" }
```

All three mask types route to the `"bad"` channel mask, which is saved to `chanmaskmaps` in the NF output frame saver.

## Input/output frame tags

| | Tag |
|--|-----|
| Input traces | `orig{anode.ident}` |
| Output traces | `raw{anode.ident}` |

## Resampler (optional)

When `use_resampler='true'`, a `Resampler` node is inserted **before** the NF for bottom drift anodes (n < 4):
- Resamples from **512 ns** tick (hardware clock) to **500 ns** (WCT standard)
- Method: `time_pad: "linear"`
- Top drift (n ≥ 4) is not resampled

## See also

[[ProtoDUNE-VD WireCell Configuration Overview]], [[Omnibus Noise Filter]], [[PDVD Detector Parameters]], [[OmniChannelNoiseDB]], [[Detector-Specific Signal Processing]]

## Sources

- [[source-pdvd-wct-config]]
