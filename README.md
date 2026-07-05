# ECO-PULSE — Green Radar-Based Flash-Flood Early Warning

AESH Sustainability Hackathon 2026 — Challenge 2: Green Radar Systems for Environmental Monitoring
Onsite Final, Egypt, 6–7 July 2026

> 

---

## 1. What this project is

ECO-PULSE is a radar-based **wadi flash-flood monitoring and early-warning system**. A ground-based FMCW/pulse
radar (24 GHz) points at a wadi (dry riverbed) channel and tracks water level and rate-of-rise in real time.
The system compares a **conventional, fixed-duty-cycle radar mode** against a **green, adaptive-duty-cycle mode**
that only transmits at full power/rate when risk is elevated, cutting energy use while keeping detection
performance and warning accuracy intact.

Radar is the right sensor here because it works in the dark, through dust/haze, and in the harsh, unstaffed,
solar-powered environments where flash floods actually occur — where optical or ultrasonic level sensors are
harder to keep clean, powered, and reliable.

## 2. Repository structure

```
eco_pulse_V8_MERGED/
├── get_params.m                    # Master parameter set (waveform, scenario, thresholds, energy model)
├── config_default.json             # External JSON mirror of get_params defaults (loaded via config_loader.m)
├── config_loader.m
├── signal_generation.m             # TX waveform: Zadoff-Chu (Mode 1) / LFM chirp (Mode 2), freq agility, windows
├── environment_model.m             # Propagation, sea/wave clutter, multipath, rain attenuation, AWGN/colored noise
├── signal_processing.m             # Matched filter, CFAR, MTI, MUSIC/ESPRIT
├── cfar_detector.m                 # CA-CFAR and Two-Stage adaptive CFAR
├── feature_extraction.m            # Level, rate, SNR, Doppler, peak-width features
├── kalman_tracker.m                # Constant-velocity Kalman filter (optional Doppler fusion)
├── arima_predictor.m / pro_lstm_forecaster_v2.m   # Forecasters (ARIMA baseline, optional LSTM)
├── decision_logic.m / decision_logic_pro.m        # Threshold + ML + forecast-driven state machine
├── adaptive_mode_select.m          # Energy manager: chooses Mode 1 (calm) vs Mode 2 (track) duty cycle
├── pro_solar_manager.m             # Solar harvest + battery bookkeeping
├── main_flood_monitoring_system.m / main_flood_monitoring_system_core.m   # Entry points
├── compare_baseline_vs_green.m     # **Baseline vs green KPI comparison (the core Challenge-2 evidence)**
├── run_full_validation.m           # Full validation suite: 5-scenario sweep, repeatability, tradeoff curve, stress test
├── run_monte_carlo.m               # Pd/Pfa Monte Carlo
├── dashboard_philip.m / flood_gui.m / flood_gui_ultimate.m   # Visualization / interactive GUI
├── KNOWN_LIMITATIONS.md            # Disclosed, measured limitations (see Section 6 below)
├── CHANGELOG_7_PAPERS.md           # Detailed log of every literature-derived enhancement, as actually tested
└── pro_*.m                         # Optional add-on modules (sensor fusion, XAI, alerting, AR export, etc.)
```

## 3. Setup

**Requirements:** MATLAB R2020a+ (uses `uifigure`; falls back gracefully without it) **or** GNU Octave 6+.
No paid toolboxes are required — ARIMA, LSTM, and CFAR are implemented from scratch so the project runs on
base MATLAB/Octave.

```bash
git clone [REPO_URL]
cd eco_pulse_V8_MERGED
```

Open MATLAB/Octave in this folder (or run `addpath` on it) before calling any function below.

## 4. Execution

```matlab
% One-line launcher (auto-detects GUI support)
RUN

% Or explicitly:
params  = get_params('slow_rise');      % scenarios: stable | slow_rise | fast_rise | receding | flash_flood
results = main_flood_monitoring_system_core(params);
dashboard_philip(results, params);
animate_flood_3d(results, params);

% Baseline (fixed duty) vs Green (adaptive duty) comparison — THE headline demo
cmp = compare_baseline_vs_green('slow_rise', true);   % true = show plots

% Full validation suite (all 5 scenarios, repeatability, tradeoff curve, stress test)
report = run_full_validation();
```

## 5. Key parameters and assumptions

| Parameter | Default | Meaning |
|---|---|---|
| `f0` | 24 GHz | Carrier frequency |
| `fs` | 10 kHz (baseband model) | Sample rate of the simulated baseband/decision pipeline |
| `pulse_repetition_interval` | 20 ms | PRI |
| `duty_cycle` | 0.05 | Nominal duty cycle, Mode 1 |
| Mode 1 waveform | Zadoff-Chu, length 127 | Low-power "calm" search/track waveform |
| Mode 2 waveform | LFM chirp, 20 MHz BW, 20 µs | Higher-power waveform used when risk is elevated |
| `energy.P_zc` / `energy.P_chirp` | 0.1 W / 0.3 W | Per-mode transmit power used by the energy model |
| `energy_calm_duty` | 0.6 | Duty cycle while calm (green mode) |
| `energy_low_battery_duty` | 0.5× | Further duty reduction below 20% battery |
| `levelAlert` / `levelFlood` | 2.5 m / 3.0 m | Decision thresholds |
| `cfar_type` | Two-Stage adaptive CFAR | Never allowed to report *more* false alarms than CA-CFAR on the same data (hard-capped, see `cfar_detector.m`) |
| `range_estimation_mode` | `ground_truth` (default) | See Section 6 — `signal_based` exists, is tested, and is disclosed as a limitation |

**Key assumptions:** single-target wadi cross-section, 24 GHz ground-based radar, solar/battery-powered
unattended node (30 W panel / 50 Wh battery), simulated wave/multipath/rain environment (Pierson-Moskowitz /
Elfouhaily wave spectra), AWGN or AR(1) colored noise channel.

## 6. Reproduction steps for judges

1. Run `report = run_full_validation();` — reproduces, in one command: the 5-scenario baseline-vs-green energy
   sweep, an 8-repeat statistical-stability check, an energy/quality tradeoff curve, and a 6-case stress test.
2. Run `compare_baseline_vs_green('slow_rise', true)` for the plotted baseline-vs-green comparison shown live.
3. Cross-check reported energy values against `KNOWN_LIMITATIONS.md`, which lists the exact validated
   per-scenario energy totals (bit-for-bit reconfirmed after the V8 fixes): stable = 0.483 J, slow_rise = 1.809 J,
   fast_rise = 2.075 J, receding = 1.271 J, flash_flood = 1.653 J (`ground_truth` mode, 160 s runs).
4. **Read `KNOWN_LIMITATIONS.md` before judging.** It documents, with root cause and measured evidence, that
   (a) the raw Mode-2 chirp waveform is under-sampled at the current `fs` (does not affect the validated
   decision-layer KPIs, which run on Kalman-filtered/derived quantities), and (b) naive range peak-picking is
   ambiguous under default multipath — both are disclosed rather than hidden, with a tested opt-in
   `signal_based` alternative for judges who want to see the real (harder) signal path.

## 7. AI / external resource disclosure

See `AI_External_Disclosure.docx` in the final submission package for the full list. In short: AI assistance was
used for code auditing, merging, and drafting this documentation; all quantitative results were produced by
actually running the MATLAB/Octave code, not estimated. Algorithms adapted from open literature include CA/OS/
Two-Stage CFAR, constant-velocity Kalman filtering, ARIMA(1,1,1) forecasting, an optional hand-rolled LSTM
forecaster, MUSIC/ESPRIT super-resolution, Zadoff-Chu/Barker/Frank waveform coding, and Pierson-Moskowitz/
Elfouhaily sea-surface spectra.
