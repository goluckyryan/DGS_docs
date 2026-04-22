# DGS Troubleshooting Guide

**Sources:** [wiki: Some Problems and Their Solutions](https://wiki.anl.gov/gsdaq/Some_problems_and_their_solutions) + operational experience

---

## General Approach: Things Are Broken and You Don't Know Why

### Step 1 — Check IOC status

Open **DGS Commander** main screen → click **VME Status** → brings up IOC Summary page.

The IOC Summary shows whether all VME crate controllers are responding. A non-talking IOC will be flagged (e.g. "IOC 9 not talking").

### Step 2 — Check the broken IOC's terminal

Open the terminal window for the miscreant IOC from DGS Commander.

---

## Problem: IOC Not Talking — Ethernet Issues

**Symptom:** IOC never loads; boot message shows:
```
0x3ff73940 (tBoot): miiPhyInit check cable connection
```
*(The address `0x3ff73940` is runtime-specific; `miiPhyInit check cable connection` is the canonical VxWorks MII Ethernet PHY init failure message.)* ✅ verified 2026-04-17 — DGS_SVN VxWorks network driver source (miiPhyInit() in mottsecEnd.c/miiLib.c)

**Cause:** Broken or disconnected Ethernet cable between IOC and network switch.

**Solution:**
1. Go to the shack, trace the Ethernet cable from the IOC back to the router
2. Verify the green "link" light is on for that IOC's port
3. Unplug and re-seat both ends of the cable
4. If no link after reseating → try one power cycle of the IOC
5. If still no link → test the cable by plugging both ends into unused router ports; both green link lights must illuminate — if not, **replace the cable**

---

## Problem: Router Not Locking to Digitizer Data

**Symptom:** Router counter shows a channel dropping in/out of lock.

**Common causes:**
- SERDES accidentally disabled
- SYNC bit still set in `LRUCtl02` PV of the Router (the Link L SYNC control) — data not actually being sent to Master
- Cable length mismatch (jitter budget exceeded)

**Check:** Use Router channel FIFOs to verify real discriminator data (bits 9:0) is arriving. ✅ verified 2026-04-17 — `260E_trigger_scheme.md:L38,42` (`DELAYED_DATA[9:0]`: bits[9:5] = Ge ch discriminators, bits[4:0] = BGO sum discriminators; as recovered by `chan_in` DCBAL_IN sub-block)

*(Source: [wiki: Triggers and Digitizers](https://wiki.anl.gov/gsdaq/Triggers_and_digitizers) — see `wiki_gsdaq.md`)*

---

## Problem: Master Trigger Not Receiving Router Data

**Symptom:** Master trigger shows no multiplicity even though digitizers and routers appear healthy.

**Common cause:** Forgot to clear the SYNC bit (`LRUCtl02=0`) in the Router. Data streams SYNC patterns to Master instead of real event data.

**Fix:** Clear SYNC bit — `caput VMExx:RTRy:LRUCtl02 0` for each router. Stage 5 of the SERDES link-up (`link_sys.py:L651`) does this automatically. ✅ verified 2026-04-15 — `link_sys.py:L651` (`LRUCtl02=0` cleared per router in Stage 5)

*(Source: [wiki: Triggers and Digitizers](https://wiki.anl.gov/gsdaq/Triggers_and_digitizers))*

---

## Problem: IOC Boot Fails — VxWorks

**See:** `vxworks_migration.md` for known build/boot issues from Solaris→Ubuntu migration.

Key issues documented:
- Missing `makeSymTbl` tool → replaced with `munch` process
- Missing `wtxtcl` → workaround via Docker/ISE environment
- `ld_LINKER_VERSION` mismatch → fixed by cross-compiler flags

---

## Problem: Data Rate / FIFO Issues

**Symptom:** `TotalBuffers_Lost` climbing, data gaps in output files.

**Check (actual EPICS PV names — ✅ verified 2026-04-17 — `ioc/db/daqCrate.template`):**
- `DAQC$(CRATE)_OL_TotalBufsLost` — total buffers lost in outLoop (was mislabeled `TotalBuffers_Lost` in earlier notes)
- `DAQC$(CRATE)_CV_OutLoop{0-6}` — data lost per-board (7 boards per crate)
- `DAQC$(CRATE)_OL_DataRate{0-6}` — kBytes/sec processed per board in outLoop
- `DAQC$(CRATE)_CV_SendRate` — KB/sec total output rate from MiniSender
- `DAQC$(CRATE)_OL_NumFreeBuffers` / `DAQC$(CRATE)_OL_NumSendBuffers` — buffer pool status
- Throttle line from digitizer to Router (FIFO half-full flag via TTCL LVDS2 line)

> ⚠️ `DigitizerFull`, `DigitizerAlmostFull`, `SendDataRate` are **not real PV names** — use the DAQC PVs above.

**Cause:** Trigger rate too high, FIFO filling faster than TCP readout can drain it.

**Fix:** Raise multiplicity threshold or enable software veto to reduce trigger rate.

---

## Problem: Timestamp Sync Errors

**Symptom:** `SE` (Sync Error) flag set in digitizer event headers; `DLYD_TDC_IN_NIM_IN2_RBV` shows unexpected values.

**Check:**
- SERDES lock status on all links
- Clock jitter budget — total accumulated jitter must be < 250 ps ✅ verified 2026-04-17 — `ttcl.md:L72` (from `20160418 trig command link.pdf`: "Recommendation: total accumulated jitter < 250 ps at the far end")
- Link L clock recovery on Routers

*(Source: `ttcl.md`, `DIG_firmware_expert.md`)*

---

## Problem: Missing BGO Events on Specific Detectors

**Symptom:** BGO channels for one or more Ge detectors show zero or very low rate even when beam is present.

**Possible cause:** Ge preamp reset blanking (`CHANNEL_KILLED`) can gate paired BGO channels — but only when `reg_external_disc_mode = "101"` AND `FRONT_BUS_LEFT = TRUE` (GeCenter/GeLeft pairing). In that mode, `external_disc_flag(i) <= not(CHANNEL_KILLED(i+5))` suppresses BGO channels 0–4 while the paired Ge channel (5–9) is in preamp-reset kill state. ✅ verified 2026-04-12 — `Digitizer.vhd:L1126` (mode "101", FRONT_BUS_LEFT=TRUE branch)

**Check:**
- Verify `MOD###_DIG_EXTERNAL_DISC_MODE` is set to mode 5 (`101`) for the affected channel
- High preamp reset rate for the Ge channel → check `MOD###_DIG_PREAMP_RESET_DELAY` is reasonable (too large = blanks BGO for too long)
- `MOD###_DIG_CHANNEL_CONTROL` bit 3 (`PREAMP_RESET_DELAY_EN`) — if enabled and delay is large, BGO can be effectively silenced
- Monitor `MOD###_DIG_HIHILOLO_CNT` to see how often the preamp is resetting

**Fix:** Reduce `PREAMP_RESET_DELAY` or disable it (`PREAMP_RESET_DELAY_EN=0`) for the affected channel.

*(Source: `preamp_reset_readme.md`, `Digitizer.vhd:L1121-1130`)*

---

## Problem: InfluxDB Detector Temperatures Stale / pi5-lnFill SSH Failure

**Symptom:** Grafana temperature dashboard shows stale data (no new points for many hours). InfluxDB query returns data timestamped hours ago. General DGS heartbeat reports `⚠️ InfluxDB data stale`.

**Root cause:** `SaveTemp.sh` runs every 10 minutes on **pi5-lnFill** (`192.168.203.58`). If pi5-lnFill is unreachable (SSH refusing connections, crashed, or OS-stuck), temperature writes to InfluxDB stop. Meanwhile DCS2 (`LNFill_ping_cron.sh`) detects the SSH failure and alerts Discord: `"⚠️ pi5-lnFill (192.168.203.58) is unreachable via SSH"`.

**Diagnosis:**
```bash
# From spark-ca9f or any host on onenet:
ssh dgs@192.168.203.58

# Check last InfluxDB write (via spark-ca9f):
TOKEN=$(grep '^Token:' ~/workspace/secrets/influx3_read.token | awk '{print $2}')
curl -s "http://192.168.203.56:8181/api/v3/query_sql" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{"db":"HPGeTemp","q":"SELECT time,value FROM \"Temperature\" ORDER BY time DESC LIMIT 1"}'
```

**Recovery (if pi5-lnFill SSH is available):**
```bash
ssh dgs@192.168.203.58
# Check if SaveTemp.sh is running:
pgrep -f SaveTemp.sh

# Trigger a manual temperature collection:
cd /home/dgs/lnFill
bash SaveTemp.sh
```

**Recovery (if pi5-lnFill is completely unreachable — power cycle or crash):**
1. Physically power-cycle pi5-lnFill at the rack
2. After reboot, cron restarts automatically — temperatures resume within 10 minutes
3. Verify SSH access is restored: `ssh dgs@192.168.203.58`
4. Check `journalctl -b` (on pi5) for crash cause (persistent journal enabled 2026-04-21)
5. DCS2 backup fill watchdog covers fills in the meantime (no manual fill action needed)

**Note:** Temperatures stale for <30 min = transient hiccup (SaveTemp.sh can hang occasionally). Stale for >1 hour + SSH failure = genuine pi5 problem requiring action.

**See also:** `lnfill.md` — System Roles + Emergency Fallback sections; `influxdb_grafana.md` — InfluxDB query API.

*(Documented: 2026-04-21 — based on incident where pi5-lnFill SSH was refusing connections for 9+ hours)*

---

## Useful PVs for Diagnostics

| PV | What It Shows |
|----|--------------|
| `VMExx:MDIG1:trigger_mux_select_RBV` | Current trigger mode (ExtTTCL/IntAcptAll/etc) |
| `VME99:MTRG:EN_NIM1_DELAY` | NIM In 1 delay enabled? |
| `VMExx:MTRG:MISC_STAT` | NIM IN 1/2 state + misc status bits |
| `GS$(N)_SBX_Present` | Is SBX connected for this detector? |
| `GS$(N)_SlopeBoxGe_HV_On` | Is Ge HV on for this detector? |
| `DAQC$(CRATE)_OL_TotalBufsLost` | Total outLoop buffers lost (data loss counter) |
| `DAQC$(CRATE)_CV_OutLoop{0-6}` | Per-board data lost in outLoop |
| `DAQC$(CRATE)_OL_DataRate{0-6}` | Per-board output rate (kB/s) |
| `DAQC$(CRATE)_CV_SendRate` | Total MiniSender output rate (kB/s) |
| `DAQC$(CRATE)_OL_NumFreeBuffers` | Free buffer pool entries (watch for depletion) |
| `MOD###_DIG_PREAMP_RESET_DELAY` | PRK holdoff delay per channel (8-bit x 512 x 10 ns) |
| `MOD###_DIG_HIHILOLO_CNT` | Preamp reset event counter per channel |

---

## Cross-References

- `knowledgeBase/ioc.md` — IOC config, boot scripts, firmware versions
- `knowledgeBase/IOC_cmd.md` — VxWorks / iocsh commands for diagnostics (VMERead32, dbl, dbpr)
- `knowledgeBase/link_sys_analysis.md` — SERDES link init sequence (5-stage); often the cause of SYNC/lock issues
- `knowledgeBase/trig_setup_scripts.md` — 5-stage trigger setup scripts (trig_setup_Stage1–5.sh): full step-by-step MTRG→RTRG→DIG init; use when bringing up the system from cold
- `knowledgeBase/DIG_firmware_expert.md` — trigger_mux_select modes, readout configuration
- `knowledgeBase/preamp_reset_readme.md` — PREAMP_RESET handling: CHANNEL_KILLED, delay register
- `knowledgeBase/EPICS.md` — EPICS CA tools for PV inspection
- `knowledgeBase/VME_registers.md` — VME register addresses for low-level hardware inspection
- `knowledgeBase/run_procedures.md` — Standard run start/stop procedures
- `knowledgeBase/lnfill.md` — LN2 fill system: pi5-lnFill roles, emergency fallback (DCS2 backup watchdog), fill script locations
- `knowledgeBase/influxdb_grafana.md` — InfluxDB 3 on DCS2: query API, detector temperature database (HPGeTemp)

---

*Source: [wiki: Some Problems and Their Solutions](https://wiki.anl.gov/gsdaq/Some_problems_and_their_solutions) + `wiki_gsdaq.md` trigger setup section + operational notes. Created: 2026-04-05.*
