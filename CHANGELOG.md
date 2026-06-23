# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.3] - 2026-06-23

### Added

- **`memcal_db` module** — parses `MEMCALS.md` (183 entries:
  VN/VP, VR, VS, VT, VX). `lookup_by_bcc`,
  `lookup_by_filename`, `lookup_by_pid`, `iter`. BCC
  column shows the full 8-character code for entries with
  a known suffix, the 4-letter prefix otherwise.
- **Memcal toolbar** (one row above the bin row): `[Search]`
  opens a searchable browser over the database. `[Info]`
  (on the bin row, next to `Clear`) looks the loaded bin
  up by its 4-letter prefix.
- **Wire-signal invert + echo suppression** under the port
  row. `Invert: TX / RX` are mutually-exclusive radio
  buttons (XOR each byte with 0xFF at the port boundary,
  default both off). `Echo Suppress` is a separate
  checkbox (default off) for passive 1-wire adapters;
  dropped bytes are logged as `Direction::Echo` for
  verification.
- **`Set Faults` scenario** — drives the per-frame auto-DTC
  scan to a known multi-code state (CTS 145°C, MAT -45°C,
  MAP 8 kPa, O2 stuck + slow, RPM 2250 / TPS 0% correlation).
  No `state.malfs` is set directly; the scan + correlation
  check are the test target. Useful for verifying the
  debounce timers and the tune-gated enable bitfield work
  end-to-end.

### Changed

- **Bin row no longer shows the program-ID / version tag.**
  File name, size, `Load .bin...`, `Clear`, `Info` only.
  ID and version moved to the `Info` popup.
- **Total: 168 unit tests across 12 source files + 4
  integration tests** in `tests/integration.rs`.

## [0.1.2] - 2026-06-23

### Added

- **TPS-driven engine model.** A piecewise-linear
  `target_<x>_for_tps` table per channel, plus a 2D
  `compute_spark_advance(rpm, map)` lookup. All values are
  placeholders for what would normally be loaded from the `.bin`
  via the OSE XDF.
  - **RPM** — `TPS_TARGET_RPM`: 800 at idle → 5500 at WOT.
    Same tolerance-band correlation check as 0.1.2-rc1.
  - **MAP (kPa)** — `MAP_TARGET_FOR_TPS`: 35 at idle → 95 at WOT.
    Direct sensor relationship.
  - **Injector pulse width (ms)** — `BPW_TARGET_FOR_TPS`:
    2.0 at idle → 12.0 at WOT. More air needs more fuel.
  - **IAC stepper (counts)** — `IAC_TARGET_FOR_TPS`: 80 at
    idle → 0 off-idle. Inverse relationship (IAC supplies
    air at closed throttle).
  - **Target AFR** — `AFR_TARGET_FOR_TPS`: 14.7 stoich at
    cruise → 12.5 at WOT. Power enrichment above ~70% TPS.
  - **VE (%)** — `VE_TARGET_FOR_TPS`: non-monotonic
    (40 at idle → 88 at cruise → 75 at WOT). Pumping loss
    at WOT.
  - **Spark advance (°BTDC)** — `SPARK_TABLE`: 4×4 lookup
    indexed by RPM bin (800/2000/3500/5000) and MAP bin
    (35/50/70/95). Peaks in the cruise band, retreats at
    WOT to control knock.
- `EngineState::apply_throttle(tps_pct)` — now updates
  `tps_pct`, `rpm`, `map_kpa`, `injector_pw_ms`,
  `iac_position`, `target_afr`, `ve_pct`, and
  `spark_adv_deg` (the last derived from the new RPM and
  MAP) in one call.
- `EngineState::recompute_spark_advance()` — re-derives
  `spark_adv_deg` from the current `rpm` and `map_kpa`.
  Called per-frame in the GUI so manual RPM/MAP slider
  drags also keep spark in sync.
- **MALF code 67** (TPS/RPM correlation fault). Non-MIL,
  bit 28. Wired through the per-frame check in `gui.rs` so
  dragging the RPM slider out of spec trips the CEL;
  auto-clears when RPM returns to spec.
- **GUI: TPS slider drives everything.** Moving the
  throttle slider in the Primary Sensors group now calls
  the expanded `apply_throttle()`; RPM/MAP/BPW/IAC/AFR/VE/spark
  all snap to their targets for the new position.
- **Auto-DTC sensor scan.** New
  `EngineState::scan_sensor_diagnostics(&mut DiagnosticCounters, enabled_mask)`
  inspects 12 channels per call (O2 open / stuck lean / stuck
  rich / slow, CTS / TPS / MAT / MAP high+low) against OBD 0/1
  threshold constants and sets/clears the matching MALF bits.
  Consecutive-bad-sample debounce (`FAULT_DEBOUNCE_SET = 5`)
  sets a code; consecutive-good debounce (`FAULT_DEBOUNCE_CLEAR
  = 10`) clears it. O2 open takes precedence over stuck
  lean/rich (a disconnected sensor can't be "stuck"); O2 slow
  is gated on `o2_ready` so an open-loop cold engine doesn't
  false-positive. Wired into the per-frame GUI tick so the
  scan runs against slider drags, playback samples, and any
  other state source uniformly. Counters live on
  `EmulatorApp` so debounce works across frames.
- **Tune-gated DTC enable from the .bin.** The OSE $12P XDF
  carries a per-code enable bitfield at bin offsets
  `0x1998-0x199C` (5 bytes, 40 codes, baseoffset -32768).
  `Calibration::dtc_enabled(code)` and
  `Calibration::dtc_enabled_at(code, offset)` query that
  bitfield; the per-frame scan now passes an `enabled_mask`
  built from the active profile's offset, so a tune that
  disables, say, MALF 25 (MAT high) won't trip the CEL even
  if the MAT reading crosses the threshold. Profile-aware via
  the new `Profile::dtc_enable_offset()` trait method
  (default 0x1998 for the three OSE-family profiles; future
  non-OSE profiles can override). Codes not in the OSE table
  — including 61-67 (emulator-defined) — always read as
  enabled. Short or empty calibrations also default to all
  enabled, so non-OSE bins don't accidentally suppress DTCs.
- **47 new unit tests** in `src/ecu.rs` and `src/calibration.rs`:
  6 target lookups (RPM, MAP, BPW, IAC, AFR, VE), 2D spark
  interpolation (cell reads, bilinear midpoint, outer-bin
  clamp), expanded `apply_throttle` field set, auto-DTC scan
  (set/clear debounce, intermittent-noise rejection, O2
  mutual exclusion, per-channel threshold coverage,
  no-false-positive default, disabled-channel suppression,
  bad-counter reset on re-enable), and the calibration
  enable-bitfield parser (per-word reads, codes-not-in-table
  default, short/empty-bin fallback, custom offset).

### Changed

- **TPS high/low (MALF 21/22) excluded from the auto-DTC
  scan.** The user drives `tps_pct` via the slider by design,
  so 0% and 100% are valid engine states, not sensor faults.
  The auto-scan now skips channels 3 (TPS high) and 4 (TPS
  low) entirely — counters don't advance, bits aren't
  touched. MALF 21/22 can still be set/cleared by log DTC
  columns and manual MALF grid clicks.
- **Default log playback rate is now slow (1/6 realtime).**
  `Playback::DEFAULT_RATE = 1.0 / 6.0`. New `Playback::new()`
  and `Playback::load()` both reset the rate to this value,
  so a fresh app or a freshly loaded log starts at 1/6
  realtime — the user can see playback step at a glance.
  The GUI's Slow/Fast preset buttons (relocated to the
  transport row, right of the Loop toggle, with a small
  spacer) switch between `DEFAULT_RATE` and `1.0`; the
  active preset is shown with a `●` prefix and a background
  fill so the current speed is visible at a glance.
- **2 new unit tests** for the default-rate behaviour
  (`new_playback` returns `DEFAULT_RATE`, `load` resets
  to `DEFAULT_RATE`).
- **Total: 161 unit tests across 11 source files + 4
  integration tests** in `tests/integration.rs`.

### Added

- **`CylMode::Two` variant** for TBI-swap and 2-cyl engine
  conversions using the OSE $12P hardware family (the "808"
  ECU on a TBI or converted carb engine). Encodes to
  `0x00` at offset `0x1B` bits 4-5, completing the
  2-bit cylinder-mode field. New "TBI" radio button in
  the GUI's Cylinder mode group.
- **4 new unit tests** in `src/profile.rs` for per-variant
  cylinder-mode wire encoding (Two, Four, Six, Eight).
- **Total: 159 unit tests across 11 source files + 4
  integration tests** in `tests/integration.rs`.

## [0.1.1] - 2026-06-22

### Renamed

- **Project renamed from "ALDL ECU Emulator" to "ALDL ECU Simulator".**
  The project is a wire-protocol responder, not an engine emulator
  (no CPU model, no physics), so "Simulator" is the more accurate term.
  This is a **breaking change** for anyone with scripts/shortcuts
  pointing at the old binary names:
  - CLI binary: `aldl-emulator.exe` → `aldl-ecu-simulator.exe`
  - GUI binary: `ALDL-Ecu-Emulator.exe` → `ALDL-ECU-Simulator.exe`
  - Crate name: `aldl-emulator` → `aldl-ecu-simulator`
  - Library name: `aldl_emulator` → `aldl_ecu_simulator`
  - CLI source: `src/bin/aldl-emulator.rs` →
    `src/bin/aldl-ecu-simulator.rs`
  - `RUST_LOG` filter: `aldl_emulator=*` → `aldl_ecu_simulator=*`
  - GUI window title: "ALDL ECU Emulator" → "ALDL ECU Simulator"
  - `build.sh` `BINARY_NAME`: `ALDL-Ecu-Emulator` →
    `ALDL-ECU-Simulator`

  The cross-built binary is now
  `target/x86_64-pc-windows-gnu/release/ALDL_ECU_Simulator_GUI-0.1.1.exe`
  (and `..._cli-0.1.1.exe` for the CLI).

### Added

- **Mode 4 actuator commands wired to state.** The default profile now
  mutates `EngineState` in response to scan-tool actuator commands,
  rather than acking-with-zeros:
  - `M4_FuelOn` (sub-id `0x40`, payload `0x40`) → toggles
    `state.fuel_pump_on`. Visible on the wire at status byte `0x0C`
    bit `0x08`.
  - `M4_HS_on` (sub-id `0x02`, payload `0x02`) → toggles
    `state.high_speed_fan_on`. Visible on the wire at status byte
    `0x1B` bit `0x01`. Payload `0x01` = low fan (clears high-speed
    bit); payload `0x00` = both fans off.
  - `M4_TCCon` (sub-id `0x00`, payload bytes 5-6 = `0x10 0xFF`) →
    sets `state.tcc_locked = true`. Visible on the wire at status
    byte `0x1A` bit `0x08`.
  - `20degAdvance` (sub-id `0x00`, payload byte 7 = `0x08`) → sets
    `state.spark_override = Some(20.0)`. Overrides the wire spark
    advance at data offset `0x1F-0x20` for the next Mode 1 sub-id
    `0x00` poll.
  - `M4BLMReset` (sub-id `0x00`, payload byte 2 = `0x10`) → acked
    only (the emulator has no block-learn cells to clear).
  - `M4_Norm` (sub-id `0x00`, all-zero payload) → resets TCC, fan,
    and spark override in one shot.
- **Mode 1 sub-id `0x07`: "Message 6 Dyno Modified".** 10-byte
  extended variant of the standard 8-byte Message 6. Adds three
  more bytes: TPS at offset 7, Current VE at offset 8, CTS at
  offset 9. Wire size: 14 bytes (vs 12 for standard). Driven from
  `state.tps_pct`, `state.ve_pct`, `state.clt_c`. Sub-id `0x06`
  (standard Message 6) is unchanged.
- **`EngineState::spark_override: Option<f32>`.** When `Some(deg)`,
  the full-Petrol encoder writes this value into the spark-advance
  wire bytes instead of `state.spark_adv_deg`. Default: `None`.
- **`Calibration::program_id() / firmware_major() / firmware_minor()`.**
  Read the 2-byte BE program ID at offset 0 and the major/minor
  version bytes at offsets 2-3 of the loaded `.bin`. OSE V1.12
  bins have program ID `0x0012`. (Removed from the bin row in
  0.1.3 — see Changed notes there.)
- **GUI: program ID tag in the bin row.** *(Removed in 0.1.3.)*
  Showed `ID 0x0012 v1.1` next to the loaded `.bin` filename,
  blue for the expected OSE `$12P` ID and amber for anything
  else.
- **GUI: "Mode 4 actuator overrides" section.** Displays the current
  spark override (Normal or forced value) with a "Clear (M4_Norm)"
  button, plus a "Reset all (M4_Norm)" button that clears TCC, fan,
  and spark override in one click. Greyed out when no overrides
  are active.
- **"Copy" button on the traffic log.** Tab-separated copy of
  the full log to the clipboard for pasting into Excel.
- **Active auto-scroll in the traffic log** — `stick_to_bottom` is
  passive (only sticks if you're already at the bottom), so when
  the checkbox is on and new entries arrive, the scroll position
  is force-set to the maximum.
- **8 new unit tests** for the actuator command path (TCC, fan on
  / off / low, 20° spark override, M4_Norm reset, BLM reset) and
  3 new tests for Message 6 Dyno Modified (wire size, field layout,
  running-flag behaviour).
- **Total: 102 unit tests across 10 source files + 4 integration
  tests** in `tests/integration.rs`.

### Changed

- **`Profile::handle_mode_4` now takes `&mut EngineState`.** The
  trait method signature changed from
  `fn handle_mode_4(&self, calibration: Option<&Calibration>, sub_id, payload)`
  to
  `fn handle_mode_4(&self, state: &mut EngineState, calibration: Option<&Calibration>, sub_id, payload)`.
  This lets actuator commands mutate the live state instead of just
  acking. The `$11` and `$51` profile impls accept and ignore the
  new parameter. `build_diagnostic_response` and all callers
  (CLI, GUI, tests) updated accordingly.
- **Read-memory sub-id moved from `0x02` to `0x03`.** The OSE protocol
  V1.12 ADX uses sub-id `0x02` for the fan actuator command
  (M4_HS_on), so the Delco-style read-memory command had to move.
  The OSE protocol doesn't define a read-memory command at all;
  `0x03` is an emulator extension for serving the loaded
  calibration back to the scan tool. **This is a breaking change
  for any scan tool previously configured to read memory at
  sub-id `0x02`** — reconfigure to `0x03`.
- **CLI / GUI / tests updated** to lock the state mutex mutably
  for Mode 4 requests (was previously a read-only lock + clone).

### Removed

- **Sparse / empty ADX and XDF files cleaned up** in a prior
  asset pass (see commit history).
- **Entire `assets/` directory deleted** — all 32 files (~2.5 MB)
  were user-loadable data (ADX reference, calibration bins, sample
  logs) that the simulator doesn't need at compile time or
  runtime. The OSE ADX files are public on delcohacking.net if
  you need to cross-check a field. Bins and logs are loaded
  by the user via the GUI's file dialog or by running
  `python3 tools/gen_cold_start_idle.py` to generate a sample log.

## [0.1.0] - 2026-06-15

### Added

- Initial release.
- Default profile with Mode 1 sub-ids `0x00` (56-byte petrol
  datastream, 60 bytes on wire) and `0x06` (8-byte Message
  6, 12 bytes on wire).
- `$11` (VR Commodore V8) profile stub: sub-id `0x00` (56 bytes),
  sub-id `0x04` (48-byte transmission data).
- `$51` (VS V6) profile stub: sub-id `0x00` (60 bytes), sub-id
  `0x03` (42-byte transmission data).
- Mode 4 sub-id `0x02` read-memory support (later moved to `0x03`
  in 0.1.1).
- CLI emulator (`aldl-emulator.exe`) with `--port`, `--baud`,
  `--profile`, `--bin`, `--rpm`, `--clt`, `--mat`, `--tps`,
  `--battery` flags.
- GUI emulator (`ALDL-Ecu-Emulator.exe`) with eframe 0.34:
  - Top header: port/profile/baud selectors + log controls
  - Bin row: load .bin, clear, hover for full path
  - Engine panel (right): CEL disc, Start/Stop pill, RPM gauge,
    MALF grid (28 codes), Clear button
  - Central scroll: Vehicle, Primary sensors, Fuel, Ignition,
    Idle, Status bits, Cylinder mode, Calibration read tester
  - Bottom: traffic log (RX/TX frames with timestamps)
  - Bottom-right: MIT license copyright overlay
- Calibration `.bin` loader (load, read by address/length, OOR
  pads zero).
- MALF/DTC bitfield (28 OBD-I GM codes) with `mil_active()`
  helper.
- Log playback: CSV (TunerPro RT + EFILive V4) and XDL (stub)
  parsers, column-name normalisation, `Playback` with play/pause,
  loop, range markers, custom playhead widget.
- Hardware-free smoke test: `tools/smoke_test.py` (Linux pty
  pair).
- Log generator: `tools/gen_cold_start_idle.py` produces a 6-min
  V6 cold-start-to-idle CSV at 2 Hz.
- COM-port loopback test: `tools/com_loopback_test.py` (16 frames
  + 8 KB sustained, 64 KB with `--large`).
- Cross-compile build script: `build.sh` → produces
  `ALDL-Ecu-Emulator_0.1.0.exe` for `x86_64-pc-windows-gnu`.
- 70 unit tests + 4 integration tests.

[0.1.2]: #012---2026-06-23
[0.1.1]: #011---2026-06-22
[0.1.0]: #010---2026-06-15
