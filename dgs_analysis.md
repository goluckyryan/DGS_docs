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

## working/ — Experiment-Specific Files

Holds calibration files, run-specific configs, etc. Not experiment-independent — changes per run/experiment.

---

## Data Format: GEB (GRETINA Event Builder)

Raw DGS binary files use the GEB format:
- **GEBHeader** — contains type, length, timestamp
- **Payload** — digitizer data (DIG struct): board_id, channel_id, energies, timestamp, waveform trace (optional)

`TRASH_DATA` markers in files are skipped via `skipTrash()` cursor logic.

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
