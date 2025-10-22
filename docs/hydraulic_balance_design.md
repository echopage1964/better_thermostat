# Decentralized Hydraulic Balance in Better Thermostat

This document captures the design and reasoning behind the decentralized hydraulic balance feature.

## Problem
We want to emulate a hydraulic balance across multiple TRVs without any global supply signals or direct valve feedback. Most devices only accept a temperature setpoint. Some devices (e.g., Sonoff TRVZB) expose min/max opening as percentages.

## Approach
- Per room/thermostat, use:
  - ΔT = setpoint - current temperature
  - Temperature trend (slope dT/dt over ~5–10 minutes)
  - Window state (to suppress heating)
- Compute a desired valve opening percentage (0–100). If the device cannot accept a valve percentage, derive an equivalent setpoint reduction (flow_cap_K) so that the effective target becomes target - flow_cap.
- For Sonoff TRVZB provide recommendations for min_open% and max_open% caps.

Optional PID mode
- In addition to the heuristic mapping, a PID controller can be enabled per TRV.
- P/I act on the error e = target − current; the derivative term operates on the measurement (d_on_measurement) using a blended temperature input: a configurable mix of the TRV-internal temperature and the external room temperature.
- The derivative measurement is smoothed via an EMA with configurable alpha to reduce noise.
- Output u = P + I + D is clamped to 0..100% and passed through the same smoothing/hysteresis/rate-limit as the heuristic.
- Anti-windup: the integrator is clamped and does not continue to charge while the output is saturated.
- Near-setpoint integrator relief: when the error changes sign within the near band, the I-term is deliberately decayed slightly to speed up re-opening/re-closing around ±0.1 K.
- Conservative auto-tuning optionally adapts kp/ki/kd over time with a minimum interval between changes.
  The blending parameter trend_mix_trv denotes the EXTERNAL temperature weight in [0..1] (1.0 = external only, 0.0 = internal only). Default: 0.7.

## Glossary and Symbols
- TRV: Thermostatic Radiator Valve
- ΔT: Temperature difference = setpoint − current temperature (Kelvin)
- slope: Temperature trend dT/dt in K/min (positive = rising, negative = falling)
- valve_percent p: Desired valve opening 0–100%
- flow_cap_K c: Setpoint reduction in Kelvin applied to emulate throttling on setpoint-only devices
- band_near, band_far: Inner/outer ΔT bands controlling fine vs. full action
- P, I, D: Proportional, integral, and derivative components of the PID controller
- meas_blend: Blended measurement for the derivative term (mix of TRV-internal and external temps)

## Per-room contract (inputs/outputs)
Inputs (available today in BT):
- current_temp_C (sensor)
- target_temp_C (BT target)
- window_open (binary)
- temp_slope_K_per_min (computed from recent readings, 5–10 min window)
- trv_temp_C (internal TRV temperature), used for PID D-term blending

Outputs (consumed by controlling/adapters):
- valve_percent (0–100) for devices with valve control
- setpoint_eff_C = target_temp_C − flow_cap_K for setpoint-only devices
- flow_cap_K (0..cap_max_K) for telemetry/tuning
- sonoff_min_open_pct, sonoff_max_open_pct for Sonoff TRVZB
- suggested_valve_percent: convenience attribute mirroring the recommended valve_percent for graphing/automations

State (lightweight, per room key):
- ema_slope (smoothed slope)
- last_percent (smoothed valve percentage)
- last_update_ts (rate limiting)
- PID state (if enabled): pid_integral, pid_last_meas (smoothed blended measurement), pid_last_time; learned pid_kp/ki/kd and last_tune_ts

## Algorithm (summary)
- If ΔT >= band_far → open up to the learned max_open% cap (defaults to 100% until learned)
- If ΔT <= -band_far → 0% (close/hold)
- Strong overshoot fast path: If ΔT ≤ −band_far, we bypass hysteresis and rate-limit and immediately drive the output to 0% to avoid sticking at small residual openings.
- Else map ΔT linearly to 0..max_cap and correct by slope (max_cap = learned max_open% for the current target bucket; defaults to 100% if not yet learned):
  - positive slope → reduce opening (prevent overshoot)
  - negative slope (while ΔT>0) → ensure at least a minimum opening
- Smooth via EMA, apply hysteresis and a minimum update interval to save battery and reduce traffic. The overshoot fast path above bypasses these guards. Clamping respects the learned max_cap.
- flow_cap_K = cap_max_K * (1 - valve_percent/100)
- For Sonoff: sonoff_max_open_pct = valve_percent; sonoff_min_open_pct ≈ 0–5% depending on overshoot.
- Phase-aware learning of caps:
  - Update max_open only in heating phase (ΔT ≥ band_near).
  - Update min_open only in holding/cooling phases (ΔT ≤ band_near).
  - Ensure min_open ≤ max_open. Coarse 5% steps far from setpoint, 1% steps near setpoint.

## Algorithm (details)
1) Base mapping from ΔT to p (0..100):
  - Clamp ΔT to [−band_far, +band_far]
  - Linear map: 0% at −band_far, 100% at +band_far
  - Formula: $p_{base} = 100 \cdot \frac{\Delta T + band\_far}{2\, band\_far}$

2) Slope smoothing (EMA):
  - $s_{ema} \leftarrow 0.6\, s_{ema} + 0.4\, s$ (on new slope s)

3) Slope correction on p:
  - $p_{adj} = p_{base} + G \cdot s_{ema}$ with $G<0$ (positive slope reduces opening)

4) Near-setpoint refinement (|ΔT| ≤ band_near):
  - If $s_{ema} ≥ s\_{up}$: $p_{adj} \leftarrow p_{adj} \cdot 0.7$ (throttle)
  - If $s_{ema} ≤ s\_{down}$ and ΔT>0: $p_{adj} \leftarrow \max(p_{adj}, 60)$ (ensure opening)

5) Full-demand/overshoot guards:
  - If ΔT ≥ band_far → p = max_cap (max_cap = learned max_open%; defaults to 100% until learned)
  - If ΔT ≤ −band_far → p = 0 (and trigger fast-close path below)
  - Else p = clamp(p_adj, 0..max_cap)

6) Percentage smoothing (EMA) + hysteresis + rate limit:
  - $p_{smooth} = (1-\alpha)\, p_{last} + \alpha\, p$
  - Apply only if |p_smooth − p_last| ≥ hysteresis and Δt ≥ min_update_interval
  - Overshoot fast-close: When ΔT ≤ −band_far, bypass the rate limit and hysteresis, and set $p_{smooth} \leftarrow 0$ to close immediately.
  - Final output is clamped to [0, max_cap].

7) Setpoint throttling (generic devices):
  - $c = cap\_max \cdot (1 - p/100)$
  - $setpoint\_{eff} = setpoint - c$ (apply only on overshoot/at target: ΔT ≤ 0. With heating demand (ΔT > 0) no reduction to avoid fighting the TRV.)
  - Adapters that expose direct valve control use the valve_percent output instead. For devices without valve control, setpoint throttling can be used when calibration support is enabled; with "No Calibration" active, BT refrains from sending setpoint changes and setpoint_eff is provided for telemetry only.

8) Sonoff min/max recommendations (and learning):
  - $max\_open \%= p$ (recommendation)
  - $min\_open \% \approx 0$ on overshoot (ΔT ≤ −band\_near), otherwise a small comfort value (e.g., 5%)
  - Learning per target temperature bucket is phase-dependent:
    - max_open is only updated when heating (ΔT ≥ band\_near)
    - min_open is only updated when holding/cooling (ΔT ≤ band\_near)
    - Values are approached in reasonable steps (5% coarse, 1% fine) and clamped (0..100, min ≤ max)

All steps are per-room and require no global information.

### PID mode (details)
1) Error and timing
  - Error: $e = T_{set} - T_{cur}$ (Kelvin). Time step dt is computed from a monotonic clock.

2) P and I
  - $P = k_p \cdot e$
  - $I \leftarrow I + k_i \cdot e \cdot dt$ with anti-windup clamping to configurable bounds (in percent points equivalent). When the error flips sign inside the near band, the stored I-term is decayed to enable an earlier reaction.

3) D on measurement with blended input and smoothing
  - Blended measurement: $m = \alpha_{ext} \cdot T_{ext} + (1-\alpha_{ext}) \cdot T_{trv}$ with $\alpha_{ext} \in [0,1]$ (trend_mix_trv = external weight)
  - Smoothed: $m_s \leftarrow (1-\beta)\, m_s + \beta\, m$ with 0≤β≤1 (d_smoothing_alpha)
  - Derivative: $D = -k_d \cdot \frac{dm_s}{dt}$ (negative sign because derivative is on measurement)

4) Output and clamps
  - $u = P + I + D$, then clamp to 0..100 and pass through the same EMA/hysteresis/rate-limit used for the heuristic.

5) Conservative auto-tuning (optional)
  - Overshoot (sign change with amplitude > threshold `overshoot_threshold_K`, default 0.2 K): decrease k_p slightly, increase k_d slightly
  - Sluggish (ΔT ≫ band_near and |slope| small): increase k_i slightly
  - Steady state (|ΔT| < band and u small): decrease k_i slightly to avoid drift
  - Enforce min/max bounds, and a minimum interval (e.g., 30 min) between tune steps.

## Implementation
- balance.py: pure computation, no side effects.
- events/temperature.py: compute and store a smoothed slope (K/min).
- events/trv.py: integrates balance to compute per-TRV recommendations and learns Sonoff/TRV `min_open%` and `max_open%` per target temperature bucket depending on the phase (max only during heating up, min during holding/cooling). For devices without valve control, `setpoint_eff` can be used (if calibration/setpoint writing is allowed); with active "No Calibration", no setpoints are written. Debug information per TRV (including PID) is available under `real_trvs[trv]['balance']`. On initial setup, reasonable defaults are used (min ≈ 5%, max = 100%) if no suggestion is available for the current phase.
- events/temperature.py: additionally writes the external temperature used by BT (without hysteresis) to devices that have an "external_temperature_input" (e.g., Sonoff TRVZB), so that the device consistently sees the same reference.
- climate.py: exposes learned caps as a JSON attribute `trv_open_caps`. Structure per TRV:
  `{ "current_bucket": "XX.X", "buckets": { "XX.X": { "min_open_pct": N, "max_open_pct": M, "suggested_min_open_pct"?: n, "suggested_max_open_pct"?: m, "stats"?: { "samples": k, "avg_slope_K_min": s, "avg_valve_percent": p, "avg_delta_T_K": d, "last_update_ts": iso } }, ... } }`. Persisted across restarts.
- Additionally, climate schedules an external temperature keepalive every ~30 minutes (plus an immediate send after startup) for devices requiring periodic refresh.
- utils/controlling.py: if a valve entity exists (e.g., MQTT/Z2M), it forwards the computed percentage without an extra local hysteresis. Hysteresis/rate-limit is enforced centrally by the balance module.
- model_quirks/TRVZB.py: direct access to Sonoff entities `number.*.valve_opening_degree` and `number.*.valve_closing_degree` (with closing=100−opening), as well as `number.*.external_temperature_input` (0..99.9°C, 0.1 steps), to set the valve and external temperature without automation detours.

### Code integration points
- `balance.py`: pure math and simple per-room in-memory state
- `events/temperature.py`: computes `temp_slope` as a smoothed K/min trend
- `events/trv.py`: calls `compute_balance(...)`, records debug and updates learned caps per target temperature bucket; does not modify setpoints
- `climate.py`: aggregates and exposes `trv_open_caps` for HA automations to write to TRV fields
- `utils/controlling.py`: if `valve_position_entity` exists for a TRV, sends `set_valve(percent)` with a 3% hysteresis (optional)

## Configuration
- No dedicated calibration mode is required. Feature works alongside `no_calibration` (setpoints are left untouched).
- Advanced per-TRV options (setup and options flow):
  - balance_mode: heuristic | pid
  - pid_auto_tune: enable/disable auto-tuning (default: true)
  - pid gains: pid_kp, pid_ki, pid_kd
  - trend_mix_trv (0..1): Anteil der EXTERNEN Temperatur für den D‑Term (1.0 = nur extern, 0.0 = nur intern; Standard 0.7)
  - d_smoothing_alpha (0..1): EMA-Glättung nur für den D-Kanal
  - percent_hysteresis_pts: Hysterese am Aktuator in %-Punkten (Default: 0.5)
  - min_update_interval_s: Mindestabstand zwischen Stellgrößen-Updates (Default: 60s)
  - Auto‑Tuning Schwellen (PID):
    - overshoot_threshold_K: Overshoot-Erkennung in Kelvin (Default: 0.2)
    - sluggish_slope_threshold_K_min: Schwelle für „zu träge“ in K/min (Default: 0.005)
    - steady_state_band_K: Band für quasi-stationären Zustand in Kelvin (Default: 0.1)
    - tune_min_interval_s: Mindestabstand zwischen Tuning-Schritten (Default: 1800 s)
- Future tuning parameters (cap_max, bands, hysteresis) can be exposed if needed.

Suggested defaults (current):
- band_near_K = 0.1
- band_far_K = 0.3
- cap_max_K = 0.8
- slope_up_K_per_min = +0.02
- slope_down_K_per_min = −0.01
- slope_gain_per_K_per_min = −1000.0 (e.g., +0.02 K/min → −20 pp)
- percent_smoothing_alpha = 0.35
- percent_hysteresis_pts = 0.5
- min_update_interval_s = 60
- PID: kp = 100.0, ki = 0.03, kd = 2000.0, integrator clamp ±60
- Auto‑Tuning: overshoot_threshold_K = 0.2, sluggish_slope_threshold_K_min = 0.005, steady_state_band_K = 0.1, tune_min_interval_s = 1800

## Notes
- Works with existing BT signals only (current temperature, setpoint, window state). No valve feedback or global supply info is required.
- Emergent behavior: the weakest room tends to run with valve_percent ~ 100%, others get throttled near their setpoint.
- For Sonoff TRVZB, we will extend adapters to write min/max opening if available via the underlying integration.

## Next steps
- Extend adapters/blueprints to detect and write Sonoff min/max opening endpoints where available, consuming `trv_open_caps`.
- Optional: expose balance parameters in the UI.

## Edge cases and mitigations
- Very slow systems (e.g., floor heating): increase band_near, reduce slope_up (less aggressive throttling), and/or lower slope_gain magnitude.
- Many rooms heat simultaneously: local logic still works; emergent behavior limits strong rooms near setpoint. If desired later, add a mild global coordinator.
- Noisy sensors: EMA smoothing on slope and hysteresis on outputs reduce flapping.
- Battery/traffic: min update interval and percent hysteresis avoid frequent writes.
- Devices without OFF mode: balance only affects setpoint; existing “min temp as OFF” safeguards remain.
- Window open: heating is suppressed; balance logic is skipped.

## Telemetry and debugging
- Climate attributes:
  - `temp_slope_K_min`: current smoothed slope in K/min
  - `balance` (JSON per TRV): `{ "valve%": p, "flow_capK": c }`
  - `trv_open_caps` (JSON): per TRV `{ "current_bucket": "XX.X", "buckets": { ... } }`
  - `suggested_valve_percent`: flaches Attribut mit der aktuellen Stellgröße in % (bequem für Graphen/Automationen)
  - PID (bei aktivem PID‑Modus) – pro TRV im gespeicherten Debug unter `real_trvs[trv]['balance']['debug']['pid']`:
    - `dt_s`, `e_K`, `p`, `i`, `d`, `u`, `kp`, `ki`, `kd`
    - Messwerte & Mix: `meas_external_C`, `meas_trv_C`, `mix_w_external`, `mix_w_internal`, `meas_blend_C`, `meas_smooth_C`
    - Ableitung: `d_meas_per_s` (in K/s)
    - Schutz/Komfort: `anti_windup_blocked` (Integrator gestoppt bei Sättigung), `i_relief` (Integrator nah am Soll bewusst entlastet)
  - Zusätzlich (außerhalb von `pid`): `delta_T`, `slope_ema` (K/min)
- Per-TRV stored debug under `real_trvs[trv]['balance']` includes Sonoff min/max.

## Testing strategy
1) Unit-like tests in a dev instance:
  - Synthetic temperature ramps verifying percent and setpoint_eff reactions.
  - Overshoot scenarios: ensure throttling near setpoint.
2) Integration tests with MQTT/Z2M devices:
  - Detect `valve_position_entity` and validate `set_valve` hysteresis.
3) Field validation:
  - Observe average |ΔT| and overshoot frequency over multiple days.
  - Check frequency of setpoint/valve writes to confirm low traffic.

## Limitations
- Without real flow/pressure feedback, percent ↔ delivered power is heuristic.
- Interactions with TRV internal PID vary by model; throttling is designed to act mainly near setpoint to minimize conflicts.

## Future enhancements
- Expose tuning parameters in Advanced options per TRV.
- Adapter support for Sonoff TRVZB min/max open writing (Z2M/ZHA specifics).
- Optional lightweight global coordinator (e.g., “weakest wins” raising caps, soft budgets).
- Diagnostics panel showing trends and balance actions over time.

## FAQ
Q: Does this replace mechanical hydraulic balancing?
A: No. It emulates a dynamic throttling near setpoint to reduce overshoot and undue dominance by strong rooms. It improves comfort and potentially efficiency, but does not replace proper mechanical balancing.

Q: What happens when I change the boiler supply temperature?
A: The measured slopes decrease; the algorithm gradually reduces flow_cap (i.e., increases effective opening) until stability near setpoint is restored.

Q: Do I need valve feedback?
A: No. It’s optional. With valve feedback (via MQTT/Z2M), the algorithm can set a valve percentage; otherwise it uses setpoint throttling.

## Maintenance
This document is a living specification. Whenever we modify the hydraulic balance logic, configuration, adapters, or telemetry, update this file in the same change. Treat it as the single source of truth for the feature’s behavior, parameters, and integration points.

### Persistence
Both the per-room balance learning state (EMA of slope and last percent/rate-limit timestamp) and the learned min/max open caps per TRV and target bucket are persisted across Home Assistant restarts using HA storage. Keys are scoped by the Better Thermostat entity `unique_id` and the TRV entity id (format: `<unique_id>:<trv_entity_id>[:<bucket>]`). State is loaded during entity startup and saved in a debounced manner after updates to avoid re-learning after restarts.
