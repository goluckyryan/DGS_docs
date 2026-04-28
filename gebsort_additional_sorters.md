# GEBSort — Additional and GRETINA-Specific Sorters

Stability: C2 - Active / semi-stable

_Split from `gebsort.md` 2026-04-26. Source: `DGS_tools_pack/gebsort/`._

This file covers non-core GEBSort sort modules: GRETINA-specific sorters and additional detector sorters
for TAC-II, DuoGe, X-Array, DFMA, S800, and other ancillary detectors.
For the core GEBSort framework, calibration workflow, GEBMerge, bin_dgs, and jta, see `gebsort.md`.

## Table of Contents

- [GRETINA-Specific Sort Functions](#gretina-specific-sort-functions)
- [Additional Detector Sorters (bin_*.c)](#additional-detector-sorters-binc)
- [DGS-Specific Sorter Variants](#dgs-specific-sorter-variants)
- [GEBSort Calibration Utilities](#gebsort-calibration-utilities)
- [GRETINA Mode-1 and Mode-2 Sort Functions](#gretina-mode-1-and-mode-2-sort-functions)
- [GEBSort Math & Utility Helpers](#gebsort-math--utility-helpers)
- [Cross-References](#cross-references)

---

## GRETINA-Specific Sort Functions

_Source: `gebsort/bin_angcor_GT.c`, `bin_DCO_GT.c`, `bin_g4sim.c`, `bin_gtcal.c`, `bin_ft.c`, `bin_final.c`. Code-read 2026-04-26._

These sorters are GRETINA-oriented extensions. They require `GTMerge.h` and process tracked gamma-ray data (`GEB_TYPE_TRACK`, `TRACKED_GAMMA_HIT`). They are not active in the standard DGS build but coexist in the same gebsort tree.

### `bin_angcor_GT.c` — GRETINA Angular Correlation Sorter ✅ verified 2026-04-26 — `bin_angcor_GT.c:L155-387`

Angular correlation sorter for **gamma-tracking data** (GRETINA `GEB_TYPE_TRACK`). Requires gamma-tracking preprocessing (comment: "always run through bin_mode1 before using this function so that Doppler corrections can be handled properly").

**Spectra created:**
- `angcor_cube` — 3D ROOT `TH3F`: E1 × E2 × opening_angle; dimensions `GGMAX × GGMAX × 36` bins (0–180°). Event-mixed background is stored in `angcor_cube_oo`.
- `angcor_cube_oo` — same shape as `angcor_cube`; filled against a rolling queue of the last 25 events (LOQ=25) for combinatorial background.
- `linpol_polused` — 1D histogram of the angle used (181 bins, 0–180°); diagnostic.

**Warning:** ROOT has a 1,073,741,822-byte object limit; at large `GGMAX` the cube may exceed this and cause an immediate exit.

**Angle selection (`Pars.angcor_useplaneang`):**
- `false` (default): angle between the two gamma-ray unit vectors (suitable for source data).
- `true`: angle between the planes defined by each gamma ray and the beam axis (z-axis) — used for DCO in in-beam data.

For each event: extracts all tracked gamma rays from `TRACKED_GAMMA_HIT`, requires exactly one `GEB_TYPE_TRACK` header per event (exits if more or fewer), converts positions to unit vectors, fills all pairs `(j,k)` and `(k,j)` into `angcor_cube`, then updates `angcor_cube_oo` against the queue from the previous 25 events.

### `bin_DCO_GT.c` — GRETINA DCO Ratio Sorter ✅ verified 2026-04-26 — `bin_DCO_GT.c:L1-100`

Direction Correlation from Oriented nuclei (DCO) sorter for GRETINA tracked gamma data. Produces 3D matrices for DCO analysis:

**Spectra created:**
- `DCOpp_sig` / `DCOpp_res` — pair-pair signal/residual 3D matrices
- `DCOpb_sig` / `DCOpb_res` — pair-beam signal/residual 3D matrices
- `DCObb_sig` / `DCObb_res` — beam-beam signal/residual 3D matrices
- `DCOpp_sig_relang` / `DCOpp_res_relang` — 1D relative-angle spectra
- `DCO_gate` / `DCO_all` — gated and ungated 1D diagnostic spectra

Also uses a LOQ=25 event queue for combinatorial background (same pattern as `bin_angcor_GT`). Maintains a lookup table `lkup[4096]` and `leok[5][6]` error counters.

### `bin_gtcal.c` — GRETINA Calibration Sorter ✅ verified 2026-04-26 — `bin_gtcal.c:L1-117`

Special calibration sorter for GRETINA, maintained by Shaofei Zhu. Operates on `CRYS_INTPTS` (crystal interaction points) from `GEB_TYPE_DECOMP`. The current code is largely a stub — `sup_gtcal()` defines no active histograms (commented out), and `bin_gtcal()` iterates over `GEB_TYPE_DECOMP` payloads but does nothing with them yet (implementation placeholder). Not used in production DGS analysis.

### `bin_g4sim.c` — GEANT4 Simulation Event Sorter ✅ verified 2026-04-26 — `bin_g4sim.c:L1-144`

Sorter for GEANT4-simulated events (`GEB_TYPE_G4SIM`). Decodes the `G4SIM_EGS` struct:

| Field | Description |
|---|---|
| `type` | Simulation type code |
| `num` | Number of simulated gamma rays (max `MAX_SIM_GAMMAS`=10) |
| `full` | Full deposited energy |
| `g4Sim_emittedGamma[i].{e,x,y,z,phi,theta,beta}` | Per-gamma energy, position, angle, velocity |

Currently only logs events when `CurEvNo <= NumToPrint`; no spectra defined (`sup_g4sim()` is empty). Used for debugging simulation input into GEBSort. Commented-out code shows planned histogram fill that was never activated.

### `bin_ft.c` — Fast-Timing / Flash-ADC Sort Function ✅ verified 2026-04-26 — `bin_ft.c:L1-260`

Sorter for `GEB_TYPE_FT` (Flash-Timing / FADC) events. Defines the `FTEVENT` struct (32 `uint16` fields + 256-sample trace):

| Field | Description |
|---|---|
| `Fixed1/2` | Fixed header words |
| `USER_PACKET_DATA` | User-defined ID |
| `GeoAddr_Packet_Length` | VME geo address + packet length |
| `TrigTimeLO/LMI/HI` | Trigger timestamp (3× 16-bit) |
| `Types` | Discriminator/type bits |
| `CFD_raw_0..3` | Raw CFD samples |
| `BLavg_low/high` | Baseline average |
| `Esum_lead/trail_low/high` | Leading/trailing energy sums |
| `Energy` | Computed energy |
| `PSA_max/base/sum0/sum1` | Pulse shape analysis values |
| `EXT_TS_0..2` | External timestamp |
| `ModuleTupe` | Module type (sic — typo in source) |
| `Channel_ID` | Channel identifier |
| `trace[256]` | 256-sample ADC waveform |

Creates two placeholder 1D ROOT spectra (`tf_sp1`, `tf_sp2`, 2048 bins each). The `bin_ft()` function locates `GEB_TYPE_FT` events but the histogram fill is commented out/TBD — this is a skeleton sorter. Used for early flash-ADC detector integration work.

### `bin_final.c` — DGS+DFMA Combined Final Sorter ✅ verified 2026-04-26 — `bin_final.c:L1-100`

A combined final-stage sorter that works on **pre-processed DGS + DFMA** data. Unlike `bin_dgs.c` which handles raw DGS events, `bin_final.c` consumes `DGSEvent[]` and `DFMAEvent[]` arrays already populated by earlier sort stages, plus tracked gamma rays from `nTrackedGammas`.

**Spectra:**
- `firstType` — 2D: first GEB type in event × multiplicity (diagnostic)
- `dgsdfma_hit` — 2D: DGS hit vs DFMA hit coincidence map
- `SMAP_egated1/2` — 2D: spatial maps, energy-gated (pair 1 and pair 2)
- `sp_dt_fmagt` — 1D: timing difference DGS vs DFMA (HUNTFMAGTCOIN path)
- `sp_gam` — 1D: gamma energy spectrum
- `dtXspicketID` — 2D: timing × picket ID (fence)

Has a compile-time flag `HUNTFMAGTCOIN` (hardcoded 1) for DFMA-GRETINA coincidence hunting.

---

## Additional Detector Sorters (bin_*.c)

_Source: `gebsort/bin_tac2.c`, `bin_dub.c`, `bin_XA.c`, `bin_dfma.c`, `bin_angdis.c`, `bin_ndc.c`, `bin_mux.c`, `bin_s800.c`, `bin_mode3.c`, `bin_linpol.c`. Code-read 2026-04-26._

All follow the same GEBSort `sup_` / `bin_` / `exit_` entry-point convention. Enabled via uncommented lines in `GEBSort.chat`.

### `bin_tac2.c` — TAC-II Timing Sorter (534 L)

Decodes TAC-II trigger packets (GEB type `GEB_TYPE_DGSTRIG`) within coincidence events. Purpose: compute the sub-4-ns timing between the DGS trigger timestamp and the TAC-II TDC timestamp.

**Key logic:**
- Loops over coincidence hits; selects `GEB_TYPE_DGSTRIG` events
- Unpacks 10-word TAC-II packet: extracts `TRIG_TS` (48-bit trigger timestamp ×10 ns), coarse TDC counter, 4 vernier words (A/B/C/D), valid flags, vernier pattern
- **Vernier resolution:** 50 ps/LSB (`nspervernier = 0.050`) ✅ verified 2026-04-26 — `bin_tac2.c:L116,L430`
- **4-ns phased counters:** `TDC_TS4X = MOD_TDC_COARSE_TS + 4 × Cnt4ns_X` (4 ns per count; MOD_TDC_COARSE_TS = base floored to 2^18 boundary) ✅ verified 2026-04-26 — `bin_tac2.c:L281,L285,L289,L293`
- **Net TDC timestamp per channel:** `TDC_NET_TS_A = TDC_TS4A - TDC_VERNIER_A_ns`; B/C/D get +1/+2/+3 ns phase offset added (`TDC_NET_TS_B = TDC_TS4B - TDC_VERNIER_B_ns + 1`, etc.) ✅ verified 2026-04-26 — `bin_tac2.c:L461-464`
- **Output:** global `dgs_tac2` (best-channel net timestamp), `dgs_tac2_valid` flag
- **Histogram:** `tac2dev` — 1D: `dgs_tac2 - TRIG_TS` deviation (4096 bins, ±2048 ticks)
- Tracks `TDC_TS4A_last` etc. as static state for consecutive-event timing analysis
- **Rollover handling:** if `TDC_PERIOD < TRIG_PERIOD` (i.e., TDC coarse counter advanced less than expected), adds `655360` (= 65536 × 10) to `TDC_COARSE_TS`; `2^18` = 262144 is used separately as the **MOD divisor** for `MOD_TDC_COARSE_TS` = `TDC_COARSE_TS` floored to nearest 262144 boundary. ✅ verified 2026-04-26 — `bin_tac2.c:L217-223` (rollover add 655360); `L244-245` (MOD: `d1 = TDC_COARSE_TS/262144; AH = TDC_COARSE_TS - (long long)d1*262144`). **Correction:** earlier KB stated "adds 2^18" — incorrect; 2^18=262144 is the mod period, not the rollover increment.

**External variables set:** `dgs_tac2`, `dgs_tac2_valid`, `dgs_trig_ts` (all used by `bin_dgs.c` for DGS-TAC2 timing correlation).

See also: `knowledgeBase/tac2.md` — TAC-II TDC hardware and packet format.

---

### `bin_dub.c` — DuoGe (DUO) Sorter (264 L)

Handles **DuoGe** (DUO) — a two-detector array experiment. Uses the same `DUBEvent[]` array as the DUO system.

**Key structures / constants:**
- `NDUB = 97` — max DUO detectors (96 + 1) ✅ verified 2026-04-26 — `bin_dub.c:L36` (`#define NDUB 96+1`)
- `TAPE_MOVED=10`, `EBIS_CLOCK=11`, `BETA_FIRED=12` — channel type codes
- `ALL2DS = 0` — compile-time flag (disabled) ✅ verified 2026-04-26 — `bin_dub.c:L30`
- Per-detector 2D PID spectra: `dubpid[NDUB]` — 2D ROOT histograms (PID = particle identification)
- `dubhit` — 1D hit counter
- **Maps:** `DUBtlkup[]` (type lookup) and `DUBtid[]` (detector ID); loaded from `dgs_map.dat` format (same as DGS)
- Includes `gsang.h` for Gammasphere angular mapping

**Physics:** DUO used at ATLAS for two-detector coincidence experiments (similar to DGS but with DUO detector geometry).

---

### `bin_XA.c` — X-Array Sorter (971 L)

Handles the **X-Array** — a clover detector array used for beta/gamma coincidence experiments (tape transport, EBIS, beta detector).

**Key constants:**
- `NXA = 40` — max X-Array clover detectors ✅ verified 2026-04-26 — `bin_XA.c:L26`
- `TAPE_MOVED=10`, `EBIS_CLOCK=11`, `BETA_FIRED=12` — special channel codes
- `PZ_SPECTRA = 1` — enable pole-zero diagnostic spectra
- `NAVETRACE = 1000` — trace averaging buffer depth
- Timing gates: `gb_dt_lim = -10/+40` (Ge-Beta), `gg_lim1 = -20/+20` (clover-clover), `tg_lim1/2 = 620/1220` (tape gate) ✅ verified 2026-04-26 — `bin_XA.c:L52-54`

**Key histograms:**
- `hBetaCounter` — beta detector event count
- `hEhiBeta`, `hEhiClo` — energy vs beta/clover coincidence 2D
- `hGeBeta_DT`, `hGeTape_DT`, `hGeGe_DT` — Ge-Beta, Ge-Tape, Ge-Ge timing differences (2D)
- `hClovID`, `hClovBetaID` — clover ID vs energy (2D)
- `hClovClov_DT`, `hClovClov` — clover-clover timing / coincidence (2D)
- `xa_hEhiRaw`, `xa_EhiCln`, `xa_EhiDrty` — raw / clean / dirty event energy 2D per detector
- `xa_PZraw` — PZ diagnostic: energy vs pre-rise baseline (2D)
- Per-detector trace baseline diagnostics: `xa_base1_diff`, `xa_base2_diff`

**External arrays:** `XAEvent[MAXCOINEV]`, `XAng` (coincidence multiplicity). Maps: `XAtlkup[]`, `XAtid[]`.

---

### `bin_dfma.c` — DFMA Detector Sorter (4,897 L)

The **Digital Focal-plane Multi-channel Analyzer** sorter — used for AGFA (Argonne Gas-Filled Analyzer) recoil-implant-decay correlation experiments. This is the most complex single sorter in GEBSort.

**Physics purpose:** correlates heavy-ion recoils (identified by energy in a DSSD) with subsequent alpha/beta decay events in the same pixel, plus prompt gamma rays from DGS. Supports implant-decay correlation with up to 6 decays per chain.

**Key compile-time flags:**
- `WITHDGS = 1` — include DGS gamma events in the sort
- `EnergyFromTrace = 0` — use SZ energy, not raw trace integral
- `SmoothOrNot = 1` — apply trace smoothing
- `FRTIDLOW/HIGH = 1/160` — DSSD front strip type ID range (160 strips)
- `BATIDHIGH = 320`, `BATIDLOW = 161` — DSSD back strip range (160 strips)

**Key data structures:**
- `strip_type` — per-strip calibration: `phystrip`, `thr`, `off`, `gain`, `baseline`; arrays `map_fr[321]` (front) + `map_ba[321]` (back) + `map_box[321]` (Si box)
- `recoil_type` — recoil event: timestamp, energy, pileup, left/right/x focal plane, Ge hits (up to 15), DSSD front/back 256-sample traces
- `decay_type` — decay event: timestamp, energy (fr+ba), pileup flags, time since implant, DSSD traces, associated Ge hits, Si hits, timing fields
- `chain_type` — full implant-decay chain: front/back strip, one `recoil_type`, up to 6 `decay_type` entries, `corr_type` flag
- `pixel_type` — per-pixel state: `status` + `chain_type`
- `focal_plane` — focal plane detector: left/right/icde/ppacde/ppacde2 energies + timestamps
- `pixel_type dssd_corr[161][161]` — 2D pixel correlation array (front × back strip)

**DFMAEvDecompose_v3()** (1,323–1,609 L): decodes raw DFMA event payload
- Extracts `chan_id` (bits 3:0), `board_id` (bits 15:4), `id = board_id×10 + chan_id`
- Looks up `tpe` (type: DSSD/FP) and `tid` from `tlkup[]`/`tid[]` tables
- Extracts `wheel` (word 6 bits 31:16), `LEDts` (48-bit from words 1–2), `header_type` (word 2 bits 19:16)
- Unpacks 256-sample traces from words 13+ (two 16-bit samples per 32-bit word)
- Computes baseline + energy values from trace pre-rise and post-rise windows

**bin_dfma()** main sort function (1,611–4,849 L):
- Loops over coincidence hits, calls `DFMAEvDecompose_v3()` for each GEB type 16 (DFMA) event
- Classifies hits as `DSSD` or `FP` (focal plane)
- **Pixel-by-pixel correlation:** identifies recoil (implant) vs decay events by per-pixel state machine in `dssd_corr[fr][ba]`
- **Output: ROOT TTree** (`TREE_FILES/new_presort.<runname>`) with branches: `s_fr`, `s_ba`, `recoil` (full recoil struct), `ndec`, `dec1`–`dec6` (decay structs)
- Chains are filled into the tree in `exit_dfma()` when the run ends

**Key histograms (selected):**
- `h2_dssd_fr_emax`, `h2_dssd_ba_emax` — DSSD front/back max energy per strip (2D)
- `h2_corr_gammas` — DGS gamma energy vs DSSD energy (2D, 2000×2000)
- `h2_dTgdssd` — DGS-DSSD timing difference (2D, ±2000 ticks)
- `h2_clr`, `h2_clrg` — MCP left/right raw and gamma-gated (2D)
- `h1_decay_rate`, `h1_recoil_rate` — 1D rate monitors
- `h2_dssd_fr_p2`, `h2_dssd_ba_p2` — DSSD strip pileup (2D)

**External dependencies:** `DFMAEvDecompose_v3` (declared extern), `DFMAEvent[MAXCOINEV]` (extern), `DGSEvent[]` (extern), `functions_dfma.h`, `get_dead_layer_corrections2.cpp`.

---

### `bin_angdis.c` — Angular Distribution Sorter (311 L)

Computes angular distributions from GRETINA tracked gamma data. Uses `TRACK_STRUCT` to hold tracking output.

**Key constants/gates:**
- `PPLO=1`, `PPHI=2`, `BBLO=3`, `BBHI=4` — peak/background gate indices into `gate_spe[]`
- Uses external `gate_spe[NGATE_SPE]` — lookup array of gate values

**Key histograms:**
- `mpolang`, `mpolang1`–`mpolang3` — 2D: angular distribution multipole × angle
- `angcenter1` — 1D: centroid angle

Primarily used with GRETINA tracking output (not DGS-native).

---

### `bin_ndc.c` — NDC Stub Sorter (137 L)

**Near-empty stub** for an NDC (neutron detector ?) sorter. `sup_ndc()` and `bin_ndc()` extract crystal/module IDs but have no histogram fills or actual analysis logic; `exit_bin_ndc()` (note: function is named `exit_bin_ndc`, not `exit_ndc`) is empty. ✅ verified 2026-04-26 — `bin_ndc.c:L43-137` Only external reference: `ehi[MAXDETPOS+1]` (from `GEBSort`). This is a template for future work — not an active sorter.

---

### `bin_mux.c` — Segment Multiplexer Sorter (200 L)

Handles multiplexed detector segment data (GRETINA or similar segmented germanium detector arrays).

**Key structures:**
- `PAYLOAD` — raw byte buffer (`char p[MAXDATASIZE]`)
- `TRACK_STRUCT` — tracking payload container

**Key histograms:**
- `segseg[130]` — 2D per-segment spectra
- `smoke1[130]`, `smoke2[130]` — 2D diagnostic spectra
- `NBINS = 180` ✅ verified 2026-04-26 — `bin_mux.c:L42`

Uses external `exchange` struct from `GEBSort`. Primarily a GRETINA analysis helper for segment-segment coincidence mapping.

---

### `bin_s800.c` — S800 Spectrograph Sorter (609 L) ✅ verified 2026-04-26 — `bin_s800.c:L44-113,L166-177`

Handles data from the **S800 magnetic spectrograph** (MSU/NSCL) for particle identification in Gammasphere+S800 experiments.

**Key histogram:**
- `S800_PID` — 2D PID (particle identification) matrix (TH2F, axes: tof vs de) ✅ verified 2026-04-26 — `bin_s800.c:L51,L171-173`
- `PIDwin` — `TCutG` gate for S800 PID selection ✅ verified 2026-04-26 — `bin_s800.c:L52`
- `s800_stat[30]` — counters per S800 channel ✅ verified 2026-04-26 — `bin_s800.c:L54`

**Key function:** `rd2dwin(winname)` (L59–70) — reads a 2D graphical cut (`TCutG`) from the ROOT file by name (`f->Get(winname)`). Used to load the PID gate at startup. ✅ verified 2026-04-26 — `bin_s800.c:L58-68`

Uses external `exchange` struct (`extern EXCHANGE exchange` at L45). The `bin_s800()` function loops over coincidence events, extracts S800-type hits, applies PID gate.

---

### `bin_linpol.c` — Linear Polarization Sorter (673 L) ✅ verified 2026-04-26 — `bin_linpol.c:L33-111,L139-241`

Computes linear polarization of gamma rays using the azimuthal angle of the Compton-scattering plane normal relative to the reaction plane. Uses GRETINA tracked gamma data (`GEB_TYPE_TRACK`).

**Key constants/settings:**
- `NBINS = 180` — histogram half-range (±180° azimuth)
- `LOQ = 1000` — event mixing queue depth for reference spectrum

**Histograms:**
- `lp_azin` — 2D: gamma energy (`Pars.GGMAX` bins) × azimuth angle of Compton-scatter plane normal (−180° to +180°, 361 bins); Y-axis: "linear polarization: azimuth angle of normal"
- `lp_azin_ref` — same shape; filled against the LOQ=1000 rolling event queue for mixed-event reference

**Algorithm:** For each event, identifies the polarization gamma ray (emitted near beam axis), locates a scattered companion gamma ray, computes the normal vector to the Compton scattering plane in the reaction frame, projects onto the azimuthal angle, and fills `lp_azin`. The reference `lp_azin_ref` is filled against mixed events from the LOQ. The asymmetry is extracted offline from `lp_azin` vs `lp_azin_ref`. Used with GRETINA or segmented clover detectors.

---

### `bin_mode3.c` — GRETINA Mode3 Sorter (564 L) ✅ verified 2026-04-26 — `bin_mode3.c:L20-25,L149-560`

Decodes **GRETINA Mode-3** raw crystal data. Mode 3 is the raw per-crystal digitizer output (distinct from tracked/decomposed output). Processes `GEB_TYPE_RAW` and `GEB_TYPE_GT_MOD29` event types using the `GTmode3.h` format.

**Key constants (from `GTmode3.h`):**
- `NCRYSTALS = 121` — max crystal IDs (0–120)
- `NSEG = 31×4×40` — total segment count
- `MAXLENINTS = 519` — max Mode-3 payload length (32-bit words)
- `EOE = 0xaaaaaaaa` — end-of-event marker

**Histograms:**
- `mode3_hitpat` — 1D: crystal ID hit pattern (121 bins)
- `mode3_hitpat_chan` — 1D: channel ID distribution (45 bins, 0–44)
- `eCC_a/b/c/d` — 2D crystal-crystal energy coincidence matrices (4 variants)
- `eSeg` — 2D: segment energy spectrum
- `spb88e` — 2D: diagnostic energy spectrum

Used for experiments requiring raw GRETINA crystal data (mode3 format — not tracked or decomposed). Only processes GEB types `GEB_TYPE_RAW` and `GEB_TYPE_GT_MOD29` — no DGS/Gammasphere data paths in this sorter. ✅ verified 2026-04-27 — `bin_mode3.c:L199,L367,L466,L520` (only GEB_TYPE_RAW and GEB_TYPE_GT_MOD29 handled; no DGS type checks). The function name is `exit_bin_mode3` (note: named with `bin_` prefix, unlike other exit functions — e.g. `exit_dgs`, `exit_tac2`, `exit_angcor_DGS`). ✅ verified 2026-04-27 — grep of all `gebsort/*.c` exit functions confirms all others use `exit_<shortname>()` pattern; only `exit_bin_mode1/2/3` and `exit_bin_ndc` use the `bin_` prefix. ⚠️ Note: earlier KB description ("decomposition/tracking output") was incorrect — corrected 2026-04-26.

---

---

## DGS-Specific Sorter Variants

_Source: `gebsort/bin_dgs_AUX.c` (1,681 L), `bin_dgs_GE.c` (1,681 L), `bin_angcor_DGS.c` (489 L). Code-read 2026-04-26._

### `bin_dgs_AUX.c` — DGS Auxiliary Detector Sorter (1,681 L) ✅ verified 2026-04-26 — `wc -l bin_dgs_AUX.c`

A variant of `bin_dgs.c` tailored for auxiliary detector data processed alongside Gammasphere. Same line count as `bin_dgs.c` (1,681 L), suggesting it is a branch/copy with auxiliary-detector specific modifications. Follows the same `sup_` / `bin_` / `exit_` convention.

Handles auxiliary detector events within the DGS coincidence window: supplemental Si detectors, EBIS trigger signals, tape-move signals, and other ancillary channels that accompany but are not part of the main Gammasphere HPGe detector array. The structure mirrors `bin_dgs.c` but routes auxiliary GEB type IDs and applies different calibration tables.

**External dependencies:** same as `bin_dgs.c` — `DGSEvent[]`, `Pars`, `exchange`; requires `gsang.h` for angular lookup.

---

### `bin_dgs_GE.c` — DGS Germanium-Only Sorter (1,681 L) ✅ verified 2026-04-26 — `wc -l bin_dgs_GE.c`

A second variant of `bin_dgs.c` (same line count: 1,681 L) focused exclusively on HPGe germanium detector events, likely with modifications to exclude BGO/auxiliary channels or to apply a stricter Ge-only coincidence filter. Same external interface as `bin_dgs.c`.

This variant may be used for runs where BGO Compton suppression is not available or not desired, or to produce a clean Ge-only reference spectrum alongside the full `bin_dgs` sort.

---

### `bin_angcor_DGS.c` — DGS Angular Correlation Sorter (489 L)

Angular correlation sorter for **native DGS Gammasphere data** (as opposed to `bin_angcor_GT.c` which works on GRETINA tracked data). Works on `DGSEvent[]` arrays already populated by `bin_dgs`.

**Requires:** `Pars.do_bin_angcor_DGS == 1` — exits immediately with error if `bin_dgs` is not also active (needs energy-calibrated and Doppler-corrected `DGSEvent[]` data).

**Setup (`sup_angcor_DGS`):**
- Reads polar angle `angtheta[i-1]` and azimuthal angle `angphi[i-1]` from `gsang.h` for all 110 Gammasphere detector positions (1-indexed: detectors 1–110)
- Converts each detector's angles to a Cartesian unit vector `(xx[i], yy[i], zz[i])`
- Fills `SMAP_DGS` (schematic detector map: 2D, 361×181 bins covering azimuth 0–360° × polar 0–180°) using `sin(theta)`-weighted azimuth

**Spectra created:**
- `Gsangdiff` — 1D: GS detector opening angle distribution (180 bins, 0–180°)
- `Angcor_cube` — 3D TH3F: E1 × E2 × opening_angle (same structure as `bin_angcor_GT`); filled for all Ge-Ge hit pairs
- `Angcor_cube_oo` — 3D TH3F: same shape; filled against rolling LOQ=15 event queue for mixed background
- `SMAP_DGS` — 2D: schematic detector map (azimuth × polar)

**Angular computation:** dot product of unit vectors `xx[k]·xx[l] + yy[k]·yy[l] + zz[k]·zz[l]`, converted to degrees via `acos`. Fills both `(k,l)` and `(l,k)` into `Angcor_cube` (symmetric fill).

**Hit pattern diagnostic (`exit_angcor_DGS`):**
- `angdis_hitp[i]` counts how many times detector `i` appears in a coincidence; printed as percentage of mean detector occupancy
- Detectors with `< 85%` or `> 115%` of mean occupancy are flagged as outliers
- Used to identify dead or hot detectors by their angular correlation contribution

---

## GEBSort Calibration Utilities

_Source: `gebsort/dgs_pz.c` (178 L), `dgs_ecal.c` (277 L), `dgs_ecal2.c` (204 L). Code-read 2026-04-26._

These standalone programs are used in the DGS calibration workflow to generate `.cal` files from SPE spectra produced by GEBSort sorts.

---

### `dgs_pz.c` — Pole-Zero Calibration Fitter (178 L)

Automatically fits pole-zero (PZ) values from `pznnn.spe` spectra (nnn = 001–110, one per Gammasphere detector).

```bash
dgs_pz  M  K  cal_file_name  factor
```

**Arguments:**
- `M`, `K` — pre-rise and post-rise sample counts used to extract the PZ spectrum
- `cal_file_name` — output calibration file (one line per detector: `det_no  pz_value`)
- `factor` — multiplicative fudge factor applied to all PZ values before writing

**Algorithm:**
1. Reads `pznnn.spe` for each detector (4096-channel, smoothed with 3-point average: `0.5×sp[j] + 0.25×sp[j-1] + 0.25×sp[j+1]`)
2. Searches for the PZ peak in the range channel 850–950; requires `sum > 1000` counts (else writes dummy `1.0`)
3. Computes centroid `mPZ` (weighted mean within ±WIDTH=50 channels of peak): `mPZ ×= (2.0 / 2000)` to normalize to [0, 1]
4. Computes exponential decay constant: `lambda = -ln(mPZ) / M`; prints `1/lambda` (decay time in 10 ns units)
5. Computes PZ for SZ0 algorithm: `pz = mPZ^{(M+K)/M} × factor`; clips at 1.0
6. Writes `det_no   pz_value` pairs to `cal_file_name`
7. Also writes `d_pz.cmd` (GEBSort display script to visualize each `pznnn.spe`)

**Output:** mean PZ value across all detectors + mean decay constant.

**PZ spectra source:** extracted from a GEBSort run using a `.x get_pz.cc` ROOT script (not in this repo).

---

### `dgs_ecal.c` — Energy Calibration Fitter v1 (277 L)

Automatically fits linear energy calibration (offset + gain) from `ehinnn.spe` spectra (one per detector, 16384 channels).

```bash
dgs_ecal  cal_file_name  source  lowch  desgain
```

**Arguments:**
- `cal_file_name` — output calibration file (`det_no  offset  gain` per line)
- `source` — calibration source: `207Bi`, `88Y`, or `60Co`
- `lowch` — lower channel limit for peak search (excludes low-energy noise)
- `desgain` — desired gain factor (all offset/gain values are divided by `desgain` before writing)

**Source peak energies (hardcoded):** ✅ verified 2026-04-26 — `dgs_ecal.c:L19-24`

| Source | Low peak (keV) | High peak (keV) |
|---|---|---|
| 207Bi | 569.702 | 1063.662 |
| 88Y | 898.0450 | 1836.0630 |
| 60Co | 1173.228 | 1332.490 |

**Algorithm:**
1. For each detector 1–110, reads `ehinnn.spe`; skips if total counts `< 1000` (writes dummy `0.0 1.0`)
2. Finds global max channel above `lowch`; sets threshold at `max/4`
3. Finds approximate hi peak: scans downward from the top until above threshold
4. Finds exact hi peak centroid: weighted mean within ±`w1=5` channels of max
5. Finds approximate lo peak: continues scanning downward from hi-50
6. Finds exact lo peak centroid: same method
7. Computes: `gain = (dphi - dplo) / (aphi - aplo)`, `off = dphi - gain × aphi`
8. Writes `det_no  off/desgain  gain/desgain` to output file
9. Writes `d_ecal.cmd` (GEBSort display script)

**Limitation:** peak finding is approximate (threshold-based, no Gaussian fit). Uses `dgs_ecal2.c` for more accurate peak fitting.

---

### `dgs_ecal2.c` — Energy Calibration Fitter v2 with Peak-Shape Fit (204 L)

Improved version of `dgs_ecal.c` using `pt_1sp()` for a proper peak-shape fit (Gaussian + skew).

```bash
dgs_ecal  cal_file_name  source  lowch  desgain
```

(Same command-line interface as `dgs_ecal.c`.)

**Differences from v1:**
- Works on detectors 1–128 (extended range: `dgs_ecal.c` only handles 1–110)
- Uses `pt_1sp(dim, sp, lowch, &peak1, &peak2, &area1, &area2, &ptval, &sigma1, &sigma2, &skew1, &skew2, &sumarea)` for each spectrum — a Gaussian+skew fit that finds two peaks simultaneously
- `pt_1sp()` returns peak centroids `peak1`/`peak2` (already in channel units); no manual threshold scanning
- Reports `peaks: peak1/kevch  peak2/kevch` (with `kevch=1.0` default, keV = channel)
- Same calibration formula and output format as v1

**`pt_1sp()` source:** `gebsort/pt_1sp.c` — spectrum peak-shape analysis routine.

---

## GRETINA Mode-1 and Mode-2 Sort Functions

_Source: `gebsort/bin_mode1.c` (1,839 L), `bin_mode2.c` (1,303 L). Code-read 2026-04-26._

### `bin_mode1.c` — GRETINA Mode-1 Data Sorter (1,839 L) ✅ verified 2026-04-26 — `wc -l bin_mode1.c`

Sorts **GRETINA Mode-1** data — the raw crystal-level decomposition output from GRETINA (waveform decomposed into interaction points). This is a pre-tracking stage: Mode-1 data contains `CRYS_INTPTS` structs per crystal hit.

**Key role:** `bin_mode1` applies Doppler correction and energy calibration to GRETINA crystal data so that downstream sorters (e.g., `bin_angcor_DGS`) receive processed interaction data rather than raw.

In the GEBSort chain, `bin_mode1` is typically listed before any sorter that needs Doppler-corrected interaction points. The sort function populates the `exchange` struct with processed data.

### `bin_mode2.c` — GRETINA Mode-2 Data Sorter (1,303 L) ✅ verified 2026-04-26 — `wc -l bin_mode2.c`

Sorts **GRETINA Mode-2** data — compressed interaction-point data (fewer bytes than Mode-1, derived from `CRYS_INTPTS` after cropping unused slots). Produces energy and position spectra from `GEB_TYPE_DECOMP` payloads.

Mode-2 sorter is often combined with DGS (`bin_dgs`) in mixed GRETINA+Gammasphere experiments. Used to produce GRETINA-specific histograms (e.g., per-crystal energy spectra, multiplicity) within the same GEBSort run as DGS analysis.

---

---

## GEBSort Math & Utility Helpers

_Source: `gebsort/findAngle.c`, `findVector.c`, `findCAngle.c`, `utils.c`, `tlutil.c`, `mkMap.c`, `printEvent.c`, `validate.c`. Code-read 2026-04-26._

These are shared library-style helper files compiled into GEBSort or the GEBMerge pipeline. They provide geometry math, data-format conversion utilities, event printing, and event validation logic.

### `findAngle.c` — Opening Angle Between Two Unit Vectors ✅ verified 2026-04-26 — `findAngle.c:L1-34`

Single function: `int findAngle(float n1[3], float n2[3], float *th)`. Computes the opening angle `*th` (in radians) between two 3D unit vectors using their dot product. Clamps the dot product to `[-1, 1]` before calling `acosf()` to guard against floating-point roundoff. Used by angular correlation sorters (`bin_angcor_DGS.c`, `bin_angcor_GT.c`) to find the angle between two gamma-ray directions.

### `findVector.c` — Normalized Difference Vector ✅ verified 2026-04-26 — `findVector.c:L1-51`

Single function: `int findVector(float x1,y1,z1, float x2,y2,z2, float *v1, *v2, *v3)`. Computes the unit vector pointing from point `(x1,y1,z1)` to `(x2,y2,z2)` — i.e., `v = (p2-p1)/|p2-p1|`. Used to construct gamma-ray direction vectors from two interaction points (e.g., first Compton interaction → second interaction defines the Compton scattering direction).

### `findCAngle.c` — Compton Scattering Angle ✅ verified 2026-04-26 — `findCAngle.c:L1-48`

Single function: `float findCAngle(float eg, float ee, float *thc)`. Computes the predicted Compton scattering angle `*thc` from:
- `eg` — incident gamma-ray energy (MeV)
- `ee` — energy deposited in the first Compton scatter (MeV)

Using the Compton kinematics formula: `cos(θ) = 1 + 0.511/Eγ - 0.511/Eγ'` where `Eγ' = Eγ - Ee`. Returns a non-zero float if the kinematics are unphysical (|cosθ| > 1), which signals the tracking algorithm to reject the event. Used in `bin_angcor_DGS.c` for Compton imaging.

### `utils.c` — 24-Bit Signed Integer Converters ✅ verified 2026-04-26 — `utils.c:L1-187`

Low-level data format helpers:

| Function | Description |
|---|---|
| `pprint_32(str, vv)` | Pretty-prints a 32-bit word in decimal/hex/binary form for debugging |
| `c32bit24bit(int vv)` | Converts a signed 32-bit int to 24-bit two's complement unsigned form |
| `c24bit32bit(uint vv)` | Converts a 24-bit unsigned two's complement value to a signed 32-bit int |
| `twoscomp_to_int_24(uint tempE)` | Converts a 24-bit unsigned value via left/right shift (sign-extend trick from Shaofei Zhu) |
| `test_convert()` | Self-test that exercises all conversion functions and exits — not called in production |

These functions handle the 24-bit signed energy encoding used in DGS/GRETINA digitizer payloads where energy words are packed into 3 bytes within the GEB event header.

### `tlutil.c` — Peak-Finding & Spectrum Math Library ✅ verified 2026-04-26 — `tlutil.c:L1-514`

A collection of standalone numerical utility functions used by calibration tools (`dgs_ecal.c`, `dgs_ecal2.c`, `fwhm_onepeak.c`) and some sorters:

| Function | Description |
|---|---|
| `zero_cross(y1,x1,y2,x2)` | Finds x-axis zero crossing of a line through two points |
| `find_parab_vertex(x1,y1,x2,y2,x3,y3,*xv,*yv)` | Fits a parabola through 3 points and returns the vertex — used in `fwhm_onepeak.c` to find the PZ minimum |
| `f1_peak(sp[], cutfac, lo, hi, *peak, *area, *sig, *skew)` | Finds a single peak in a 1D spectrum slice using centroid + moment analysis; returns peak position, area, sigma, and skew. Searches `[lo,hi]`, smooths 5×, applies a fractional-max trigger level `cutfac`. |

The `f1_peak` function is a simple non-Gaussian peak finder suitable for finding energy centroids in raw spectra without a full fit.

### `mkMap.c` — Static Detector ID Map Generator ✅ verified 2026-04-26 — `mkMap.c:L1-106`

Standalone program that prints a fixed GEBSort detector ID map to stdout, using hardcoded ID ranges. Unlike `mk_dgs_map.c` (which queries live EPICS PVs), `mkMap.c` generates a complete map for a predefined geometry:

| ID Range | Type | Count |
|---|---|---|
| 1010 – 1460 (stride 20) | BGO (5 per block), GE (5), SIDE (5), AUX (5) × 41 blocks | 41 each |
| 2000 – 2319 | DSSD strips | 320 |
| 2320 – 2339 | Focal Plane channels | 20 |
| 2340 – 2359 | X-Array channels | 20 |
| 3000 – 3010 | CHICO2 channels | 11 |

Output format per line: `<ID>  <type_int>  <local_index>  <type_name>` — the map file format consumed by GEBSort's `bin_dgs.c` and related sorters to translate raw GEB IDs to detector names. Requires `GTMerge.h` for type integer constants (BGO, GE, SIDE, AUX, etc.).

**Contrast with `mk_dgs_map.c`:** That tool queries live EPICS PVs (`GS###_Dig_Channel`, `GS###_Dig_Index`, `GS###_VME_Index`, `VMExx:MDIGy:user_package_data_RBV`) to generate the map dynamically from the actual detector configuration.

### `printEvent.c` — GEB Event Pretty-Printer ✅ verified 2026-04-26 — `printEvent.c:L1-649`

A large utility module (649 lines) providing human-readable print functions for all major GEB event payload types. Used for debugging and validation output controlled by `Pars.NumToPrint`.

| Function | Prints |
|---|---|
| `get_GEB_Type_str(type, str)` | Maps GEB type integer → human-readable string (e.g., type 14 → `"GEB_TYPE_DGS"`) |
| `print_S800PHYSDATA(fp, dirk)` | S800 physics data struct: CRDC x/y, IC sum, TOF, trigger, ata/bta/dta/yta angles |
| `printCRYS_INTPTS(fp, TT, DG)` | GRETINA crystal interaction point struct: crystal_id, tot_e, t0, chisq, norm_chisq, per-interaction x/y/z/e/seg fields |

The `ctk.h` header (not `GEBSort.h`) is used — suggesting this file predates the standard GEBSort framework and may have originated in the GRETINA tracking codebase.

### `validate.c` — GEB Event Filter / Gate Function ✅ verified 2026-04-26 — `validate.c:L1-421`

The `validate(GEB_EVENT *GEB_event)` function acts as a **pre-sort event gate**: it returns 0 (reject) or 1 (accept) for each incoming event before any `bin_xxx` function is called. GEBSort calls `validate()` after building the coincidence event window and before dispatching to sort functions.

**Active logic (always applied):**
- Counts `GEB_TYPE_DECOMP` hits above `Pars.minCCe` energy threshold → `emode2`
- Rejects events where `emode2 < Pars.minNumCC` or `emode2 > Pars.maxNumCC`
- After that check: returns 1 unconditionally (the rest of the function body is wrapped in `#if(0)` / `#if(1)` conditional blocks and is disabled)

**Disabled logic (compiled out, `#if(0)`):**
- Segment-uniqueness check: rejects events where any two interaction points share the same crystal segment
- GRETINA tracking quality checks (chisq, norm_chisq thresholds)
- Event multiplicity cap (`GEB_event->mult >= MAX_GAMMA_RAYS`)

In the current build, `validate()` effectively only applies the CC-count multiplicity gate (`minNumCC`/`maxNumCC`). All other filters are disabled.

---

## ROOT CINT Macro Helpers (`bar.cc`, `curve.cc`) ✅ verified 2026-04-27 — `gebsort/bar.cc` (43 L) + `gebsort/curve.cc` (30 L)

_Source: `gebsort/bar.cc`, `gebsort/curve.cc`. Code-read 2026-04-27. These are GRETINA tracking analysis macros, not DGS-specific._

Two small ROOT CINT macro snippets for interactive GRETINA tracking analysis. They are **not** compiled into any GEBSort binary — they are loaded directly in a ROOT interactive session via `.L bar.cc` or `{ ... }` syntax.

### `bar.cc` — ROOT TControlBar for GRETINA GUI (43 L)

Defines a ROOT `TControlBar` GUI panel (vertical layout, titled "GRETINA GEB") with clickable buttons for common GRETINA tracking analysis actions:
- Load data files: `GTDATA/wsi.root`, `nsi.root`, `test.root` via `dload()`
- Update spectra (`update()`), list objects (`ls()`), create canvas (`mkcanvas()`)
- Display histograms: `dtbtev` (1D/2D), `CCsum`, `hitpat`, `radius_all`, `SMAP_allhits`, `rate_mode2_min`, `CCe` matrix, `fm` (figure of merit), `fomXe` projections at FOM thresholds 0.5–2.0
- The FOM projections (`pjx("fomXe","x",0,N)`) demonstrate the Compton tracking Figure of Merit gate sweep used for peak-to-background optimization.

**Context:** `GSUtil.cc` must be compiled first (first button: `.L GSUtil.cc++`). This is a GRETINA interactive analysis utility — predates DGS; retained in the `gebsort/` directory as legacy GRETINA support.

### `curve.cc` — FOM Curve Export Script (30 L)

A ROOT CINT script body (no function wrapper — executed as a block) that sweeps FOM thresholds 0.1–2.05 in steps of 0.1 and exports:
- For each threshold `N`: `pjx("fomXe","x",0,N)` → `wrspe("x","fomNN.spe")` — projects the `fomXe` 2D matrix onto X at FOM < N and writes as a Radford `.spe` spectrum file.
- Additionally exports: `CCadd.spe`, `CCsum.spe`, `CCadd_raw.spe`, `CCsum_raw.spe`, `fm.spe`, `z_plot.spe`.

**Purpose:** Generates a full set of FOM-gated spectra for peak-to-background curve analysis — used to determine the optimal tracking FOM cut for a given experiment. Output `.spe` files are read by `gf3` (Radford spectrum viewer).

---

## Cross-References

- `knowledgeBase/gebsort.md` — Core GEBSort framework: GEBMerge, bin_dgs, jta, calibration workflow
- `knowledgeBase/gebsort_merge_receive.md` — GEB utility tools: GEBFilter, GEBCrop, GEBSplit, dtbtev
- `knowledgeBase/tac2.md` — TAC-II TDC hardware and packet format (consumed by `bin_tac2`)
- `knowledgeBase/data_structures.md` — GEB binary format input used by all sorters
- `knowledgeBase/dgs_analysis.md` — Modern parquet_pysort alternative pipeline

---
*Created: 2026-04-26. Split from gebsort.md. Source: `DGS_tools_pack/gebsort/`. Updated 2026-04-26: Math/utility helpers section added. Updated 2026-04-27: bar.cc + curve.cc GRETINA ROOT macro helpers added.*
