# dgs_analysis — DGS Analysis Code Collection

## What It Is

A collection of **experiment-independent analysis code** for processing DGS raw data. Structured as:
- `armory/` — reusable, experiment-independent tools (event builders, decoders, etc.)
- `working/` — experiment-specific files (calibration files, etc.)

---

## Repository

```sh
git clone --recursive https://gitlab.phy.anl.gov/dgs-tools-pack/dgs_analysis.git
# or via SSH:
git clone --recursive git@gitlab.phy.anl.gov:dgs-tools-pack/dgs_analysis.git
```

---

## armory/ — Reusable Tools

### fastEventConstructor

**Purpose:** Reads raw DGS binary files and produces CERN ROOT TTrees (event-built data).

**Data format:** Raw DGS binary files contain **GEB (GRETINA Event Builder) format** records — each record has a `GEBHeader` + payload. The `Hit` class stores these, decoding the payload into a `DIG` struct.

**Important fields per hit:**
- `id` — `board_id * 10 + channel_id`
- `pre_rise_energy` — energy before rise
- `post_rise_energy` — energy after rise
- `timestamp` — timing

**EventBuilder Variants:**

| Variant | Key Feature | Use Case |
|---------|-------------|----------|
| `EventBuilder` | Original; priority queue, parallel file scan, no intermediate files | General use |
| `EventBuilder_S` | Pre-scan pass for timestamp ranges; single-threaded k-way merge | When timestamp bounds needed |
| `EventBuilder_Q` | Async double-buffered k-way merge; batch pre-decoding; pipelined ROOT writers | Fast I/O, large datasets |
| `EventBuilder_PQ` | Parallel k-way merge with sector partitioning; N parallel threads | Maximum throughput, multi-core |

**EventBuilder_Q optimizations:**
- Batch pre-decoding (not per-hit)
- ReadPool: async pre-fills next batch while current is consumed (double buffering)
- Lightweight merge heap: `{timestamp, groupIndex}` only (16 bytes vs full DIG struct)
- 4 pipelined ROOT writers (round-robin)

**EventBuilder_PQ optimizations (extends Q):**
- Pre-scan phase: reads only GEB headers to get per-file timestamp bounds + seek index
- Sector partitioning: divides time span into N sectors with ghost regions at boundaries
- N parallel merge threads × M pipelined writers per thread
- Per-sector double-buffered ReadPool

```sh
# Build
make                    # all variants
make EventBuilder_Q     # single variant
make EventBuilder_PQ

# Usage
./EventBuilder_Q [outfile] [timeWindow] [useTrigTS] [saveTrace] [nWorkers] [file1] [file2] ...
```

### parquet_pysort — Python/C++ Parquet Pipeline

**Purpose:** Decodes DGS binary data to Parquet format, sorts events, builds event structure.

Key files:
| File | Role |
|------|------|
| `decode.py` | Main decoder |
| `dgs_decode_lib.cpp` | C++ decode library |
| `geb_format.py` | GEB format definitions |
| `event_builder.py` | Event building logic |
| `make_filemap_dgs.py` | Build file map for dataset |
| `read_parquet.py` | Read/inspect Parquet output |
| `dgs_gain.cal` | Gain calibration |
| `dgs_pz.cal` | Pole-zero calibration |
| `angtheta.dat` | Angular/theta data |
| `map.dat` | Detector mapping |

### gray_apps

Additional analysis applications (contents not fully explored).

---

## working/ — Experiment-Specific Scripts & Calibration

Holds experiment-specific scripts and calibration files. All paths driven by `expInfo.sh` from `~/ANLDAQ/tcpReceiver/expInfo.sh`.

*Updated 2026-04-05 from git pull (commits up to 5126a11)*

### RunParquet

Runs the full parquet_pysort pipeline for a single run:
```
make_filemap_dgs.py → decode.py → event_builder.py
```
- Stage 1 skipped if filemap exists and is newer than raw files
- Also called automatically by `stop_run.sh` after each run

```bash
./working/RunParquet [--decode-only] <expInfo.sh> <run_number> [TIMEWIN] [THREADS]
```

| Arg | Default | Description |
|-----|---------|-------------|
| `--decode-only` | off | Stages 1+2 only, skip event builder (for pole-zero prep) |
| `TIMEWIN` | 1000 | Coincidence window in ticks |
| `THREADS` | 78 | Threads for decode + event builder |

**Output:**
- `$expFolder/Parquet/decode/$expName_NNN_dgs.parquet` — timestamp-sorted hits
- `$expFolder/Parquet/events/$expName_NNN_events.parquet` — coincidence events

**Parquet schema:**

| File | Key Columns |
|------|-------------|
| `_dgs.parquet` | `tid`, `timestamp`, `sum1`, `sum2`, `e_raw`, `e_cal`, `e_dc`, `CSflag` |
| `_events.parquet` | `event_id`, `gs_mult`, `gs_hitid`, `gs_ts`, `gs_cryid`, `gs_eraw`, `gs_ecal`, `gs_edc`, `gs_flag` |

### parquetCLI

Interactive REPL for exploring any `_dgs.parquet` or `_events.parquet` file. Columns discovered dynamically at load time.

```bash
./working/parquetCLI <file.parquet>
```

### gain_from_parquet.py

Extracts gain calibration from decoded parquet data.

### pz_from_parquet.py

Extracts pole-zero calibration from decoded parquet data. Use with `--decode-only` RunParquet:
```bash
python working/pz_from_parquet.py $expFolder/Parquet/decode/exp2008_003_dgs.parquet --output working/dgs_pz.cal
```

### ProcessRUN

Higher-level run processing wrapper.

---

## Data Format: GEB (GRETINA Event Builder)

Raw DGS binary files use the GEB format:
- **GEBHeader** — contains type, length, timestamp
- **Payload** — digitizer data (DIG struct): board_id, channel_id, energies, timestamp, waveform trace (optional)

`TRASH_DATA` markers in files are skipped via `skipTrash()` cursor logic.

### GEB Type Codes
_Source: `DGS_SVN/dgs/gtReceiver/dgsReceiver/dgsReceiver.cpp` — list from Torben (GRETINA), as of 2021-12-07_

| Code | GEB Type | Notes |
|------|----------|-------|
| 1 | `GEB_TYPE_DECOMP` | Decomposed GRETINA |
| 2 | `GEB_TYPE_RAW` | Raw GRETINA |
| 3 | `GEB_TYPE_TRACK` | Tracked |
| 4 | `GEB_TYPE_BGS` | BGS detector |
| 5 | `GEB_TYPE_S800_RAW` | S800 spectrometer raw |
| 6 | `GEB_TYPE_NSCLnonevent` | NSCL non-event |
| 7 | `GEB_TYPE_GT_SCALER` | GRETINA scaler |
| 8 | `GEB_TYPE_GT_MOD29` | GRETINA Module 29 |
| 9 | `GEB_TYPE_S800PHYSDATA` | S800 physics data |
| 10 | `GEB_TYPE_NSCLNONEVTS` | NSCL non-events |
| 11 | `GEB_TYPE_G4SIM` | Geant4 simulation |
| 12 | `GEB_TYPE_CHICO` | CHICO detector |
| **14** | **`GEB_TYPE_DGS`** | **DGS digitizer data** |
| **15** | **`GEB_TYPE_DGSTRIG`** | **DGS trigger data** |
| 16 | `GEB_TYPE_DFMA` | DFMA (Digital Fast Multiplicity Array) |
| 17 | `GEB_TYPE_PHOSWICH` | Phoswich detector |
| 18 | `GEB_TYPE_PHOSWICHAUX` | Phoswich aux |
| 19 | `GEB_TYPE_GODDESS` | GODDESS array |
| 20 | `GEB_TYPE_LABR` | LaBr3 detector |
| 21 | `GEB_TYPE_LENDA` | LENDA neutron detector |
| 22 | `GEB_TYPE_GODDESSAUX` | GODDESS aux |
| 23 | `GEB_TYPE_XA` | X-Array |
| 24 | `MAX_GEB_TYPE` | |

Note: type 13 is absent (unassigned). DGS uses types 14 (digitizer hits) and 15 (trigger data).

---

## Connections to Other Subsystems

- **ANLDAQ** — `tcpReceiverMT` writes the raw binary files that `dgs_analysis` reads
- **ioc/** — the IOC munch file determines what data fields are available in the payload
- **fpga/** — DIG firmware determines signal processing (energy, timestamp, trace format)

---

## Notes

- Save traces (`saveTrace=true`) allocates ~153 KB per event — keep disabled for large runs unless needed
- Parquet pipeline is an alternative to ROOT for Python-native workflows
- `angtheta.dat` and `map.dat` are geometry files mapping detector IDs to physical positions
