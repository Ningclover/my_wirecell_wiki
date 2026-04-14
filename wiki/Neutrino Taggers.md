# Neutrino Taggers

Six tagger algorithms run in Stage 6 (`TaggerCheckNeutrino`) after [[Particle Identification]]. Each sets a flag on the event's neutrino vertex.

## NuMu tagger

Identifies νμ CC interactions: one long MIP track + no large shower.

Key criteria:
- At least one segment classified as muon (long, MIP dQ/dx, `!is_shower`)
- Muon segment length > `numu_track_length_min` (configurable, typically 50 cm)
- Total shower energy < `numu_shower_energy_max`
- Vertex inside fiducial volume

Outputs: `kNuMu` flag, muon momentum estimate from range (stopping muon) or MCS (exiting muon).

## NuE tagger

Identifies νe CC interactions: one EM shower pointing back to vertex + no long MIP track.

Key criteria:
- Exactly one primary shower with `is_shower = true`
- Shower start within `nue_shower_gap_max` cm of the neutrino vertex (gap < 3 cm → electron, not photon)
- No segment longer than `nue_track_length_max` classified as muon
- Shower energy > `nue_shower_energy_min`

Outputs: `kNuE` flag, electron energy from shower calorimetry.

## Pi0 tagger

Identifies π⁰ → γγ: two photon showers with consistent invariant mass.

Key criteria:
- Exactly two `PR::Shower` objects with `is_shower = true`
- Both showers have a gap (photon conversion distance > 2 cm) from the vertex
- Opening angle between shower directions consistent with π⁰ mass (~135 MeV)
- Invariant mass window: 80–200 MeV (configurable)

Outputs: `kPi0` flag, reconstructed π⁰ mass.

## Cosmic tagger

Identifies through-going or entering cosmic muons.

Key criteria:
- Track enters or exits the TPC fiducial volume at both ends (through-going)
- Or track starts outside the fiducial volume (entering)
- Or track direction is steeply downward (nearly vertical, consistent with cosmic muon)
- Flash-matching score below threshold (if PMT data available)

Outputs: `kCosmic` flag. Cosmic-tagged clusters are suppressed in neutrino analyses.

## SSM tagger (Stopping Shower Muon)

Identifies muons that stop and produce a Michel electron shower.

Key criteria:
- Primary track classified as muon (MIP dQ/dx)
- Track endpoint has a short shower segment attached (Michel electron, < 50 MeV)
- Michel shower is separated from track endpoint by < `ssm_gap_max` cm
- Michel energy in expected range (0–52.8 MeV for µ+ decay at rest)

Outputs: `kSSM` flag, Michel electron energy.

## SinglePhoton tagger

Identifies NC interactions producing a single photon (radiative Δ decay, π⁰ with one photon lost).

Key criteria:
- Exactly one `PR::Shower` with `is_shower = true`
- Shower has a gap > `singlephoton_gap_min` cm from the neutrino vertex (conversion gap → photon, not electron)
- No muon-classified segment
- Shower energy > `singlephoton_shower_energy_min`

Outputs: `kSinglePhoton` flag. Used to study NC π⁰ background in νe analyses.

## Tagger priority and combination

Taggers are not mutually exclusive by default. An event can carry multiple flags (e.g., `kNuMu + kSSM` for a stopping muon with Michel). Downstream analysis code applies priority ordering (typically: Cosmic > NuMu > NuE > Pi0 > SSM > SinglePhoton) to assign a single interaction hypothesis.

## See also

- [[WireCell Clus Pipeline Overview]]
- [[Track Shower Separation]]
- [[Neutrino Vertex Determination]]
- [[Particle Identification]]
