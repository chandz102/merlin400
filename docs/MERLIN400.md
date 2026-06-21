# Merlin400 — Small Extractor (Pi)

## Overview

Aftermarket open-source software for the Merlin400 extractor (discontinued by Drizzle).
Running on a Raspberry Pi 2/3 (ARMv7) at `192.168.1.130`.

**Upstream repo:** https://github.com/64bandil/merlin400/tree/main
**Private repo (our fork):** https://github.com/chandz102/merlin400-system
**Local checkout:** `/Users/chandler/Documents/Merlin400`

**Related docs (in this folder):**
- `CODEX_HANDOVER.md` — full running log of all changes/deploys (keep current)
- `SET_MERLIN400_AS_DEFAULT_BOOT.md` — guide for making merlin400-system the default boot (for someone else)

## Hardware

| Component | Detail |
|-----------|--------|
| Pi model | ARMv7 (Pi 2 or 3) |
| OS | Raspbian Bullseye (11) |
| RAM | 426MB |
| Storage | 7.2GB SD card |
| Hostname | m40dec2 |
| Machine ID | 000000000b40dec2 |

## Access

```bash
ssh merlin400          # key-based alias (no password) — set up 2026-06-18
# or fallback:
ssh pi@192.168.1.130   # password: dubqengm  (password login still enabled)
```

- Key: `~/.ssh/merlin400_ed25519`, alias in `~/.ssh/config`. `scp`/`ssh merlin400 '<cmd>'` work non-interactively.
- Deploy: `scp <file> merlin400:/home/pi/merlin400-system/src/...` → `ssh merlin400 'sudo systemctl restart merlin400-system'` → verify `/api/status` is Ready/idle/pump 0.
- Gotchas: `__pycache__` is root-owned (syntax-check with `python3 -c "import ast; ast.parse(...)"`, not `py_compile`); `config.ini` is root-owned **and gitignored** (edit with `sudo`; per-machine, not in repo).

Web API: `http://192.168.1.130/api/status`

## Software

| File/Path | Purpose |
|-----------|---------|
| `~/merlin400-system/` | Aftermarket system software |
| `~/merlin400-system/src/startup.py` | Entry point (+ config validation on boot) |
| `~/merlin400-system/src/hardware/module_FSM.py` | FSM orchestrator/wiring only (~260 lines) |
| `~/merlin400-system/src/hardware/states/` | One file per FSM state (22 states) — refactored 2026-06-19 |
| `~/merlin400-system/src/hardware/commands/reset.py` | Reset command |
| `~/merlin400-system/src/webserver.py` | Flask API server (`/api/status`, `/api/stats`, programs) |
| `~/merlin400-system/src/common/config_validator.py` | Validates config.ini numeric keys on boot |
| `~/merlin400-system/src/common/stats_reader.py` | Backs `/api/stats` (runtime + session history) |
| `~/projects/merlin400/` | Project notes folder (this file) |

## Systemd Services

| Service | Status |
|---------|--------|
| `merlin400-system` | **Enabled** — starts on boot |
| `drizzle` | Disabled (original software, replaced) |

### Common commands

```bash
sudo systemctl status merlin400-system
sudo systemctl restart merlin400-system
sudo journalctl -u merlin400-system -f
```

## API

```bash
# Current status
curl http://192.168.1.130/api/status
```

Key fields in response:
- `machineState` — idle / running
- `currentStatus` — Ready / etc.
- `hardwareMonitor.pump_power` — 0 or 100
- `hardwareMonitor.pressure` — mbar
- `hardwareMonitor.bottom_heater_temperature` — °C
- `hardwareMonitor.gas_temp` — °C
- `activeProgram` — current extraction details

## Program Parameters (defaults)

| Parameter | Value |
|-----------|-------|
| dist_temperature | 125°C |
| after_heat_temp | 107°C |
| after_heat_time | 240s |
| soakTime | 10s |
| number_of_flushes | 1 |
| final_air_cycles | 16 |

## Runtime Stats

- Total run time: 1,821 minutes since 2024-05-16
- Firmware: 1.0-preview2

## Work done (2026-06-18 → 06-19)

> Full detail + commit hashes in `CODEX_HANDOVER.md`. Headlines:

**Bug fixes**
- `reset.py` — pump wasn't shut off on reset (added `pump_value = 0`).
- `StateReady.Enter()` — pump/fan not shut off when returning to Ready.
- **StateFlush.Exit** — was orphaned at module scope (not a method), so the pump never shut off / valves never closed on flush exit. This was the **real root cause** of "pump still running after Extract Only." Fixed by indenting it into the class. Confirmed in real runs.
- `/logfile` hardcoded `drizzle_log_2024.txt` → now serves newest log dynamically.
- Heartbeat warning spam during state transitions → logs once per stretch.

**Hardening**
- Heater check ignores transient bad thermistor reads (a single ~1°C glitch was falsely aborting distills); errors only on a plausible flat temp or 3 consecutive glitches. NOTE: also a likely **physical** intermittent bottom-heater thermistor connection — worth a connector check.
- Config validation on startup (`config_validator.py`) — bad numeric config caught before hardware boots.

**Features**
- `/api/stats` endpoint — runtime totals + session history from `stats.db`.
- "Flush to Distiller" manual flush — built, tested, then **removed** at user's request (won't achieve the goal; the pump can only suck, not push). Do not re-add.

**Refactor**
- Split the 2761-line `module_FSM.py` into one file per state under `src/hardware/states/` (22 states + `base.py`). `module_FSM.py` is now ~260 lines (orchestrator only). Done one state at a time, each verified on the Pi.

**Validated on real hardware**
- Full Extract (program 1) ran end-to-end with real plant matter (2026-06-19): aspirate → flush → distill → after-heat → final solvent removal → Ready. All states worked.
- Volume check: dry run aspirated to the 150 mL target (then flush adds ~20 mL more → ~170 mL in chamber). With plant, aspirate flows slower but still reaches target. Clean mid-aspirate accuracy check still TODO if wanted.

## Notes

- Alcohol sensor is disabled — machine runs full extraction mode
- Machine runs a pressure/valve/heater self-check on every startup before becoming Ready
- API is polled every 5 seconds by the frontend at `192.168.1.96`
