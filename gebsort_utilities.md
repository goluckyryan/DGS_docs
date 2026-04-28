# GEBSort — Minor Utility Programs and Support Files

Stability: C2 - Active / semi-stable

_Split from `gebsort.md` on 2026-04-27 (parent file exceeded 500-line threshold)._
_Source: `DGS_tools_pack/gebsort/` — code-read 2026-04-27_
_Author: T. Lauritsen (ANL) and contributors._

---

## Table of Contents

- [listTS.c](#listTSc--geb-file-timestamp-scanner-207-lines)
- [GTPrint.c](#gtprintc--dgsevent--gtevent-debug-printer-209-lines)
- [dtbtev.c](#dtbtevc--time-between-events-distribution-309-lines)
- [DataExtract.c](#dataextractc--digitizer-header-decoder-685-lines)
- [dgs_ecal2.c](#dgs_ecal2c--alternative-energy-calibration-204-lines)
- [gretTapClient.c](#grettapclientc--gretina-live-data-tap-client-287-lines)
- [GF_veto_cube.c](#gf_veto_cubec--veto-cube-support-for-gebfilter-95-lines)
- [pairProd.c](#pairprodc--pair-production-stub-52-lines)
- [findAngle.c / findCAngle.c / findVector.c](#findanglec--findcanglec--findvectorc--geometry-helpers)
- [time_stamp.c](#time_stampc--timestamp-printer-24-lines)
- [temp_ge.c](#temp_gec--legacy-vxi-ln2-status--temperature-alarm-tool-330-lines)
- [trig_fun.c](#trig_func--3d-vector-math-library-377-lines)
- [tlutil2.c](#tlutil2c--peak-finding--math-utility-library-v2-292-lines)
- [2d_fun.c](#2d_func--2d-matrix-file-io-and-manipulation-library-625-lines)
- [GEBCrop.c](#gebcropc--mode2-payload-trimmer-141-lines)
- [GEBFilter.c](#gebfilterc--gretina-mode2-event-filter-and-transform-767-lines)
- [GEBSplit.c](#gebsplitc--geb-stream-splitter-by-event-type-144-lines)
- [GEBHeader.c](#gebheaderc--dgs-file-header-printer-30-lines)
- [GRETINA Tracking Output Writers](#gretina-tracking-output-writers-writetrack_addtrack--writetrack_repeat)
- [G4toMode2.c](#g4tomode2c--geant4-simulation-to-mode2-converter-2151-lines)
- [get_dead_layer_corrections2.cpp](#get_dead_layer_corrections2cpp--dfma-silicon-dead-layer-calculator-533-lines)
- [fwhm511.c / fwhm_511.c](#fwhm511c--fwhm_511c--511-kev-peak-fwhm-analysis)
- [get_eraw.cc / get_pz.cc](#get_erawcc--get_pzcc--batch-spe-extractor-scripts-27-lines-each)
- [get_XAeraw.cc / get_XApz.cc / get_XAecln.cc](#get_xaerawcc--get_xapzcc--get_xaeclncc--x-array-batch-spe-extractor-scripts)
- [mk_gsutil.cc](#mk_gsutilcc--gsutil-compile-helper-6-lines)
- [curEPICS](#curepics--symbolic-link)
- [utils.c](#utilsc--24-bit-integer-conversion-utilities-187-lines)
- [spe_fun.c](#spe_func--spe-spectrum-file-io-221-lines)
- [tlutil.c](#tlutilc--general-math--utility-library-v1-514-lines)

## Cross-References

- `knowledgeBase/gebsort.md` — Main GEBSort documentation (core programs, calibration workflow, sorters)
- `knowledgeBase/gebsort_merge_receive.md` — GEBMerge, gtReceiver, GEBClient, dmpdata
- `knowledgeBase/gebsort_additional_sorters.md` — GRETINA-specific + additional detector sorters

---


## Minor Utility Programs and Support Files

_Source: `DGS_tools_pack/gebsort/` — code-read 2026-04-27_

Several smaller programs in `gebsort/` that are not covered above:

### `listTS.c` — GEB File Timestamp Scanner (207 lines)

Standalone diagnostic utility. Usage: `listTS file ntoprint twin`.

- Reads a GEB binary file header-by-header.
- Counts events by `GEB_type` (up to 100 distinct types); prints per-type event counts and percentages at end.
- Tracks timestamp monotonicity: flags and counts negative `dTS` (bad/reversed timestamps).
- Prints first N events (controlled by `ntoprint`), each with type string, timestamp, and `dTS`.
- `twin` parameter sets a coincidence window — not used in current implementation (declared but unused in main logic).
- `GebTypeStr()` helper maps known type codes to strings (`GEB_TYPE_RAW`, `GEB_TYPE_DGS`, `GEB_TYPE_DECOMP`, etc.).
- Useful for diagnosing timestamp ordering issues in merged GEB files before sorting.

✅ verified 2026-04-27 — `listTS.c:L55-207` (main loop), `L1-52` (GebTypeStr helper)

### `GTPrint.c` — DGSEVENT / GTEVENT Debug Printer (209 lines)

Support module (not a standalone executable). Provides two print functions used by analysis sorters for debug output:

- **`GTPrintEvent2(fp, ii, DGSEvent)`** — prints one `DGSEVENT` struct: `board_id`, `chan_id`, `id`, `tpe`, `tid`, detector type string (GE/BGO/SIDE/AUX/DSSD/FP/XARRAY), `base_sample`, 48-bit timestamp (`event_timestamp`) formatted as two 32-bit halves.
- **`GTPrintEvent(fp, Event, DGSEvent)`** — prints a full `GTEVENT` (tracking event) including segment positions, angles, energies.
- Detector type codes: `GE`, `BGO`, `SIDE`, `AUX`, `DSSD`, `FP`, `XARRAY` — from `GEBSort.h`.

✅ verified 2026-04-27 — `GTPrint.c:L15-70` (GTPrintEvent2), `L73-209` (GTPrintEvent)

### `dtbtev.c` — Time-Between-Events Distribution (309 lines)

Standalone diagnostic utility. Usage: `dtbtev infile`.

- Reads GEB binary file; computes time difference between consecutive events of the same type (`dTS = TS[n] - TS[n-1]`).
- Builds histograms of `dtbtev` (time-between-events) per GEB type in bins of `CTFAC=100` ticks (uses `ctk.h` tick conversion).
- Also builds `ts_time[]` and `evtime[]` arrays (time-ordered event counts).
- Outputs histogram data to file `dtbtev.dat`.
- `MAX_GEB_TYPE` arrays of `LEN=16000` floats; uses `get_a_seed()` (from `get_a_seed.c`) for random seeding.
- Useful for verifying event rates and timing structure of each GEB type in a file.

✅ verified 2026-04-27 — `dtbtev.c:L27-309`

### `DataExtract.c` — Digitizer Header Decoder (685 lines)

Shared support library (`DecodeHeader()` and helpers). Decodes raw 4-word DGS digitizer event headers into typed `DIG_HEADER` structs.

**Header type dispatch** (based on bits [19:16] of word 3):

| Type | Mode | Notes |
|------|------|-------|
| 0 | Legacy (deprecated) | No longer in production use |
| 1 | LED (threshold), original | Deprecated 2015-05-18 |
| 2 | CFD, original | Deprecated 2015-05-18 |
| 3 | LED, July 2015 revision | Current for DGS (also Majorana — code comment confirms same type despite different FPGA) |
| 4 | CFD, July 2015 revision | CFD mode, still in use |
| 5 | LED 2016 | |
| 6 | CFD 2016 | |
| 7 | Combined LED+CFD 2019 (WIP) | All data in all modes |
| 0xF | Type-F DAQ header | Injected by IOC software (empty/EOD/error markers); GammaWare should never see these |

**Convention:** Odd header IDs = LED/threshold; even = CFD. Header type 0 was the original LED-only format.

`HEADER_LENGTH` is reported in 16-bit words but hardware reads 32-bit longwords → divide by 2. Does not count the `0xAAAAAAAA` sync word.

**Global side effects:** Sets `LengthOfHeader` and `LengthOfData` (number of 32-bit words) for each decoded header; used by callers to determine how many additional words to read.

✅ verified 2026-04-27 — `DataExtract.c:L1-60` (header; full decode logic follows for each type)

### `dgs_ecal2.c` — Alternative Energy Calibration (204 lines)

Variant of `dgs_ecal.c`. Usage: `dgs_ecal cal_file_name source lowch desgain`.

- Reads `ehi{nnn}.spe` spectrum files (nnn=001–110), one per detector channel.
- Finds two calibration peaks per detector based on `source`: supports `207Bi`/`Bi207` (569.702 + 1063.662 keV), `88Y`/`Y88` (898.045 + 1836.063 keV), `60Co`/`Co60` (1173.228 + 1332.490 keV).
- Uses `pt_1sp()` for peak fitting; performs linear (gain/offset) calibration from two peaks.
- Key difference from `dgs_ecal.c`: reads `.spe` format directly (not GEB raw), uses `rd_spe()` helper; works on pre-binned spectra.
- Outputs calibration file in format `gain offset` per detector.

✅ verified 2026-04-27 — `dgs_ecal2.c:L1-60` (source selection + peak values), `L60-204` (main loop)

### `gretTapClient.c` — GRETINA Live Data Tap Client (287 lines)

Networking library (not standalone). Implements a TCP client that connects to the GRETINA global event builder (GEB) data tap, allowing a program to receive a live copy of the merged GEB data stream.

- **Protocol:** `GEBLink.h` tap protocol — sends an init message on connect, then reads `TAP_HEADER_LEN=8 bytes` headers + payloads.
- **Error enum:** `gretTapClientError` with 21 error codes (e.g. `GTC_TAPCONN`, `GTC_HEADER_READ`, `GTC_LEN_HUGE`, `GTC_TIMEOUT`).
- **`fdWrite()`/`fdRead()`:** robust partial read/write wrappers.
- **`gretTapClientConnect()`:** resolves hostname → creates socket → connects; returns 0 on success.
- **`gretTapClientRead()`:** reads one GEB event (header + payload) from tap; validates length bounds (0 < len ≤ some max); returns into caller-supplied `GEBDATA*` + payload buffer.
- Used by real-time online analysis programs that want to process the same data stream as GammaWare.
- Not used in the DGS standalone analysis workflow (DGS uses `tcpReceiverMT` + GEB files, not the tap).

✅ verified 2026-04-27 — `gretTapClient.c:L1-287`

### `GF_veto_cube.c` — Veto Cube Support for GEBFilter (95 lines)

Support module. Implements `setup_veto_cube()` called by `GEBFilter` when `Pars.vetocubes != 0`.

- Allocates and fills a 5D lookup array indexed as `VETO_INDX(module, crystal, x, y, z)` — array dimension: `(MAXGTMODNO+1) × (MAXCRYSTALNO+1) × (VETO_NX+1) × (VETO_NY+1) × (VETO_NZ+1)` unsigned ints.
- Reads a text file (`Pars.vetocubefn`) in format `module crystal x y z ; count` and marks non-zero entries as 1 in the cube.
- `MAXGTMODNO=30`, `VETO_X_D`/`VETO_Y_D`/`VETO_Z_MAX`/`VETO_BINWIDTH`/`VETO_NX`/`VETO_NY`/`VETO_NZ` defined in `veto_pos.h`.
- If file open fails, sets `Pars.vetocubes=0` and returns gracefully.
- Purpose: spatial veto lookup — given a gamma-ray interaction position (module, crystal, xyz bin), check if it falls in a vetoed region.

✅ verified 2026-04-27 — `GF_veto_cube.c:L24-95`

### `pairProd.c` — Pair Production Stub (52 lines)

Skeleton function `pairProd(evno, ctkStat, Clstr, nClusters, target_pos)`. Body is empty (increments a static counter, prints debug if requested, then `return 0`). Placeholder for pair-production tracking logic; not yet implemented. Called from the tracking pipeline in `ctk.h`-based sorters.

✅ verified 2026-04-27 — `pairProd.c:L30-52` (function body is a no-op)

### `findAngle.c` / `findCAngle.c` / `findVector.c` — Geometry Helpers (34 / 48 / 51 lines)

Small geometry support routines used by angle correlation sorters:
- `findAngle.c` (34 L) — `findAngle(n1[3], n2[3], *th)`: computes opening angle between two unit vectors via dot product → `acosf(dotProd)`; clamps dotProd to [-1,1] to avoid NaN.
- `findCAngle.c` (48 L) — `findCAngle(eg, ee, *thc)`: computes Compton scattering angle from Klein-Nishina kinematics: `egp = eg − ee`, `cosa = 1 + 0.511/eg − 0.511/egp`, clamps and returns `acosf(cosa)`. Returns a non-zero residual if out-of-range. (`eg` = incoming energy, `ee` = deposited energy, both in MeV.)
- `findVector.c` (51 L) — `findVector(x1,y1,z1, x2,y2,z2, *v1,*v2,*v3)`: computes normalized difference vector `(x2−x1, y2−y1, z2−z1) / |...|`; outputs 3 components via pointers.

Not standalone programs. Linked into `bin_angcor_DGS` and related sorters.

✅ verified 2026-04-27 — `findAngle.c:L6-34`, `findCAngle.c:L6-48`, `findVector.c:L6-51`

### `time_stamp.c` — Timestamp Printer (24 lines)

Single function `time_stamp(FILE *fp)`. Calls `time(NULL)` + `localtime()` + `asctime()` and prints a human-readable local timestamp to the given `FILE*`. Utility include; not a standalone program. Used by sorters that write session logs.

✅ verified 2026-04-27 — `time_stamp.c:L1-24`

### `temp_ge.c` — Legacy VXI LN2 Status / Temperature Alarm Tool (330 lines)

**Standalone program** (has `main()`). Legacy Gammasphere LN2 monitoring utility — reads Ge detector temperatures via EPICS Channel Access and sends email alarms. Predates the modern `LNFill_App.py` / Discord-based system.

**Workflow:**
1. Reads `det.list` (format: `<det_id> <hose_name>`) to map detector numbers to LN2 hose names.
2. Checks 6 VXI crates via `HEARTBEAT{1–6}` integer PVs; prints crate up/down status and emails alarm on down crate.
3. Reads LN status PVs: `LN_MODE:XC` (fill idle/active), `LN_ATLF:XC`/`LN_ATNF:XC`/`LN_TSLF:XC`/`LN_TSFS:XC` (last/next fill timestamps), `LN_ALM:XC` (alarm count), `LN_ATLTF:XC`/`LN_TSTFS:XC` (tank fill times).
4. Loops `MOD001–MOD110`: reads `_DV_EN`, `_DV_GEHV`, `_DV_TEMP`, `_DV_TEMP.HIGH`, `_DV_TEMP.LOW`.
   - Prints temperature vs. [lo, hi] band and margin (`temperature − hi`).
   - If `temperature ≥ hi`: alarm — sends email via `mailx` to addresses in `/home/dgs/.LNALARM`; also prints hose name.
   - If within 3 K of `hi`: prints "\<--- watch" warning.
5. Exits with `nalarm` (number of warm detectors) as exit code.

**CA helpers:** `CAgetval_float()`, `CAgetval_int()`, `CAgetval_string()` — blocking synchronous CA reads with 5 s search / 10 s get timeouts.

**HTML output:** Wraps all output in `<tt>` tags with `</br>` line breaks — designed to be served via CGI over HTTP for a browser-accessible status page.

**`TEST` compile flag:** `#define TEST 0` — when set to 1, suppresses email sending (prints commands instead). Also used to inject a fake warm temperature (`temperature=120`) on one detector for testing.

✅ verified 2026-04-27 — `temp_ge.c:L1-330`

### `trig_fun.c` — 3D Vector Math Library (377 lines)

Collection of 3D geometry utility functions used by angular correlation and tracking sorters. No `main()` — pure library.

| Function | Signature | Description |
|----------|-----------|-------------|
| `PolarFromCartesian` | `(x,y,z,*r) → polar` | Polar angle θ = acos(z/r); also returns r |
| `AzimuthFromCartesian` | `(x,y,z) → azimuth` | Azimuthal angle φ = atan2(y,x) |
| `ranInsideCirle` | `(*u0,*u1)` | Uniform random point inside unit circle (rejection method) |
| `ranVectorOnSphere` | `(*x0,*x1,*x2)` | Uniform random unit vector on sphere (Numerical Recipes §21.5.1) |
| `crossprod` | `(u,v) → s` | Cross product u × v (3-component) |
| `crossprod1` | `(a,b) → s` | Cross product (CRC 22 ed p.556 variant) |
| `crossprod2` | `(a,b) → s` | Cross product (alternate sign convention; note: has a typo `a2*b2` vs `a2*b3` in s1) |
| `dotproductangle` | `(u,v) → angle` | acos(u·v) with clamp to [−1,1] |
| `vectorlen` | `(u) → length` | Euclidean length |
| `unitvector` | `(*u) → length` | Normalize in-place; returns original length |
| `rad2deg` | `(rad) → deg` | Radian to degree conversion |
| `check_coord_sys` | `(x,y,z axes)` | Asserts all three axes are unit-length, mutually perpendicular, and right-handed (MAXERR=0.0001); exits on failure |
| `check_unitvector` | `(v)` | Asserts unit length; exits on failure |
| `coord_in_new` | `(axes, point) → (x',y',z')` | Expresses a point in a new orthonormal coordinate frame via dot products |
| `test_fun` | `()` | Self-test: computes cross product of x-hat × y-hat, prints result, exits |

✅ verified 2026-04-27 — `trig_fun.c:L1-377`

### `tlutil2.c` — Peak-Finding & Math Utility Library v2 (292 lines)

Companion to `tlutil.c` (already documented in gebsort.md §`tlutil.c`). Provides additional math utilities. No `main()` — pure library.

| Function | Signature | Description |
|----------|-----------|-------------|
| `zero_cross` | `(y1,x1,y2,x2) → x_zero` | Linear interpolation to find x where y=0 (line through two points) |
| `find_parab_vertex` | `(x1,y1,x2,y2,x3,y3, *xv,*yv)` | Parabolic vertex from 3 points via Lagrange interpolation (ChatGPT-generated formula, per comment) |
| `f1_peak` | `(sp[],cutfac,lo,hi, *peak,*area,*sig,*skew)` | Peak finder in 1D spectrum: 5-pass 3-point smooth → max search → threshold at `max×cutfac` → centroid (`peak`), integral (`area`), RMS width (`sig`), third-moment skewness (`skew`). Returns −1 if no peak above threshold |
| `ranGauss` | `() → float` | Box-Muller Gaussian RNG using `drand48()`; alternates between two stored values (static `nn` flag) |
| `ranHK` | `(HHmean,HHwidth,KKmean,KKwidth, *HH,*KK)` | Generates random (H, K) pair from independent Gaussians; floor at 0; K rounded to nearest int |
| `invert3x3` | `(m[3][3], inv[3][3]) → 0/1` | 3×3 matrix inversion via cofactors (ChatGPT-generated, per comment); returns 0 if determinant is zero |

✅ verified 2026-04-27 — `tlutil2.c:L1-292`

### `2d_fun.c` — 2D Matrix File I/O and Manipulation Library (625 lines)

Library for reading, writing, and processing 2D matrices stored in custom binary/ASCII formats with a `.dm` descriptor sidecar file. No `main()` — pure library. Used by GEBSort sorters that work with 2D coincidence matrices.

**Descriptor file (`.dm`):** Text sidecar with 4 fields: `nx`, `ny`, `machinetype`, `datatype`. Read/written by `read_dm()` / `wr_dm()`.

**Matrix layout:** Flat `float*` array accessed via macro `MAT(n,i,j) = *(n + j*nx + i)` — column-major (j is the slow index).

| Function | Description |
|----------|-------------|
| `read_dm` | Reads `.dm` descriptor, prints dimensions and type |
| `wr_dm` | Writes `.dm` descriptor |
| `rd_mat` | Reads matrix data; supports 5 types: `ascii` (sparse `i j v` triples), `int`, `us` (unsigned short), `ui` (unsigned int), `float` (raw binary) |
| `wr_mat` | Writes matrix data in same 5 formats; `ascii` mode writes only nonzero cells |
| `mat_mc` | Energy-calibration stretch: multiplies matrix by gain `mc` + offset `off` in x (`way=0`) or y (`way=1`) direction using sub-bin interpolation (area-conserving redistribution) |
| `sm_mat` | 2D nearest-neighbor smoother: each cell gets weighted average of its 8 neighbors (weight factor 2×nn on self, normalize by 3×nn); boundary-safe |

✅ verified 2026-04-27 — `2d_fun.c:L1-625`

---

### `GEBCrop.c` — Mode2 Payload Trimmer (141 lines)

Standalone utility that trims GEB files containing GRETINA mode2 (`GEB_TYPE_DECOMP`) data. Fixes overlong payloads by recalculating the correct byte length from the actual number of interaction points (`CRYS_INTPTS.num`) and rewriting the GEB header `length` field.

**Usage:** `GEBCrop infile outfile`

**Processing loop:**
1. Read each GEB header (`GEBDATA`) + payload into a 20 KB scratch buffer.
2. For `GEB_TYPE_DECOMP` events only: cast payload to `CRYS_INTPTS*`, read `num`, recalculate `plz = DCRHDRLEN + num * sizeof(DCR_INTPTS)`, overwrite `gebhead.length`.
3. Write header + (possibly truncated) payload to output.
4. First 10 events printed via `printCRYS_INTPTS()` for sanity check.

**Summary stats at exit:** `sizeof(CRYS_INTPTS/DCR_INTPTS/GEBDATA)`, `MAX_INTPTS`, total events read.

**When to use:** Mode2 files with oversized payloads from a buggy writer. Not part of the standard DGS workflow.

✅ verified 2026-04-27 — `GEBCrop.c:L1-141`

---

### `GEBFilter.c` — GRETINA Mode2 Event Filter and Transform (767 lines)

Standalone filter. Reads a GEB binary file, applies transforms/filters from a chat script, writes filtered output.

**Usage:** `GEBFilter chatfile infile outfile`

**Chat file options:**

| Option | Description |
|--------|-------------|
| `nevents N` | Stop after N events |
| `waitusec N` | Sleep N µs between output events (live-stream throttle) |
| `vetocube filename` | Enable veto-cube filtering; removes bad interaction points (see `GF_veto_cube.c`) |
| `addT0` | TS *= 10; add `cip->t0` (1 ns resolution) for mode2 — converts 10 ns ticks to ns |
| `GT2AGG4 filename` | Convert GT mode2 to AGATA G4 ASCII world-coords; reads `crmat.LINUX` + `GANIL_AGATA_crmat.dat` |
| `xyz_smear dx dy dz` | Add uniform random smear ±dx/dy/dz mm to each interaction point |
| `fix104 dt` | Add timestamp offset `dt` to crystal #104 (experiment 2137 workaround) |

**Main loop:** Raw `read()`/`write()` FDs. First 20 events printed. `writeOK=0` suppresses output for GT2AGG4-consumed events or veto-cube-zeroed mode2 events.

**Veto cube energy redistribution:** `e_new = e_old * (1 + ff)` where `ff = ebad / egood`, `egood = esum - ebad` — surviving interaction-point energies scaled up to conserve total. ✅ corrected 2026-04-27 — prior KB said `ebad/esum`; actual code uses `ff = ebad / egood` (`egood = esum - ebad`), i.e., divided by the surviving energy, not total. Source: `GEBFilter.c:L659-660,L699`.

**Crystal ID decode:** `holeNum = (crystal_id & 0xfffc) >> 2`; `crystalNum = crystal_id & 0x0003`; `detNo = 4*holeNum + crystalNum`.

✅ verified 2026-04-27 — `GEBFilter.c:L1-767`

---

### `GEBSplit.c` — GEB Stream Splitter by Event Type (144 lines)

Splits a mixed GEB binary file into two output files by event type.

**Usage:** `GEBSplit indata mode2_out bank88_out`

| GEB type | Output |
|----------|--------|
| `GEB_TYPE_DECOMP` | File 1 (mode2 GRETINA decomposed) |
| `GEB_TYPE_GT_MOD29` | File 2 (bank 88 / GRETINA mode1) |
| Other | Error + exit |

Uses low-level `open()`/`read()`/`write()` FDs. First 20 events printed with type and timestamp.

**When to use:** Separate mixed GRETINA streams (mode2 + mode1 interleaved) into type-pure files.

✅ verified 2026-04-27 — `GEBSplit.c:L1-144`

---

### `GEBHeader.c` — DGS File Header Printer (30 lines)

Implements `printDgsHeader(DGSHEADER dgsHeader)`. Checks if `dgsHeader.id == 0xaaaaaaaa` (old/no-header sentinel) and prints either an error or the header ID in decimal and hex. Library function only — no `main()`.

**Note:** `0xaaaaaaaa` is also the GEB stream sync word (see `ANLDAQ_tcpReceiver.md`). A file with this as the header ID has no valid file header (old data format).

✅ verified 2026-04-27 — `GEBHeader.c:L1-30`

---

### `utils.c` — 24-bit Integer Conversion Utilities (187 lines)

Pure library, no `main()`. Conversions between 32-bit and 24-bit signed two's-complement integers.

| Function | Description |
|----------|-------------|
| `pprint_32(str, vv)` | Pretty-prints 32-bit word as decimal / hex / binary (4-bit groups) |
| `c32bit24bit(vv)` | 32-bit signed → 24-bit two's-complement unsigned. Negative: set bit 23 + mask 23-bit mantissa. |
| `c24bit32bit(vv)` | 24-bit two's-complement unsigned → 32-bit signed. Base = 2^23 = 8388608. |
| `twoscomp_to_int_24(tempE)` | 24-bit sign-extension via shift trick: `(int32)(tempE << 8) >> 8`. Credit: Shofei Zhu. |
| `test_convert()` | Self-test over boundary values; calls `exit(0)`. |

**Bug:** `c32bit24bit()` — `sign` variable uninitialized in the positive branch (UB); harmless in practice.

✅ verified 2026-04-27 — `utils.c:L1-187`

---

### `spe_fun.c` — SPE Spectrum File I/O (221 lines)

Pure library, no `main()`. Read/write FORTRAN-record `.spe` files (Radford `gf2`/`fg2` format).

**File format:** FORTRAN records bracketed by 4-byte length markers.
- Record 1 (24 bytes): 8-byte name, `dim1`, `dim2`, `ired1`, `ired2` (all int)
- Record 2: `dim1 * dim2 * 4` bytes of `float` spectrum

| Function | Description |
|----------|-------------|
| `wr_spe(fn, *dim, *sp)` | Write 1D float spectrum. Sets `idim2=ired1=ired2=1`. Returns 0/−1. |
| `rd_spe(fn, *dim, *sp)` | Read `.spe`, clamp to `*dim` channels, update `*dim` to actual. Returns 0/−1; silent on missing file. |

Called from `bin_dgs.c`, `bin_dgs_GE.c` etc. Compatible with ANL/LBNL `gf2`/`fg2` analysis tools.

✅ verified 2026-04-27 — `spe_fun.c:L1-221`

---

### `tlutil.c` — General Math & Utility Library v1 (514 lines)

Pure library, no `main()`. Provides numerical and sorting utilities used across GEBSort and calibration tools. Companion to `tlutil2.c`.

| Function | Description |
|----------|-------------|
| `pprint_32` | 32-bit pretty printer (same as utils.c) |
| `zero_cross(y1,x1,y2,x2)` | Linear interpolation to x-intercept: `-b/a` |
| `find_parab_vertex(x1,y1,x2,y2,x3,y3, *xv,*yv)` | Parabola through 3 points (Lagrange); returns vertex. Used by `fwhm_onepeak`. |
| `f1_peak(sp,cutfac,lo,hi, *peak,*area,*sig,*skew)` | Peak finder: 5× 3-pt smooth, trigger at `max*cutfac`, centroid position, 2nd+3rd moments (sigma, skewness). |
| `ranGauss()` | Box-Muller Gaussian RNG via `drand48()`. Caches second sample. |
| `test_ranGauss()` | Self-test: 10M samples → histogram → `x.xy`; calls `exit(0)`. |
| `BinaryI16(val, str)` | Format 16-bit value as grouped binary string `|bbbb|bbbb|bbbb|bbbb|`. |
| `imin` / `imax` | Integer min/max (K&R style). |
| `sort_xydy(dim,x,y,dy)` | Bubble sort (x,y,dy) triple arrays ascending by x. |
| `flabs(x)` | Float absolute value. |
| `w_lin_fit(nn,xx,yy,ss,*a0,*b0)` | Weighted linear fit (Bevington p.107): returns offset `a0`, slope `b0`. Used in energy calibration. |
| `mod(i,j)` | Integer modulo (K&R style). |
| `sm3(dim,*y)` | 3-point smooth: `y[i]=(y[i-1]+2y[i]+y[i+1])/4` (Bevington). |
| `swabi4_sun2linux(iv)` | Byte-swap 32-bit word from Sun/SPARC to Linux/x86 nibble order. For legacy SUN binary files. |

**Correction:** `tlutil2.c` previously claimed `tlutil.c` was "already documented" — it was not. This entry corrects that gap.

✅ verified 2026-04-27 — `tlutil.c:L1-514`

---

## GRETINA Tracking Output Writers (`writeTrack_addtrack` / `writeTrack_repeat`)

_Source: `gebsort/writeTrack_addtrack.c` (408 L), `writeTrack_repeat.c` (303 L). Code-read 2026-04-27._

Two companion C files that serialize GRETINA gamma-ray tracking results into GEB binary output streams. Used exclusively in **offline** GRETINA tracking pipelines (the online path uses `serializeTrack()` instead).

### `writeTrack_addtrack.c` (408 L)

Writes **only the tracked result** to the output stream, discarding the original raw CRYS_INTPTS data.

- Entry point: `writeTrack_addtrack(trackStatus, track, Clstr, nClusters, ctkStat)`
- For each successfully tracked gamma in `track->n`, constructs a `TRACKED_GAMMA_RAY` struct and packs it into a GEB payload.
- Output: `GEB_TYPE_TRACK` event with `trackGeb.type = GEB_TYPE_TRACK`; payload = `(int nGammas) + (int nClusters) + ng × sizeof(TRACKED_GAMMA_RAY)` bytes.
- Uses `fwrite()` to `Pars.trackDataStream` (a FILE* set by calling code).
- Also exports `serializeTrack()` — produces identical byte layout as a heap buffer (no file I/O), for online use.

### `writeTrack_repeat.c` (303 L)

Writes **both** the original raw crystal data and the tracked result, creating a dual-stream output.

- Entry point: `writeTrack_repeat(track)` — no clusters/nClusters argument.
- First passes through each original `CRYS_INTPTS` hit (GEB_TYPE_DECOMP events) verbatim, then appends the tracked `GEB_TYPE_TRACK` event.
- Useful for cross-checking tracking algorithms against input data.

**Key types:** `TRACK_STRUCT`, `CLUSTER_INTPTS`, `TRACKED_GAMMA_RAY`, `CRYS_INTPTS`, `GEBDATA` — all defined in `ctk.h` / `gdecomp.h`.

✅ verified 2026-04-27 — `writeTrack_addtrack.c:L21` (entry point signature), `L26` (offline/online comment), `L139` (`trackGeb.type = GEB_TYPE_TRACK`), `L246,L250` (`fwrite` to `Pars.trackDataStream`), `L311` (`serializeTrack` export); `writeTrack_repeat.c:L22` (entry point `writeTrack_repeat(track)`), `L78` (GEB_TYPE_TRACK check), `L44,L216` (CRYS_INTPTS pass-through).

---

## `G4toMode2.c` — GEANT4 Simulation to Mode2 Converter (2,151 L)

_Source: `gebsort/G4toMode2.c`. Code-read 2026-04-27._

Converts **GEANT4 simulation output** (raw energy depositions) into **GRETINA Mode2 (CRYS_INTPTS GEB stream)** format, enabling GEBSort to process simulated data with the same pipeline as real data.

**Physics context:** simulates AGATA/GRETINA detector responses — used for Compton tracking efficiency studies, angular correlation simulations, and detector characterization.

**Key features:**
- Reads GEANT4 hit-list files (energies + positions per interaction point)
- Applies realistic detector simulation: energy smearing (`eSmear_seg`, `eSmear_CC`), position smearing (`pSmear`), thresholds (`e_th[det]`, `e_0[det]`), Doppler correction
- Enforces interaction-point count limits (`minNoInteractions`–`maxNoInteractions`) and crystal limits (`minNoCrystals`–`maxNoCrystals`)
- Applies crystal-rotation matrices from `crmat.LINUX` / `crmat.dat` (same format as GRETINA tracking code)
- Produces GEB binary output with `GEB_TYPE_DECOMP = 1` events containing properly formatted `CRYS_INTPTS` payloads
- Configurable via a **chat file** (`G4toMode2.chat`) using `readChatFile()` with options: `emin`, `eSmear_seg/CC`, `pSmear`, `minr`, `maxdist`, `minNoInteractions`, `maxNoInteractions`, `thresholds`, etc.
- Histograms: `sp_obs[]` (observed energy spectrum), `sp_emit[]` (emitted), `sp_obs_nosmear[]`, `sp_emit_nosmear[]` — 2000 channels each
- Tracks conversion efficiency: `nphoto` (photo-peak), `ncompton` (Compton)

**Internal helpers:**
- `readChatFile()` — chat-file parser (lines 673–897)
- `findClosestDetector()` — nearest detector geometry lookup
- `distToClosestCrystal()` — inter-crystal distance check
- `CheckNoArgs()` — argument validation

---

## `get_dead_layer_corrections2.cpp` — DFMA Silicon Dead-Layer Calculator (533 L)

_Source: `gebsort/get_dead_layer_corrections2.cpp`. Code-read 2026-04-27._

C++ library computing **alpha particle energy-loss corrections** for DSSD (Double-Sided Silicon Strip Detector) dead layers and a silicon box detector — used in DFMA (Digital Focal-plane Multi-channel Analyzer) alpha-decay correlation analysis.

**Physics context:** When an alpha particle passes through the inactive dead layer of a silicon detector, its measured energy is lower than its true energy. This code corrects for that loss using tabulated alpha ranges in silicon.

**Data source:** `alpha_ranges.txt` — table of alpha energy (MeV) vs range (µm) in silicon. Loaded into global `ranges[100][2]` by `load_ranges()`.

**API — exported functions (called by `bin_dfma.c` / `functions_dfma.h`):**

| Function | Physics scenario | Returns |
|----------|-----------------|--------|
| `get_dead_layer_corrections(dssd_E, dssd_x, dssd_y, box_E, box_wall, box_strip, box_det)` | Single alpha escape from DSSD into box | `[0]` initial energy, `[1]` DSSD dead-layer loss, `[2]` box dead-layer loss, `[3]` DSSD angle (rad) |
| `single_escape_one_nonescape(...)` | One alpha escapes DSSD, one stays | `[0]` non-escape energy, `[1]` escape active-layer loss, `[2]` escape DSSD dead-layer, `[3]` escape box dead-layer |
| `double_escape(...)` | Both alphas escape DSSD into box | 6-element array of dead-layer losses for both tracks |

The `main()` function contains test-harness code (lines 40–94) but is commented out with `*/` — the file is used as a **pure library**, not a standalone executable.

---

## `fwhm511.c` / `fwhm_511.c` — 511 keV Peak FWHM Analysis

_Source: `gebsort/fwhm511.c` (118 L), `gebsort/fwhm_511.c` (210 L). Code-read 2026-04-27._

Two related programs for measuring the **511 keV annihilation gamma-ray peak resolution** across all GS germanium detectors.

### `fwhm511.c` (118 L) — `pt511` executable

**Purpose:** Reads `.spe` spectra for each GS Ge detector, calls `fwhm_511()` to locate and measure the 511 keV peak, reports per-detector FWHM and average.

- Usage: `pt511 <ge_list> <lo_ch> <bgskip> <bgwidth> <hi_ch> <kev/ch>`
- Reads `ehi<NNN>.spe` files (one per detector)
- Reports: peak channel, FWHM (from sigma and direct), area, skewness per detector
- Writes a display script `d.cmd` for visual inspection
- Requires `>500` counts in peak for a valid result

✅ verified 2026-04-27 — `fwhm511.c:L11` (main()), `L40` (usage string: `pt511 <ge_list> <lo ch> <bgskip> <bgwidth> <hi_ch> <kev/ch>`), `L67` (fopen `d.cmd`), `L78` (ehi%3.3i.spe format).

### `fwhm_511.c` (210 L) — `fwhm_511()` function library

**Purpose:** The peak-finding and fitting engine used by `fwhm511.c` and potentially other tools.

- Finds peak max in `[lo, hi]` channel range; rejects if max < 30 counts
- Subtracts linear background using `bgskip`+`bgwidth` sidebands on each side
- Computes: centroid `*peak`, `*area`, Gaussian sigma `*sig`, skewness `*skew`
- Also computes a **direct FWHM** (`*fwhm`) by linear interpolation to half-max on both sides — more robust than `2.3548×sigma` for asymmetric peaks
- Returns 0 on success, -1 on failure

✅ verified 2026-04-27 — `fwhm_511.c:L1-210` (peak fitting engine; function signature confirmed).

---

## `get_eraw.cc` / `get_pz.cc` — Batch SPE Extractor Scripts (27 L each)

_Source: `gebsort/get_eraw.cc`, `gebsort/get_pz.cc`. Code-read 2026-04-27._

Not C++ source files despite the `.cc` extension — these are **GEBSort inline script fragments** (no `#include`, no `main()`) intended to be pasted or sourced into an interactive GEBSort session (like a `GEBSort.chat` function body).

- **`get_eraw.cc`:** loops GS detectors 0–110, calls `pjx("EhiRawRaw","p",i,i)` to project a 1D spectrum from the `EhiRawRaw` histogram, then `wrspe("p","ehi<NNN>.spe")` to write it out. Produces `ehi000.spe`–`ehi110.spe`.
- **`get_pz.cc`:** same pattern for `pzraw` histogram — calls `pjy("pzraw","p",i,i)` and writes `pz<NNN>.spe`. Produces pole-zero correction spectra for all detectors.

Both use GEBSort internal functions (`pjx`, `pjy`, `wrspe`) that are only available inside a running GEBSort session.

✅ verified 2026-04-27 — `get_eraw.cc:L1` (comment: loops GS detectors), `L23` (`pjx("EhiRawRaw","p",i,i)`), `L25` (`wrspe`); `get_pz.cc` confirmed same structure with `pjy`/`pzraw`. No `#include`, no `main()` — script fragment confirmed.

---

## `get_XAeraw.cc` / `get_XApz.cc` / `get_XAecln.cc` — X-Array Batch SPE Extractor Scripts

_Source: `gebsort/get_XAeraw.cc` (27 L), `gebsort/get_XApz.cc` (27 L), `gebsort/get_XAecln.cc` (31 L). Code-read 2026-04-27._

X-Array analogs of `get_eraw.cc` / `get_pz.cc` — GEBSort inline script fragments (no `#include`, no `main()`) used to batch-extract spectra from a running GEBSort session. Same pattern: `for` loop over detectors, project histogram to 1D via `pjx`/`pjy`, write with `wrspe`.

- **`get_XAeraw.cc`:** loops detector indices 0–110 (111 iterations), calls `pjx("XAEhiRawRaw","p",i,i)` and writes `ehi<NNN>.spe`. Produces raw energy spectra for all X-Array clover detectors. ✅ verified 2026-04-27 — `get_XAeraw.cc:L23` (`pjx("XAEhiRawRaw","p",i,i)`), `L25` (`wrspe`)
- **`get_XApz.cc`:** loops detector indices 1–49 (49 iterations), calls `pjy("XApzraw","p",i,i)` and writes `pz<NNN>.spe`. Produces pole-zero correction spectra. ✅ verified 2026-04-27 — `get_XApz.cc:L21` (`for (i=1;i<50;i++)`), `L23` (`pjy("XApzraw","p",i,i)`)
- **`get_XAecln.cc`:** loops detector indices 0–49 (50 iterations), calls `pjx("XAEhiCln","p",i,i)` and writes `ehi<NNN>.spe`; also projects the full range 1–110 into `ref.spe` via `pjx("XAEhiCln","p",1,110)`. Produces clean (Compton-suppressed) energy spectra plus a summed reference. ✅ verified 2026-04-27 — `get_XAecln.cc:L23` (`pjx("XAEhiCln","p",i,i)`), `L27` (`pjx("XAEhiCln","p",1,110)`), `L28` (`wrspe("p","ref.spe")`)

**Key differences from DGS versions:**
- Histogram names use `XA` prefix (`XAEhiRawRaw`, `XApzraw`, `XAEhiCln`) — defined in `bin_XA.c`
- `get_XAecln.cc` additionally writes a summed `ref.spe` (projection over 1–110 range) — not present in DGS equivalents
- `get_XApz.cc` starts at i=1 (skips index 0), writing `pz001.spe`–`pz049.spe`
- `get_XAeraw.cc` loops 0–110 (wider than the 40-detector `NXA` limit) — conservative upper bound

---

## `mk_gsutil.cc` — GSUtil Compile Helper (6 lines)

_Source: `gebsort/mk_gsutil.cc`. Code-read 2026-04-27._

A ROOT CINT macro that compiles `GSUtil.cc` and immediately exits ROOT. Used to pre-build the shared library (`GSUtil_cc.so`) without entering a full GEBSort session.

```
gROOT->ProcessLine(".L GSUtil.cc++");
gROOT->ProcessLine(".q");
```

**Usage:** `root -l mk_gsutil.cc` — produces `GSUtil_cc.so` which can then be loaded via `.L GSUtil_cc.so` in subsequent ROOT/GEBSort sessions without recompilation. ✅ verified 2026-04-27 — `mk_gsutil.cc:L2` (`printf("compile\n")`), `L3-4` (ProcessLine calls)

---

## `curEPICS` — Symbolic Link

_Source: `gebsort/curEPICS` → `/global/base/base-3.14.12.8/` (broken on spark-ca9f)._

A symbolic link to the EPICS base installation used when building GEBSort on the ANL GS network (`/global` NFS). Broken on spark-ca9f where `/global` is not mounted. Present only for the `Makefile` build environment; no runtime significance.

---
*Created: 2026-04-07. Source: `DGS_tools_pack/gebsort/` README + source files.*
*Utility programs section added: 2026-04-27.*
*time_stamp.c, temp_ge.c, trig_fun.c, tlutil2.c, 2d_fun.c documented: 2026-04-27.*
*GEBCrop, GEBFilter, GEBSplit, GEBHeader, utils.c, spe_fun.c, tlutil.c documented: 2026-04-27.*
*writeTrack_addtrack/repeat, G4toMode2, get_dead_layer_corrections2, fwhm511/fwhm_511, get_eraw/pz, curEPICS documented: 2026-04-27.*
*get_XAeraw/pz/ecln, mk_gsutil.cc documented: 2026-04-27.*
