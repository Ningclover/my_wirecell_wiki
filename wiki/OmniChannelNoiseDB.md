---
tags: [component]
sources: 1
updated: 2026-04-14
---

# OmniChannelNoiseDB

JSON-configurable per-channel noise database. Provides [[Omnibus Noise Filter]] with per-channel noise parameters and frequency-domain filter spectra.

**File:** `src/OmniChannelNoiseDB.cxx` (702 lines)  
**Header:** `inc/WireCellSigProc/OmniChannelNoiseDB.h`

## What it stores per channel

- Scalar parameters: baseline, gain correction, response offset, RMS cuts
- Frequency-domain filter spectra: RCRC response, electronics reconfig, frequency masks, response

## Filter caching

- RCRC filters cached by rounded RC constant
- Reconfig filters cached by packed int key (gain + shaping + from-gain + from-shaping into 8-bit fields)
- Response filters cached by wire plane ID

## Known issues / bugs

- **BUG-NF-4** (NOT FIXED): Reconfig cache key overflow — each packed field is 8 bits (0-255). Values > 25.5 in natural units overflow into adjacent fields, causing silent cache key collisions and wrong filters returned.
- **BUG-NF-5** (NOT FIXED): Four functions return reference to a static empty `filter_t dummy` when filter pointer is null. Callers expecting size `m_nsamples` get empty vector → out-of-bounds access.
- **BUG-NF-11** (NOT FIXED): Static `default_filter` initialized once on first call. If `m_nsamples` changes via reconfiguration, static filter retains old size.

## SimpleChannelNoiseDB

Simpler programmatic alternative, registered as `testChannelNoiseDB`. Parameters set via C++ setter methods rather than JSON. Uses lazy index allocation — first-seen channel gets next available index.

- **BUG-NF-8** (NOT FIXED): `chind()` is `const` but mutates `m_ch2ind` via `mutable`. Unknown channel queries permanently pollute the index map.

## See also

[[Omnibus Noise Filter]], [[WireCellSigProc Pipeline Overview]]

## Sources

- [[source-sigproc-examination]]
