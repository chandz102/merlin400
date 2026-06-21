# Merlin400 — Codex Handover

**Date:** 2026-06-19 (started 2026-06-18)  
**Pi:** `pi@<PI_IP_ADDRESS>` (SSH key-based access; password login available as fallback)  
**Software root:** `~/merlin400-system/`  
**Upstream repo:** https://github.com/64bandil/merlin400/tree/main  
**Private repo:** https://github.com/chandz102/merlin400-system  
**Local checkout:** User's local development directory  
**Current private repo commit:** `bdfc44d` (`main`, `origin/main`) — `Mark FSM states refactor complete`  
**Project notes:** Documentation in project folder  
**Handover rule:** Update this file after every meaningful code change, deploy, verification step, or repo/remotes change.

---

## Access (SSH)

From your development machine:
- **`ssh merlin400`** — key-based SSH configured in `~/.ssh/config`. `scp`/`ssh merlin400 '<cmd>'` work non-interactively.
- Fallback: SSH key-based or password login (credentials stored securely)

Deploy pattern: `scp <file> merlin400:/home/pi/merlin400-system/src/...` → `ssh merlin400 'sudo systemctl restart merlin400-system'` → verify `ssh merlin400 'curl -s http://localhost/api/status'` is Ready/idle/pump 0.
- `__pycache__` is root-owned → syntax-check with `python3 -c "import ast; ast.parse(open(f).read())"` (not `py_compile`).
- `config.ini` is root-owned and **gitignored** → edit with `sudo`.

---

## System Overview

Aftermarket open-source firmware for the discontinued Drizzle Merlin400 extractor machine.

- **Entry point:** `src/startup.py` — starts two daemon threads: `ControlThread` and `ServerThread` (Flask on port 80)
- **State machine:** `src/hardware/module_FSM.py` (FSM wiring/orchestrator only; all `State*` classes live in `src/hardware/states/`)
- **Hardware abstraction:** `src/hardware/module_HardwareControlSystem.py`
- **API:** Flask REST at `http://192.168.1.130/api/`
- **Frontend:** Static files in `wwwroot/`
- **Config:** `~/merlin400-system/config.ini`
- **Stats DB:** `~/merlin400-system/stats.db` (SQLite)
- **Logs:** `~/merlin400-system/logs/drizzle_log_*.txt`

### FSM State Flow (Full Extraction)
```
StateReady → StateSystemCheck → StatePreFillTubes → StateFirstDepressurize
→ StateMeasureEXCVolume → StateSecondDepressurize → StateSecondLeakCheck
→ StateTopUpEXC → StateSoak → StateThirdDepressurize → StateAspirate
→ StateFlush → StateExtraFlushDepressurize → StateDistillBulk
→ StateAfterDistill → StateFinalSolventRemoval → StateReady
```
**Extract Only** (programId=5, `runFull=False`) — see `src/hardware/EXTRACT_ONLY_FLOW.md` for full traced path.

### API Endpoints
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/status` | Full machine status JSON |
| GET | `/api/stats` | Runtime stats + session history from stats.db |
| POST | `/api/start/<id>` | Start program (1=Full, 2=Decarb, 3=HeatOil, 4=Distill, 5=ExtractOnly, 6=VentPump) |
| POST | `/api/pause` | Pause current program |
| POST | `/api/resume` | Resume paused program |
| POST | `/api/reset` | Reset machine to Ready |
| POST | `/api/startcleanpump` | Run pump clean cycle |
| POST | `/api/cleanvalve/<n>` | Clean valve n |
| GET | `/logfile` | Download latest log file |

---

## Completed Work

### 1. Pump not stopped on reset
**File:** `src/hardware/commands/reset.py`  
`Command_Reset.execute()` set valves, heater, and fan to 0 but never touched the pump.  
**Fix:** Added `hardwareControlSystem.pump_value = 0` after `fan_value = 0`.

### 2. Pump stayed on after extraction completed
**File:** `src/hardware/module_FSM.py` — `StateReady.Enter()`  
Pump and fan were not shut down when machine returned to Ready after a run.  
**Fix:** Added `self.FSM.machine.fan_value = 0` and `self.FSM.machine.pump_value = 0`.

### 3. Log endpoint hardcoded year
**File:** `src/webserver.py`  
`/logfile` always served `drizzle_log_2024.txt`.  
**Fix:** Now finds and serves the newest `drizzle_log_*.txt` dynamically.

### 4. Heartbeat warning flood
**File:** `src/startup.py`  
Logged `Heartbeat not set...` every 0.5s during state transitions.  
**Fix:** Added `_heartbeat_warning_active` flag — now logs once per unset stretch only.

### 5. Stats endpoint
**Files:** `src/common/stats_reader.py`, `src/webserver.py`  
Implemented `get_summary()` + `get_history()`. `/api/stats` now live and verified.

### 6. Config validation on startup
**Files:** `src/common/config_validator.py`, `src/startup.py`  
Validates all critical numeric keys in `[FSM_EV]`, `[FSM_EX]`, `[DECARB]`, `[PID]` before hardware boots. Verified: 0 errors against current `config.ini`.

### 7. GitHub mirror + remotes
- Private repo: `chandz102/merlin400-system`
- Latest pushed commit: `bdfc44d` (`Mark FSM states refactor complete`)
- Local checkout is clean and matches `origin/main` as of 2026-06-18
- `origin` → `https://github.com/chandz102/merlin400-system.git`
- `upstream` → `https://github.com/64bandil/merlin400.git`

### Deploy note
Files must go into `src/common/` not `src/` — first deploy of stats/config-validator missed this. Always match package paths.

### Repo sync note
The deployed stats endpoint, config validator, `StateFlush.Exit()` fix, Extract Only flow docs, `src/hardware/states/` refactor scaffold, FSM base extraction, and all state moves through `StateDistillBulk` are committed and pushed to the private repo. The README completion note is pushed at `bdfc44d`.

### FSM refactor progress
- **Step 1 complete and deployed:** moved `FailureMode`, `Machine`, `Transition`, and base `State` into `src/hardware/states/base.py`
- `src/hardware/module_FSM.py` now re-exports `State`, `Transition`, `FailureMode`, and `Machine` from `hardware.states.base`
- Verified locally: `python3 -m py_compile src/hardware/module_FSM.py src/hardware/states/base.py`
- Verified on Pi: `sudo python3 -m py_compile src/hardware/module_FSM.py src/hardware/states/base.py`
- Restarted `merlin400-system`; service returned `active`
- Verified `/api/status`: `Ready` / `idle`, `pump_power: 0`, heater off
- Note: non-sudo Pi compile hit root-owned `__pycache__` permissions; sudo compile is clean
- **Step 2 complete and deployed:** moved `StateCooldown` into `src/hardware/states/state_cooldown.py`
- Verified locally: `python3 -m py_compile src/hardware/module_FSM.py src/hardware/states/state_cooldown.py`
- Verified on Pi: `sudo python3 -m py_compile src/hardware/module_FSM.py src/hardware/states/state_cooldown.py`
- Restarted `merlin400-system`; service returned `active`
- Verified `/api/status`: `Ready` / `idle`, `pump_power: 0`, heater off
- **Step 3 complete and deployed:** moved `StateError` into `src/hardware/states/state_error.py`
- Verified locally: `python3 -m py_compile src/hardware/module_FSM.py src/hardware/states/state_error.py`
- Verified on Pi: `sudo python3 -m py_compile src/hardware/module_FSM.py src/hardware/states/state_error.py`
- Restarted `merlin400-system`; service returned `active`
- Verified `/api/status`: `Ready` / `idle`, `pump_power: 0`, heater off
- **Step 4 complete and deployed:** moved `StateReady` into `src/hardware/states/state_ready.py`
- Verified locally: `python3 -m py_compile src/hardware/module_FSM.py src/hardware/states/state_ready.py`
- Verified on Pi: `sudo python3 -m py_compile src/hardware/module_FSM.py src/hardware/states/state_ready.py`
- Restarted `merlin400-system`; service returned `active`
- Verified `/api/status`: `Ready` / `idle`, `pump_power: 0`, heater off
- **Step 5 complete and deployed:** moved `StateSoak` into `src/hardware/states/state_soak.py`
- Verified locally: `python3 -m py_compile src/hardware/module_FSM.py src/hardware/states/state_soak.py`
- Verified on Pi: `sudo python3 -m py_compile src/hardware/module_FSM.py src/hardware/states/state_soak.py`
- Restarted `merlin400-system`; service returned `active`
- Verified `/api/status`: `Ready` / `idle`, `pump_power: 0`, heater off
- **Step 6 complete and deployed:** moved `StatePreFillTubes` into `src/hardware/states/state_pre_fill_tubes.py`
- Verified locally: `python3 -m py_compile src/hardware/module_FSM.py src/hardware/states/state_pre_fill_tubes.py`
- Verified on Pi: `sudo python3 -m py_compile src/hardware/module_FSM.py src/hardware/states/state_pre_fill_tubes.py`
- Restarted `merlin400-system`; service returned `active`
- Verified `/api/status`: `Ready` / `idle`, `pump_power: 0`, heater off
- **Step 7 complete and deployed:** moved `StateFirstDepressurize` into `src/hardware/states/state_first_depressurize.py`
- Verified locally: `python3 -m py_compile src/hardware/module_FSM.py src/hardware/states/state_first_depressurize.py`
- Verified on Pi: AST parse clean for `src/hardware/module_FSM.py` and `src/hardware/states/state_first_depressurize.py`
- Restarted `merlin400-system`; service returned `active`
- Verified `/api/status`: `Ready` / `idle`, `pump_power: 0`, heater off
- **Step 8 complete and deployed:** moved `StateSecondDepressurize` into `src/hardware/states/state_second_depressurize.py`
- Verified locally: `python3 -m py_compile src/hardware/module_FSM.py src/hardware/states/state_second_depressurize.py`
- Verified on Pi: AST parse clean for `src/hardware/module_FSM.py` and `src/hardware/states/state_second_depressurize.py`
- Restarted `merlin400-system`; service returned `active`
- Verified `/api/status`: `Ready` / `idle`, `pump_power: 0`, heater off
- **Step 9 complete and deployed:** moved `StateThirdDepressurize` into `src/hardware/states/state_third_depressurize.py`
- Verified locally: `python3 -m py_compile src/hardware/module_FSM.py src/hardware/states/state_third_depressurize.py`
- Verified on Pi: AST parse clean for `src/hardware/module_FSM.py` and `src/hardware/states/state_third_depressurize.py`
- Restarted `merlin400-system`; service returned `active`
- Verified `/api/status`: `Ready` / `idle`, `pump_power: 0`, heater off
- **Step 10 complete and deployed:** moved `StateExtraFlushDepressurize` into `src/hardware/states/state_extra_flush_depressurize.py`
- Verified locally: `python3 -m py_compile src/hardware/module_FSM.py src/hardware/states/state_extra_flush_depressurize.py`
- Verified on Pi: AST parse clean for `src/hardware/module_FSM.py` and `src/hardware/states/state_extra_flush_depressurize.py`
- Restarted `merlin400-system`; service returned `active`
- Verified `/api/status`: `Ready` / `idle`, `pump_power: 0`, heater off
- **Step 11 complete and deployed:** moved `StateMeasureEXCVolume` into `src/hardware/states/state_measure_exc_volume.py`
- Verified locally: `python3 -m py_compile src/hardware/module_FSM.py src/hardware/states/state_measure_exc_volume.py`
- Verified on Pi: AST parse clean for `src/hardware/module_FSM.py` and `src/hardware/states/state_measure_exc_volume.py`
- Restarted `merlin400-system`; service returned `active`
- Verified `/api/status`: `Ready` / `idle`, `pump_power: 0`, heater off
- **Step 12 complete and deployed:** moved `StateSecondLeakCheck` into `src/hardware/states/state_second_leak_check.py`
- Verified locally: `python3 -m py_compile src/hardware/module_FSM.py src/hardware/states/state_second_leak_check.py`
- Verified on Pi: AST parse clean for `src/hardware/module_FSM.py` and `src/hardware/states/state_second_leak_check.py`
- Restarted `merlin400-system`; service returned `active`
- Verified `/api/status`: `Ready` / `idle`, `pump_power: 0`, heater off
- **Step 13 complete and deployed:** moved `StateTopUpEXC` into `src/hardware/states/state_top_up_exc.py`
- Verified locally: `python3 -m py_compile src/hardware/module_FSM.py src/hardware/states/state_top_up_exc.py`
- Verified on Pi: AST parse clean for `src/hardware/module_FSM.py` and `src/hardware/states/state_top_up_exc.py`
- Restarted `merlin400-system`; service returned `active`
- Verified `/api/status`: `Ready` / `idle`, `pump_power: 0`, heater off
- **Step 14 complete and deployed:** moved `StateAfterDistill` into `src/hardware/states/state_after_distill.py`
- Verified locally: `python3 -m py_compile src/hardware/module_FSM.py src/hardware/states/state_after_distill.py`
- Verified on Pi: AST parse clean for `src/hardware/module_FSM.py` and `src/hardware/states/state_after_distill.py`
- Restarted `merlin400-system`; service returned `active`
- Verified `/api/status`: `Ready` / `idle`, `pump_power: 0`, heater off
- **Step 15 complete and deployed:** moved `StateFinalSolventRemoval` into `src/hardware/states/state_final_solvent_removal.py`
- Verified locally: `python3 -m py_compile src/hardware/module_FSM.py src/hardware/states/state_final_solvent_removal.py`
- Verified on Pi: AST parse clean for `src/hardware/module_FSM.py` and `src/hardware/states/state_final_solvent_removal.py`
- Restarted `merlin400-system`; service returned `active`
- Verified `/api/status`: `Ready` / `idle`, `pump_power: 0`, heater off
- **Step 16 complete and deployed:** moved `StateVentPump` into `src/hardware/states/state_vent_pump.py`
- Verified locally: `python3 -m py_compile src/hardware/module_FSM.py src/hardware/states/state_vent_pump.py`
- Verified on Pi: AST parse clean for `src/hardware/module_FSM.py` and `src/hardware/states/state_vent_pump.py`
- Restarted `merlin400-system`; service returned `active`
- Verified `/api/status`: `Ready` / `idle`, `pump_power: 0`, heater off
- **Step 17 complete and deployed:** moved `StateDecarb` into `src/hardware/states/state_decarb.py`
- Verified locally: `python3 -m py_compile src/hardware/module_FSM.py src/hardware/states/state_decarb.py`
- Verified on Pi: AST parse clean for `src/hardware/module_FSM.py` and `src/hardware/states/state_decarb.py`
- Restarted `merlin400-system`; service returned `active`
- Verified `/api/status`: `Ready` / `idle`, `pump_power: 0`, heater off
- **Step 18 complete and deployed:** moved `StateMixOil` into `src/hardware/states/state_mix_oil.py`
- Verified locally: `python3 -m py_compile src/hardware/module_FSM.py src/hardware/states/state_mix_oil.py`
- Verified on Pi: AST parse clean for `src/hardware/module_FSM.py` and `src/hardware/states/state_mix_oil.py`
- Restarted `merlin400-system`; service returned `active`
- Verified `/api/status`: `Ready` / `idle`, `pump_power: 0`, heater off
- **Step 19 complete and deployed:** moved `StateCleanPump` into `src/hardware/states/state_clean_pump.py`
- Verified locally: `python3 -m py_compile src/hardware/module_FSM.py src/hardware/states/state_clean_pump.py`
- Verified on Pi: AST parse clean for `src/hardware/module_FSM.py` and `src/hardware/states/state_clean_pump.py`
- Restarted `merlin400-system`; service returned `active`
- Verified `/api/status`: `Ready` / `idle`, `pump_power: 0`, heater off
- **Step 20 complete and deployed:** moved `StateFlush` into `src/hardware/states/state_flush.py`
- Verified locally: `python3 -m py_compile src/hardware/module_FSM.py src/hardware/states/state_flush.py`
- Verified locally: `StateFlush.Exit.__qualname__` resolves to `StateFlush.Exit`
- Verified on Pi: AST parse clean for `src/hardware/module_FSM.py` and `src/hardware/states/state_flush.py`; `StateFlush.Exit.__qualname__` resolves to `StateFlush.Exit`
- Restarted `merlin400-system`; service returned `active`
- Verified `/api/status`: `Ready` / `idle`, `pump_power: 0`, heater off
- **Step 21 complete and deployed:** moved `StateAspirate` into `src/hardware/states/state_aspirate.py`
- Verified locally: `python3 -m py_compile src/hardware/module_FSM.py src/hardware/states/state_aspirate.py`
- Verified locally: `StateAspirate.Exit.__qualname__` resolves to `StateAspirate.Exit`
- Verified on Pi: AST parse clean for `src/hardware/module_FSM.py` and `src/hardware/states/state_aspirate.py`; `StateAspirate.Exit.__qualname__` resolves to `StateAspirate.Exit`
- Restarted `merlin400-system`; service returned `active`
- Verified `/api/status`: `Ready` / `idle`, `pump_power: 0`, heater off
- **Step 22 complete and deployed:** moved `StateSystemCheck` into `src/hardware/states/state_system_check.py`
- Verified locally: `python3 -m py_compile src/hardware/module_FSM.py src/hardware/states/state_system_check.py`
- Verified on Pi: AST parse clean for `src/hardware/module_FSM.py` and `src/hardware/states/state_system_check.py`; `StateSystemCheck.Exit.__qualname__` resolves to `StateSystemCheck.Exit`
- Restarted `merlin400-system`; service returned `active`
- Verified `/api/status`: `Ready` / `idle`, `pump_power: 0`, heater off
- **Step 23 complete and deployed:** moved `StateDistillBulk` into `src/hardware/states/state_distill_bulk.py`
- Verified locally: `python3 -m py_compile src/hardware/module_FSM.py src/hardware/states/state_distill_bulk.py`
- Verified locally: `module_FSM.py` now only defines `class FSM`; all `State*` classes live under `src/hardware/states/`
- Verified on Pi: AST parse clean for `src/hardware/module_FSM.py` and `src/hardware/states/state_distill_bulk.py`; `StateDistillBulk.Exit.__qualname__` resolves to `StateDistillBulk.Exit`
- Restarted `merlin400-system`; service returned `active`
- Verified `/api/status`: `Ready` / `idle`, `pump_power: 0`, heater off

---

### 8. StateFlush.Exit bug — ROOT CAUSE of the "pump on after Extract Only" symptom (FIXED & DEPLOYED 2026-06-18)
**File:** `src/hardware/module_FSM.py`  
`def Exit(self):` was defined at **module scope (zero indentation)** between `StateFlush.Execute()` and `class StateExtraFlushDepressurize` — orphaned, so `StateFlush` never overrode `Exit()` and the base `State.Exit()` ran (no pump-off, no valve close). This was the true root cause of the "pump still running after Extract Only" symptom; Codex's earlier `StateReady.Enter()` fix only masked it.  
**Fix:** indented the function back into `StateFlush` as a proper method (body was already correct: pump off, sleep, close all valves).  
**Verified:** `ast.parse` clean, `StateFlush` now has Enter/Execute/Exit; service `active`, machine `Ready`/`idle`, pump=0, heater=0. Pi backup `module_FSM.py.bak-flushfix`. Local checkout patched identically. Full end-to-end confirmation will come on the next real Extract Only run.

---

### 9. "Flush to Distiller" feature — BUILT, THEN REMOVED 2026-06-19
A manual on-demand flush was added (suction-based, mirroring aspirate) to pull residual solvent from the extraction chamber into the distiller. After testing, the user determined it won't achieve the goal (geometry/vacuum limits) and asked to remove it. **Fully reverted at commit `9e3644f`:** deleted `state_manual_flush.py`, `start_flush.py`, the `/api/flush` route, the UI button/JS, the `config_validator` check, the FSM wiring, and the `manual_flush_time` key (removed from the Pi's `config.ini` too). Verified: service active, Ready/idle/pump 0, `/api/flush` no longer a valid endpoint, UI button gone. Do NOT re-add.

### 10. Heater-check hardening against transient bad thermistor reads (DEPLOYED 2026-06-18)
**File:** `src/hardware/module_FSM.py` — `StateDistillBulk.heater_temperature_increase_check()`  
**Why:** a Distillation Only run errored with "heater stopped heating" — root cause was a single bad bottom-thermistor reading (1.07°C vs ~25°C ambient); the heater was actually warming. A transient sensor dropout shouldn't abort a distill.  
**Fix:** when the "did not increase" condition fires, check if the reading is implausible (`< 5°C` absolute, or `> 5°C` below the captured baseline — heating can't cool the plate). If so, treat as a sensor glitch: log a warning, re-arm the check window (keep the known-good baseline), and continue. Only error after `_heater_check_max_bad_reads = 3` consecutive glitches (→ "faulty thermistor/cable" message). A *plausible* flat temperature still errors immediately, so genuine heater failure detection is preserved. Pi backup: `module_FSM.py.bak-heaterguard`. Deployed, service active, Ready/idle.

> NOTE: likely also a physical issue — intermittent bottom-heater thermistor connection. Worth checking the connector. This fix prevents a transient blip from aborting a run; it does not fix a persistently bad sensor (that still errors after 3 glitches).

---

## Git state
All work committed + pushed to `origin/main` at **`bdfc44d`**. History: StateFlush.Exit fix #8 in `63785a4`; Flush feature #9 + heater guard #10 in `fe7821f`; Flush feature removal in `9e3644f` (heater guard #10 retained); FSM Step 7 in `9f5a28d`; FSM Step 8 in `7c256dc`; FSM Step 9 in `9fdd2cb`; FSM Step 10 in `bf630b0`; FSM Step 11 in `97ace2f`; FSM Step 12 in `68ff164`; FSM Step 13 in `809841e`; FSM Step 14 in `35ac91b`; FSM Step 15 in `abecc7d`; FSM Step 16 in `fc717fe`; FSM Step 17 in `ff4a08a`; FSM Step 18 in `3dacbe9`; FSM Step 19 in `754837c`; FSM Step 20 in `435b86f`; FSM Step 21 in `480df4e`; FSM Step 22 in `7f8d41e`; FSM Step 23 in `d614672`; README completion note in `bdfc44d`.
- NOTE: `config.ini` is **gitignored** (per-machine state the firmware rewrites).

---

## Open Work

### FSM refactor — COMPLETE
Split `module_FSM.py` into one file per state under `src/hardware/states/`. `module_FSM.py` is now the FSM wiring/orchestrator layer, while state behavior lives in the state package.

- **Done (deployed + pushed, code at `d614672`, README completion at `bdfc44d`):** Step 1 base.py (State/Transition/FailureMode/Machine + re-export), Step 2 StateCooldown, Step 3 StateError, Step 4 StateReady, Step 5 StateSoak, Step 6 StatePreFillTubes, Step 7 StateFirstDepressurize, Step 8 StateSecondDepressurize, Step 9 StateThirdDepressurize, Step 10 StateExtraFlushDepressurize, Step 11 StateMeasureEXCVolume, Step 12 StateSecondLeakCheck, Step 13 StateTopUpEXC, Step 14 StateAfterDistill, Step 15 StateFinalSolventRemoval, Step 16 StateVentPump, Step 17 StateDecarb, Step 18 StateMixOil, Step 19 StateCleanPump, Step 20 StateFlush, Step 21 StateAspirate, Step 22 StateSystemCheck, Step 23 StateDistillBulk
- **Remaining:** No FSM split work remains. Recommended next validation: run a real extraction/distillation smoke test when convenient.
- **Latest live check:** after final README push (`bdfc44d`), `merlin400-system` is `active`; `/api/status` reports `Ready` / `idle`, `pump_power: 0`, heater off.

### Alcohol sensor (parked)
`ALCOHOL_SENSOR_ENABLED = False` in `src/common/settings.py`. Hardware code and FSM logic already exists — only relevant if sensor hardware is physically present.

---

## How to Deploy Changes

```bash
# SSH to Pi
ssh pi@192.168.1.130

# Files go in ~/merlin400-system/src/ (match package paths — common/ states/ etc.)
# Restart service
sudo systemctl restart merlin400-system

# Verify
sudo journalctl -u merlin400-system -f
curl -s http://localhost/api/status | python3 -m json.tool
```
