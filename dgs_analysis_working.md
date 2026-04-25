# dgs_analysis — working/ Scripts & Calibration Tools

Stability: C2 - Active / semi-stable

_Split from `dgs_analysis.md` on 2026-04-25. Source: `DGS_tools_pack/dgs_analysis/working/`_

**See also:** [`dgs_analysis.md`](dgs_analysis.md) — armory/ reusable tools (EventBuilders, parquet_pysort, misc.h, gray_apps)

---

## Table of Contents

- [Overview](#overview)
- [RunParquet (legacy)](#runparquet-legacy--superseded-by-processrun)
- [parquetCLI](#parquetcli)
- [gain_from_parquet.py](#gain_from_parquetpy)
- [pz_from_parquet.py](#pz_from_parquetpy)
- [pz_from_evtparquet.py](#pz_from_evtparquetpy)
- [DownloadRaw.sh](#downloadrawsh)
- [ProcessRUN (primary pipeline)](#processrun-primary-pipeline--apr-2026)
- [BenchmarkTAC2_021.sh](#benchmarktac2_021sh)
- [Cross-References](#cross-references)

---

## Overview

`working/` holds experiment-specific scripts and calibration tools. All paths driven by `expInfo.sh` from `~/ANLDAQ/tcpReceiver/expInfo.sh`.

**Primary workflow (Apr 2026):** `ProcessRUN` → `EventBuilder_X` (Parquet output). `RunParquet` is legacy.

*Updated 2026-04-07 from git pull (commits up to 0100567)*

---

## RunParquet *(legacy — superseded by ProcessRUN)*

> **As of Apr 2026, `ProcessRUN` (C++ EventBuilder) is the primary pipeline.** `RunParquet` drove the Python `parquet_pysort` pipeline (`make_filemap_dgs.py → decode.py → event_builder.py`). It still exists but `ProcessRUN` is faster and preferred.

```bash
./working/RunParquet [--decode-only] <expInfo.sh> <run_number> [TIMEWIN] [THREADS]
```

| Arg | Default | Description |
|-----|---------|-------------|
| `--decode-only` | off | Stages 1+2 only, skip event builder (for pole-zero prep) |
| `TIMEWIN` | 1000 | Coincidence window in ticks ✅ verified 2026-04-09 — `RunParquet:L67` (`TIMEWIN="${2:-1000}"`) — note: script header comment still says 800 (stale) |
| `THREADS` | 78 | Threads for decode + event builder |

**Output:**
- `$expFolder/Parquet/decode/$expName_NNN_dgs.parquet` — timestamp-sorted hits
- `$expFolder/Parquet/events/$expName_NNN_events.parquet` — coincidence events

**Parquet schema:**

| File | Column | Type | Description |
|------|--------|------|-------------|
| `_dgs.parquet` | `tid` | int64 | Crystal ID |
| | `header_ts` | int64 | GEB header timestamp (10 ns/tick) |
| | `trigger_ts` | int64 | DGS payload trigger timestamp (0 if unavailable) |
| | `sum1` | int64 | Trapezoid baseline sum (ADC counts) |
| | `sum2` | int64 | Trapezoid peak sum (ADC counts) |
| | `e_raw` | float64 | Raw energy before gain/offset cal (ADC channels) |
| | `e_cal` | float64 | Calibrated energy: `e_raw × gain + offset` (keV) |
| | `e_dc` | float64 | Doppler-corrected energy (keV) |
| | `CSflag` | int8 | Compton suppression: 1=BGO-vetoed, 0=clean |
| | `pileup_count` | int32 | Pileup count from firmware (typically 0 due to known bit-shift bug) |
| `_events.parquet` | `event_id` | int64 | Global event index (0-based) |
| | `gs_mult` | int64 | Gamma-ray multiplicity (hits per event) |
| | `gs_hitid` | int64 | Hit index within event (0-based) |
| | `gs_ts` | int64 | Hit timestamp (DAQ ticks) |
| | `gs_cryid` | int64 | Crystal ID |
| | `gs_eraw` | float64 | Raw energy (ADC channels) |
| | `gs_ecal` | float64 | Calibrated energy (keV) |
| | `gs_edc` | float64 | Doppler-corrected energy (keV) |
| | `gs_flag` | int8 | Compton suppression flag |

_Source: `parquet_pysort/README.md` ✅ verified 2026-04-09_

---

## parquetCLI

Interactive REPL for exploring `_dgs.parquet` (hit-level) or `_events.parquet` (event-level) files. Columns discovered dynamically at load time.

```bash
./working/parquetCLI <file.parquet>
./working/parquetCLI <file.parquet> --script working/script.py
```

**`pq_api.py`** — IDE type-stub only; never import directly. Provides typed signatures for all REPL functions.

### Commands

| Command | Description |
|---------|-------------|
| `ls` | List columns, types, descriptions |
| `ls hist` | List stored histograms |
| `ls cal` | List loaded calibration objects |
| `info` | File metadata (rows, size, schema) |
| `lsID` | List all crystal IDs with hit counts |
| `print <col> [N] [S] [G(<gate>)]` | Print N rows starting at row S, optionally gated |
| `loadParquet <file> [file2 ...]` | Load or chain parquet files |
| `unloadParquet` | Free loaded file(s) |
| `loadCal <file.cal>` | Load gain/offset cal → callable `cal<gsid>` objects |
| `loadScript <script>` | Run a script: `.py` = full Python; otherwise custom CLI syntax (see Scripting below) |
| `rm <name>` | Delete stored object |
| `newWindow` | Next plot in a new window |
| `set <VAR> <value>` | Set a script variable; use `{VAR}` in subsequent commands (scripting) |
| `saveParquet <out.parquet> G(<gate>) [CS] [CAL(col, file.cal)]` | **Filter and save**: apply gate (required), optional CS flag, optional energy calibration → write surviving rows/events to new Parquet ✅ verified 2026-04-20 — `dgs_analysis/working/parquetCLI:L743-833` |

### Plotting

| Command | Description |
|---------|-------------|
| `plot1D [CS] <col\|expr> [bw [xmin xmax]] [G(<gate>)]` | 1D histogram |
| `plot2D [col1 col2] [bw]` | 2D histogram (default: sum1 vs sum2) |
| `plotGG [CS] <col> [bw [xmin xmax]] [G(<gate>)]` | Gamma-gamma coincidence 2D (event-level only) — builds all pairwise hit combinations per event into a symmetric 2D matrix ✅ verified 2026-04-18 — `dgs_analysis/working/README.md:L76-78` |
| `plot <name> [name ...]` | Display stored histogram(s) |
| `overlay <name> [name ...]` | Overlay multiple 1D histograms |

`CS` flag = Compton suppression (drops hits with matching negative-ID veto hit in same event).

Store a histogram with `>`:
```
dgs> plot1D e_cal 1 0 4000 > spec
dgs> plot1D CS e_cal 1 0 4000 G(tid==6) > spec_cs
```

### Gate syntax

Gates filter inline via `G(<expr>)` — combine with `&&` / `||`:
```
G(tid==6)                  single crystal
G(tid==6&&CSflag==0)       AND
G(e_cal>=500&&e_cal<1500)  energy window
```

### Formula columns

Any Python expression using column names, numpy functions, and loaded calibrations:
```
dgs> plot1D "sum2-sum1" 1 0 5000
dgs> plot1D "e_cal*1.05" 1 0 4000 G(tid==6)
dgs> plot1D "cal88(e_raw)" 1 0 4000    # apply loaded calibration
```

Available: `sqrt`, `abs`, `log`, `log10`, `exp`, `sin`, `cos`, `np.*`

### Fitting and calibration

```
dgs> fit spec 5                 # find+fit peaks above 5% of max
dgs> cal spec eu152             # auto-calibrate against 152Eu source
dgs> cal spec 10 207Bi Mx      # 10% threshold, anchor on highest-energy peak
```

Available sources: `am241`, `am243`, `ba133`, `co56`, `eu152`, `na24`, `se75`, `ta182`, `y88`, `207Bi`, etc.

```
dgs> loadCal working/dgs_gain.cal    # load per-crystal gain/offset
dgs> cal88 > dgs_gain.cal 88         # write cal to file (new)
dgs> cal89 >> dgs_gain.cal 89        # append to existing file
```

### Histogram arithmetic

```
dgs> h1 + h2 > h3
dgs> h1 - h2 > hdiff
dgs> h1 * h2 > hprod
dgs> h1 / h2 > hratio
```

### Saving / exporting

```
dgs> spec > spec.png          # save figure (png, pdf, svg, ...)
dgs> saveParquet out.parquet G(tid==6) CS CAL(e_raw, dgs_gain.cal)
# Note: G(<gate>) is required for saveParquet. CS applies CSflag==0 filter.
# CAL(col, file.cal) applies per-crystal gain/offset to <col>, adds e_cal column.
```

### Scripting

**Custom syntax** (`.txt`): supports `set`, `for`/`endfor`, `if`/`endif`, `break`, `continue`:
```
set TIDS 6 7 8 9
for TID in {TIDS}
    plot1D e_cal 1 0 4000 G(tid=={TID}) > spec_{TID}
endfor
```

**Python scripts** (`.py`): full Python with access to the CLI session. Example `working/script.py` demonstrates a calibration-then-sort workflow:

```python
# Pass 1 (isCal=True): load run 001, fit 207Bi peaks per crystal, write dgs.cal
cmd("loadParquet Parquet/exp2008_001_0.parquet")
tids = lsID()
for tid in tids:
    cmd(f"plot1D e_raw/350 1 1000 6000 G(detID=={tid}) > h{tid}")
    cmd(f"cal h{tid} 207Bi > cal{tid}")
    cmd(f"cal{tid} > dgs.cal {tid}")   # first write; >> for subsequent

# Pass 2 (isCal=False): load run 005, apply cal, plot crystal 13
cmd("loadParquet Parquet/exp2008_005_0.parquet")
cmd("loadCal dgs.cal")
for tid in tids:
    cmd(f"plot1D cal{tid}(e_raw/350) 0.1 100 1000 G(detID=={tid}) > h{tid}")
cmd("plot h13")
```

Note `e_raw/350` — divides raw sum by trapezoid gap (350 samples) to get per-sample energy before applying the gain cal function.

**Pipe via stdin:** `./working/parquetCLI data.parquet < commands.txt`

### Scripting — `if`/`endif` condition syntax
`if <cond>` / `endif` conditions in custom scripts support:
- `exists <name>` — true if a stored object named `<name>` exists
- `not exists <name>`
- `{VAR} == value`, `{VAR} != value`, `{VAR} > value`, etc.
✅ verified 2026-04-18 — `dgs_analysis/working/README.md:L214-217`

### Interactive features
- Tab completion (commands, columns, filenames)
- **Argument hints** — press Tab at the start of each argument to see what's expected ✅ verified 2026-04-18 — `dgs_analysis/working/README.md:L229`
- Persistent history (`parquetCLI.history`)
- Rectangle zoom: left-drag to zoom, right-click to reset

_Source: `dgs_analysis/working/README.md` commit b609604 (2026-04-07)_

---

## gain_from_parquet.py

Extracts **energy gain/offset calibration** from a `_dgs.parquet` file. For each crystal (`tid`), builds a 1D `e_raw` spectrum, auto-fits gamma peaks using the GrayCAL `AutoFitter`, matches to a known calibration source, and performs a linear fit: `E_keV = gain * e_raw + offset`. Writes `dgs_gain.cal`.

```bash
python working/gain_from_parquet.py <dgs.parquet> [options]
  --output FILE      Output cal file          (default: dgs_gain.cal)
  --source FILE      JSON isotope source      (default: Euautocal.json = 152Eu)
  --e-bins N         e_raw histogram bins     (default: 4096)
  --e-max N          Upper bound for e_raw    (default: 99.5th percentile)
  --threshold F      Peak sigma threshold     (default: 3.0)
  --tid N [N ...]    Process only these crystal IDs (default: all)
  --quiet            Suppress per-crystal output
```

**Pipeline per crystal:**
1. Histogram `e_raw` (skips crystals with <500 hits) ✅ verified 2026-04-08 — `gain_from_parquet.py:L182`
2. `AutoFitter` finds + fits peaks (SNIP background, 20 iterations, min_fwhm=3 channels) ✅ verified 2026-04-08 — `gain_from_parquet.py:L91-94`
3. `CalibrationPoints.add_points_from_fit()` extracts peak centroids
4. `cal_pts.match_w_source()` matches to known isotope gamma lines
5. `cal_pts.energy_calibration()` → linear fit → `gain`, `offset`
6. Skips crystals with <2 matched peaks or failed fits

**Output format** (`dgs_gain.cal`):
```
# gsid  gain  offset
  1  0.123456  -12.3456
  2  0.124001   -9.8765
...
```

Uses `gray_apps/data/isotopes/sou-files/Euautocal.json` (152Eu) as default source.

---

## pz_from_parquet.py

Extracts **pole-zero calibration** from a decoded hit-level `_dgs.parquet` file. Reads `sum1` (S1 baseline trapezoid integral), `sum2` (S2 signal trapezoid integral), and `tid` (crystal ID). For each crystal, builds a 2D S1/S2 histogram then calls `pole_zero_fitter.estimate_pz_from_histogram()` to find the PZ coefficient. Writes `dgs_pz.cal` in GEBSort format.

Use with `--decode-only` RunParquet output (hit-level, not event-level):
```bash
python working/pz_from_parquet.py $expFolder/Parquet/decode/exp2008_003_dgs.parquet \
    --output working/dgs_pz.cal \
    --method chi2   # chi2 | peakmatch | pca | ridge
    --pz-min 0.88 --pz-max 0.99 --pz-step 0.0005
    --s1-bins 512 --s2-bins 512
```

**Pipeline per crystal:**
1. Read `sum1`, `sum2`, `tid` columns from parquet
2. Build 2D numpy histogram (S1 × S2, 512×512 default bins)
3. `estimate_pz_from_histogram()` → `DetResult` dataclass with `.pz` coefficient
4. `write_pz_cal(path, pz_map)` → `dgs_pz.cal` (format: `det  pz`, one line per crystal)

For event-level parquet (list columns), use `pz_from_evtparquet.py` instead.

---

## pz_from_evtparquet.py *(added Apr 2026)*

Extract pole-zero constants from **event-level** `.parquet` files (where `detID`, `sum1`, `sum2` are `list<>` columns rather than flat columns). Flattens lists to rows then runs the same 2D-histogram + PZ fitting pipeline per crystal. ✅ verified 2026-04-17 — `pz_from_evtparquet.py:L1-30` (docstring + argparse confirm interface)

```bash
python working/pz_from_evtparquet.py <file.parquet> [options]
  --output FILE      Output cal file (default: dgs_pz.cal)
  --method METHOD    chi2 | peakmatch | pca | ridge  (default: chi2)
  --pz-min/max/step  PZ scan bounds (default: 0.88–0.99, step 0.0005)
  --s1-bins/s2-bins  Histogram bins (default: 512 each)
  --detID N ...      Process only these crystal IDs (default: all)
  --quiet            Suppress per-crystal progress
```

---

## DownloadRaw.sh *(added Apr 2026)*

Copies raw GEB run data from NFS to local `expFolder/data/` via rsync. ✅ verified 2026-04-17 — `DownloadRaw.sh:L1-30` (header comments + nfsFolder check at L59 confirmed)

```bash
./working/DownloadRaw.sh [--dry-run] <expInfo.sh> <run_number> [run_number ...]
# e.g.: ./working/DownloadRaw.sh expInfo.sh 3 5 7
```

Requires `nfsFolder` defined in `expInfo.sh` (root of NFS data mount). Supports `--dry-run` to preview without copying.

---

## ProcessRUN *(primary pipeline — Apr 2026)*

Higher-level run processing wrapper. Drives **EventBuilder_X** (Parquet output, no ROOT dependency). Pole-zero and ROOT analysis steps are currently commented out in the script.

```bash
./working/ProcessRUN [expInfo.sh] <run_number> [BUILD] [ANALYSIS]
  expInfo.sh : experiment config (default: expInfo.sh in script dir)
  run_number : integer run number (e.g. 3 → 003)
  BUILD      : 1=build if stale (default), 0=skip, -1=force rebuild
  ANALYSIS   : 1=run ROOT analyzer (default), 0=skip
```

Sources `expInfo.sh` (from arg or script dir) for `expName`, `expFolder`, `dataFolder`. **Recommended setup:** create a symlink so the default path works without passing the arg every time:
```bash
ln -s ~/ANLDAQ/tcpReceiver/expInfo.sh working/expInfo.sh
```

**Key settings (hardcoded in script):**
- `timeWin=1000` ticks coincidence window ✅ verified 2026-04-09 — `ProcessRUN:L39`
- `useTrigTS=0`, `saveTrace=0`, `nMerge=40` threads ✅ verified 2026-04-09 — `ProcessRUN:L40-42`
- Output: `${expFolder}/Parquet/${expName}_${RUN}_0.parquet` (filename = `_${useTrigTS}.parquet`; `0` because useTrigTS=0) ✅ verified 2026-04-09 — `ProcessRUN:L66`
- Input: `${expFolder}/data/${expName}_${RUN}/${expName}_${RUN}*` (excludes `.geb` and `.txt`) ✅ verified 2026-04-09 — `ProcessRUN:L65`

**EventBuilder_X** — the builder used by ProcessRUN. Identical parallel k-way merge engine as EventBuilder_PQ, but writes **Apache Parquet** output instead of ROOT (no ROOT dependency required). Output schema (nested, one row = one coincidence event):

| Column | Type | Description |
|--------|------|-------------|
| `evID` | uint32 | Event index |
| `NumHits` | uint16 | Hits in this event |
| `id` | list\<uint16\> | Raw digitizer id (boardID×100 + channel) |
| `detID` | list\<int16\> | Mapped detector id (999=trigger, negative=BGO) |
| `sum1` | list\<uint32\> | Pre-rise trapezoidal sum S1 (24-bit) |
| `sum2` | list\<uint32\> | Post-rise trapezoidal sum S2 (24-bit) |
| `e_raw` | list\<float32\> | PZ-corrected energy: S2 − pz×S1 |
| `eventTS` | list\<uint64\> | EVENT_TIMESTAMP in ticks |
| `trigTS` | list\<uint64\> | TS_OF_TRIGGER_FULL in ticks |
| `trace*` | list (optional) | Waveform + GSL fit when saveTrace=1 |

_Source: `fastEventContructor/EventBuilder_X.cpp` header comment. Verified 2026-04-08._

---

## BenchmarkTAC2_021.sh

Performance comparison script for the three event-building approaches:
```bash
./working/BenchmarkTAC2_021.sh [THREADS]
```
- Runs `EventBuilder_Q`, `EventBuilder_PQ`, and `parquet_pysort` against the `TAC2_021` dataset
- Reports wall time + output size for each; prints Q/PQS, PQ/PQS, and Q/PQ speedup ratios
- Default: 8 threads; uses `TIMEWIN=1000` (ticks) for fair apples-to-apples comparison
- Requires ROOT environment (`/opt/root/bin/thisroot.sh`) and Python venv (`.venv/`)
- Output log: `working/benchmark_output/benchmark_<timestamp>.log`

---

## Cross-References

- [`dgs_analysis.md`](dgs_analysis.md) — armory/ reusable tools: EventBuilders (S/Q/PQ/X/XR), parquet_pysort, misc.h, gray_apps summary, data format, benchmark results
- [`dgs_analysis_grayapps.md`](dgs_analysis_grayapps.md) — gray_apps full reference: GrayCAL, GrayMAN, grayfit
- [`dgs_analysis_root_scripts.md`](dgs_analysis_root_scripts.md) — ROOT analyzer scripts: analyzer_*.cpp, Cali_e, checkTACFile, findMapping/findGS
- [`pole_zero.md`](pole_zero.md) — Pole-zero correction theory; `pz_from_parquet.py` workflow detail
- [`run_procedures.md`](run_procedures.md) — Typical DGS run procedures; ANLDAQ + GEBSort workflow
- [`expMemory_2008_Chiara.md`](expMemory_2008_Chiara.md) — Active experiment data locations; `expInfo.sh` setup for exp2008_Chiara
