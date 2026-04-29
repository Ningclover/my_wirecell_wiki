---
tags: [source]
type: conversation
date: 2026-04-29
context: Generated MIP-track simulations for ProtoDUNE-VD/HD per-anode per-plane validation; investigated wires-JSON schema, active-volume bounds, jsonnet extVars.
files_touched:
  - dunereco/dunereco/DUNEWireCell/protodunevd/protodunevd-wires-larsoft-v3.json.bz2
  - dunereco/dunereco/DUNEWireCell/pdhd/protodunehd-wires-larsoft-v1.json.bz2
  - toolkit/cfg/pgrapher/experiment/protodunevd/params.jsonnet
  - toolkit/cfg/pgrapher/experiment/protodunevd/simparams.jsonnet
  - toolkit/cfg/pgrapher/experiment/pdhd/params.jsonnet
  - toolkit/cfg/pgrapher/experiment/pdhd/simparams.jsonnet
  - toolkit/cfg/pgrapher/experiment/pdhd/wct-sim-check.jsonnet
  - toolkit/cfg/pgrapher/experiment/pdhd/nf.jsonnet
  - toolkit/cfg/pgrapher/experiment/pdhd/sim.jsonnet
  - toolkit/cfg/pgrapher/common/params.jsonnet
  - toolkit/cfg/pgrapher/common/tools.jsonnet
updated: 2026-04-29
---

# source-session-2026-04-29-track-sim-geometry

Investigation session that generated per-(anode, plane) MIP tracks for ProtoDUNE-VD (8 anodes × U/V/W) and ProtoDUNE-HD (4 anodes × U/V/W), each parallel to a wire plane and perpendicular to one specific plane's wires. Required understanding of: the wires-JSON schema, the per-anode active-drift-volume bounds, jsonnet `extVar` conventions, HD vs VD nf trace-tag conventions, and how the reframer establishes the readout tick origin.

## Confirmed findings

### Wires JSON schema
- The format `Store.{anodes,faces,planes,wires,points}` uses **list positions** (not the local `ident` field) for cross-references — `ident` only encodes the local index within a parent record (face 0/1, plane 0/1/2). Indexing by `ident` collapses 16 VD faces into 2. — see [[WireCell Wires Schema]]
- VD: 8 anodes / 16 faces / 48 planes / 13 840 wires; HD: 4 anodes / 8 faces / 24 planes / 22 208 wires. — see [[WireCell Wires Schema]]
- HD per anode has two faces at slightly different x with **opposite-sign U/V handedness**; the cathode-facing face is `Anode.faces[0]` for anodes 0,2 and `Anode.faces[1]` for anodes 1,3. — see [[WireCell Wires Schema]], [[pdhd Detector Parameters]]
- Per-plane wire directions and pitches: VD U/V at ±60° (pitch 7.65 mm), W vertical (5.10 mm); HD U/V at ±35.7° (pitch 4.669 mm), W vertical (4.79 mm). — see [[WireCell Wires Schema]]

### Active drift volume
- For each anode, depos with x outside `[anode_plane, cathode_plane]` are dropped before drifting. — see [[PDVD Detector Parameters]], [[pdhd Detector Parameters]]
- The wire-plane x sits **outside** the active volume (offset by `0.5 × apa_g2g` toward the cathode for VD, `0.5 × apa_g2g − plane_gap` for HD). Placing a depo at the wire-plane x silently produces no signal. — see [[WireCell Sim Track Conventions]]
- The response plane is at `0.5 × apa_w2w + response_plane` from the wire plane (22.39 cm for VD, 14.29 cm for HD). Tracks should sit at or beyond it on the cathode side for clean field response. — see [[WireCell Sim Track Conventions]]

### Tick axis and reframer
- The per-anode `Reframer` trims `tbin = elec.fields.nticks = (response_plane / drift_speed) / tick` ticks off the front. Effective tick 0 corresponds to the moment a depo at the response plane reaches the wires. — see [[WireCell Sim Track Conventions]]
- Empirical drift-offset → tick conversion verified at 1.48 mm/µs (≈ `drift_speed`) for VD. — see [[WireCell Sim Track Conventions]]

### Sim track config conventions
- `sim.tracks(tracklist, step=0.1*wc.mm)` with `charge: -500` is the MIP convention (5000 e/mm). Do not retune for stronger tracks. — see [[WireCell Sim Track Conventions]]
- Track ray endpoints use `wc.point(x, y, z, wc.cm)`; values are in cm. — see [[WireCell Sim Track Conventions]]
- For a track perpendicular to plane P's wires, the expected per-plane "hot" channel count is `length × 10 mm/cm × |cos(track, pitch_dir_plane)| / pitch_plane`. Validated against simulation output for VD U-perpendicular and HD APA1 U/V/W tracks. — see [[WireCell Sim Track Conventions]]

### pdhd jsonnet specifics
- `pgrapher/experiment/pdhd/params.jsonnet:114` reads `gain` via `std.parseJson(std.extVar("elecGain"))`. Must be passed as a **string** (`-V elecGain=14` at wire-cell, `--ext-str elecGain=14` at jsonnet). `--ext-code` fails with `Unexpected type number, expected string`. — see [[pdhd Detector Parameters]]
- Default `elecGain=14` mV/fC; alternative `elecGain=7.8` for low-gain runs; `params.jsonnet:169–170` selects noise-spectrum file by gain (`>8` → `14mVfC`, else `7d8mVfC`). — see [[pdhd Detector Parameters]]
- `pdhd/nf.jsonnet` uses `intraces: ''` (wildcard) so no frame `Retagger` is needed between sim and NF, unlike PDVD where `nf.jsonnet` hardcodes `intraces: 'orig'`. — see [[pdhd Detector Parameters]]
- HD nf output trace tag is `'raw%d' % n`; sim digitizer output trace tag is `'orig'` with per-anode frame tag `'orig%d' % n`. — see [[pdhd Detector Parameters]]

## Context

The user wanted per-(anode, plane) sim tracks to validate field-response and noise-filtering for ProtoDUNE-VD and ProtoDUNE-HD. The first attempt produced empty plots because the tracks were placed at the wire-plane x (which is outside the active drift volume); fixing this required reading the per-detector `params.jsonnet` to derive the anode/cathode/response-plane geometry. While porting an existing VD `wct-sim-check-track.jsonnet` to HD, the user encountered the `elecGain` extVar requirement and learned the `--ext-str` vs `--ext-code` distinction. Validation that the sim was geometrically correct came from comparing per-plane "hot channel" counts to `length × |cos| / pitch`, which agreed within a few channels for VD anode 0 and HD APA 1.
