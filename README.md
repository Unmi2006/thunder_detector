# ⚡ Thunder Detector Pro

> Real-Time Storm Intelligence System — Streamlit Edition

![Python](https://img.shields.io/badge/Python-3.11+-3776AB?style=flat&logo=python&logoColor=white)
![Streamlit](https://img.shields.io/badge/Streamlit-1.32+-FF4B4B?style=flat&logo=streamlit&logoColor=white)
![Lines](https://img.shields.io/badge/Lines-2847-0A1825?style=flat)
![Tabs](https://img.shields.io/badge/Tabs-8-F5C518?style=flat)
![Functions](https://img.shields.io/badge/Functions-32-00E676?style=flat)

---

## 📋 Table of Contents

1. [Overview](#overview)
2. [Quick Start](#quick-start)
3. [Requirements](#requirements)
4. [Architecture](#architecture)
5. [Feature Reference](#feature-reference)
6. [Alert Levels](#alert-levels)
7. [Auto-Simulation Scenarios](#auto-simulation-scenarios)
8. [Audio DSP Reference](#audio-dsp-reference)
9. [Severity Score Formula](#severity-score-formula)
10. [Optional API Integrations](#optional-api-integrations)
11. [Critical: Thread Safety](#critical-thread-safety)
12. [Known Limitations](#known-limitations)
13. [File Structure](#file-structure)
14. [Function Reference](#function-reference)

---

## Overview

Thunder Detector Pro is a fully-featured, real-time storm monitoring application built with Python and Streamlit. It listens to your microphone, applies digital signal processing to isolate thunder frequencies (20–120 Hz), detects strikes, estimates storm distance using flash-to-thunder timing, and presents all data through a rich 8-tab dashboard.

The app works entirely offline for detection and simulation. Optional integrations (Blitzortung, OpenWeatherMap, RainViewer) add live global data when internet is available.

**Key capabilities at a glance:**

- Real-time microphone detection with Butterworth bandpass filter + EMA noise floor suppression
- 5 scripted auto-simulation scenarios (approaching storm, passing storm, distant rumble, severe outbreak, random)
- 8-tab dashboard: Live Monitor · Analytics · Heatmap · AI Analysis · Event Log · Storm Map · Radar & 3D · Session
- Arrival time predictor (weighted least-squares regression), storm decay predictor (exponential fit)
- Browser-native Text-to-Speech announcer + Web Audio siren via JS injection
- Blitzortung live lightning feed with simulated fallback, RainViewer radar tiles, OpenWeatherMap weather
- Polar storm radar with animated sweep, 3D trajectory plot (time × distance × amplitude)
- Session save/load as JSON, PDF export via ReportLab, CSV export
- Browser geolocation auto-detect — defaults to West Bengal (22.5726°N, 88.3639°E)

---

## Quick Start

### 1. Install dependencies

```bash
pip install streamlit sounddevice numpy scipy plotly pandas reportlab
```

> `reportlab` is optional — only needed for PDF export.  
> `sounddevice` requires PortAudio. On Ubuntu/Debian: `sudo apt install portaudio19-dev`

### 2. Run

```bash
streamlit run app.py
```

Opens at `http://localhost:8501`.

### 3. First steps

- **With microphone:** Click **🎙 START MIC** in the header → press **🌩 LOG LIGHTNING FLASH** when you see lightning → the next thunder detection auto-calculates distance
- **Without microphone:** Press **▶ START AUTO SIM** in the sidebar → choose a scenario → watch the dashboard populate live
- Switch tabs to explore waveform, analytics, radar, 3D trajectory, and the storm map

---

## Requirements

| Package | Version | Purpose |
|---|---|---|
| `streamlit` | ≥ 1.32 | Web UI framework |
| `sounddevice` | ≥ 0.4.6 | Microphone capture |
| `numpy` | ≥ 1.24 | Array math, RMS computation |
| `scipy` | ≥ 1.11 | Butterworth filter, Welch PSD |
| `plotly` | ≥ 5.18 | All charts (waveform, gauge, 3D, polar, mapbox) |
| `pandas` | ≥ 2.0 | DataFrames for analytics and export |
| `reportlab` | ≥ 4.0 | PDF report generation *(optional)* |

---

## Architecture

### Audio Pipeline

```
Microphone
    │
    ▼
_AUDIO_QUEUE  ◄──── audio_callback() [C thread — NEVER touches session_state]
    │
    ▼  process_audio_block() — runs each Streamlit rerun
    │
    ├── Butterworth bandpass filter (20–120 Hz, order 5)
    ├── EMA noise floor suppression  (α = 0.92)
    ├── 20-sample rolling RMS smoother
    ├── Welch PSD spectral centroid
    │
    └── Threshold comparison ──► add_event()
                                      │
                                      ├── classify()          → DANGER/WARNING/WATCH/CLEAR
                                      ├── detect_storm_trend()→ APPROACHING/MOVING_AWAY/STATIONARY
                                      ├── compute_severity()  → 0-100 score
                                      ├── predict_arrival()   → ETA to 3 km
                                      ├── compute_storm_vector() → speed + bearing
                                      ├── predict_decay()     → storm end time
                                      ├── maybe_speak()       → TTS alert
                                      └── play_siren()        → Web Audio tone
```

### Auto-Simulation Engine

The auto-simulation engine drives the app when no microphone is available. It uses a scripted scenario system returning `(delay_s, km, amplitude)` step lists. The engine checks `sim_next_tick` against `time.time()` each rerun and fires the next step when due. The `random` scenario regenerates endlessly; others stop cleanly when complete.

```python
# Rerun loop (0.28s interval)
if auto_sim and time.time() >= sim_next_tick:
    delay, km, amp = sim_steps[sim_step]
    add_event(amp, km)
    build_waveform_spike(amp)
    sim_step += 1
    sim_next_tick = time.time() + sim_steps[sim_step][0]
```

### Session State Keys

| Key | Type | Purpose |
|---|---|---|
| `events` | `list[dict]` | All detected strikes, newest first |
| `rms_history` | `deque(400)` | Rolling RMS values for waveform |
| `severity_history` | `list[(ts, score)]` | Score over time for history chart |
| `storm_trend` | `str` | APPROACHING / MOVING_AWAY / STATIONARY / UNKNOWN |
| `decay_info` | `dict` | Result of `predict_decay()` — eta, half-life, rate |
| `auto_sim` | `bool` | Whether auto-simulation is running |
| `sim_steps` | `list[(delay, km, amp)]` | Current scenario step sequence |
| `sim_next_tick` | `float` | `time.time()` of next scheduled event |
| `radar_frame` | `int` | Frame counter for polar radar sweep animation |
| `noise_floor` | `float` | EMA-tracked ambient noise level |
| `auto_threshold` | `float` | Calibrated detection threshold |
| `user_lat` / `user_lon` | `float` | Browser-geolocated coordinates |

> ⚠️ `_AUDIO_QUEUE` is **module-level**, not in `session_state`. See [Thread Safety](#critical-thread-safety).

---

## Feature Reference

### 📡 Tab 1 — Live Monitor

| Feature | Details |
|---|---|
| **Audio Waveform** | Dual-layer chart: raw bandpass RMS (cyan) + 10-sample rolling average (amber dashed). Threshold line overlaid. |
| **Distance Gauge** | Plotly indicator gauge 0–40 km with delta arrow showing change vs previous strike. |
| **Spectral Centroid** | Secondary chart of thunder texture frequency (Hz) via Welch PSD, with RMS overlay on secondary Y-axis. |
| **Flash Timer** | Press LOG FLASH when lightning is seen → next thunder detection auto-calculates distance using Δt × 0.343 km/s. |

### 📊 Tab 2 — Analytics

| Feature | Details |
|---|---|
| **Strike Timeline** | Scatter plot: time vs distance, bubble size = amplitude, colour = alert level, rolling trend line. |
| **Amplitude Histogram** | Distribution of amplitudes by level, overlaid bars. |
| **Distance vs Amplitude** | Correlation scatter with OLS trendline (requires ≥ 3 events). |
| **Level Breakdown** | Donut chart of DANGER / WARNING / WATCH / CLEAR proportions. |
| **Severity Score History** | Line chart of 0–100 score over time with 5 colour-coded background zones. |
| **Storm Decay Predictor** | Exponential fit on 1-minute strike-rate buckets — estimates storm end time, half-life, and current rate. |

### 🗺 Tab 3 — Heatmap

| Feature | Details |
|---|---|
| **Intensity Heatmap** | Time × Distance Zone grid, cell colour = mean amplitude. 5 distance buckets: 0–3 / 3–8 / 8–15 / 15–25 / 25–40 km. |
| **Storm Path** | Line chart of distance over time with coloured markers and DANGER / WARNING zone bands. |

### 🤖 Tab 4 — AI Analysis

| Feature | Details |
|---|---|
| **Storm Summary** | Rule-based analysis covering proximity, trend, intensity, frequency, and danger count. |
| **Severity Gauge** | Plotly gauge 0–100 with MINIMAL / LOW / MODERATE / SEVERE / EXTREME colour zones. |
| **All-Clear Timer** | 30-minute countdown since last strike. Progress bar + Plotly gauge. Resets on new strike. |
| **Safety Recommendations** | 6 contextual safety tips in a two-column layout. |
| **PDF Export** | ReportLab A4 PDF with executive summary, AI analysis, and full strike table. |
| **CSV Export** | All events with time, amplitude, distance, level, spectral centroid. |

### 📋 Tab 5 — Event Log

| Feature | Details |
|---|---|
| **Filterable log** | Multi-select by alert level. Sort by time, amplitude, or distance. |
| **Styled table** | Level column colour-coded. Monospace font. Row count shown. |

### 🌍 Tab 6 — Storm Map

| Feature | Details |
|---|---|
| **Browser Geolocation** | Auto-detects coordinates via `navigator.geolocation`. West Bengal default. Manual override fields + Re-detect button. |
| **Blitzortung Live Data** | Real lightning strikes from public feed. Age-coloured dots (bright = fresh). Falls back to simulation. Cached 30s. |
| **Range Rings** | 4 concentric circles at 3 / 8 / 20 / 40 km, colour-coded by alert level. |
| **Your Detections** | Star markers at correct distance radius (bearing randomised — single mic can't determine direction). |
| **Storm Vector Compass** | Plotly polar chart with animated bearing arrow + speed. Toward / away status. |
| **Weather Overlay** | OpenWeatherMap current conditions: temp, humidity, wind, pressure, visibility. Cached 5 min. |
| **Proximity Table** | Top 15 nearest strikes sorted by distance, colour-coded by alert level. |

### 🌀 Tab 7 — Radar & 3D

| Feature | Details |
|---|---|
| **Polar Storm Radar** | Animated cyan sweep line with 7-step afterglow trail. Strike dots fade over 5 min. 4 range rings. Animates each rerun. |
| **3D Storm Trajectory** | `Scatter3d`: X = time (s), Y = distance (km), Z = amplitude. Red danger plane at 3 km. Floor projection shadow. Draggable/rotatable. |
| **Rain Radar** | RainViewer free tile API — last 6 frames (~30 min). Plotly Mapbox overlay with frame slider. No API key needed. |

### 💾 Tab 8 — Session

| Feature | Details |
|---|---|
| **Save Session** | Download full session as timestamped `.json` (all events, heatmap data, peak RMS, session start time). |
| **Load Session** | Upload any `.json` file to restore a previous session — recalculates trend, severity, AI summary automatically. |
| **Session Summary** | Duration, total events, closest strike, peak amplitude, severity score. |
| **Session Timeline** | 2-panel chart: distance line + amplitude bar, shared X-axis. |

---

## Alert Levels

| Level | Distance | Colour | Action |
|---|---|---|---|
| 🔴 **DANGER** | ≤ 3 km | Red `#FF2D55` | Seek shelter immediately |
| 🟠 **WARNING** | 3 – 8 km | Orange `#FF8C00` | Move indoors now |
| 🟡 **WATCH** | 8 – 20 km | Amber `#F5C518` | Monitor conditions closely |
| 🟢 **CLEAR** | > 20 km | Green `#00E676` | Storm distant or passing |
| ⚫ **UNKNOWN** | — | Grey `#3A6070` | Distance not yet calculated |

---

## Auto-Simulation Scenarios

| Scenario | Steps | Description |
|---|---|---|
| `random` | 20 (loops) | Organic wandering storm. Biased inward early, then random. Regenerates endlessly. |
| `approaching storm` | 16 | Starts at 38 km, closes to 1.8 km peak, then retreats to 35 km. |
| `passing storm` | 12 | Enters at 12 km, peaks at 2.2 km, exits to 23 km. |
| `distant rumble` | 8 | Stays 33–40 km throughout. Low amplitude (0.11–0.17). |
| `severe outbreak` | 33 | Three storm cells in sequence, each approaching and receding. |

Speed multiplier options: `0.5×` · `1×` · `2×` · `4×`

---

## Audio DSP Reference

| Constant | Value | Purpose |
|---|---|---|
| `SAMPLE_RATE` | 44100 Hz | Standard audio sample rate |
| `BLOCK_SIZE` | 2048 samples | ~46 ms per audio block |
| `THUNDER_LOW_HZ` | 20 Hz | Lower bound of thunder frequency band |
| `THUNDER_HIGH_HZ` | 120 Hz | Upper bound of thunder frequency band |
| `SPEED_SOUND_KMS` | 0.343 km/s | Speed of sound at 20°C |
| `FLASH_WINDOW_SEC` | 30 s | Max valid flash-to-thunder window |
| `RMS_HISTORY_LEN` | 400 samples | Waveform rolling buffer length |
| `NOISE_SMOOTHING` | 0.92 (EMA α) | Noise floor adaptation rate |
| `ALL_CLEAR_MINUTES` | 30 min | Post-storm safety wait time |
| `REFRESH_INTERVAL` | 0.28 s | Streamlit rerun rate during detection/simulation |
| `CALIBRATION_SECS` | 5 s | Auto-calibration recording duration |

### Filter design

```python
# 5th-order Butterworth bandpass, 20–120 Hz
b, a = butter(5, [20/22050, 120/22050], btype='band')
filtered = lfilter(b, a, samples.astype(np.float32))

# EMA noise floor
noise_floor = 0.92 * noise_floor + 0.08 * raw_rms

# Noise-suppressed RMS
rms = max(0.0, raw_rms - noise_floor * 0.5)
```

---

## Severity Score Formula

The 0–100 severity score is a weighted composite of four sub-scores:

| Sub-score | Max | Formula |
|---|---|---|
| Proximity | 40 pts | 40 if d ≤ 1 km · 35 if ≤ 3 · 28 if ≤ 8 · 18 if ≤ 20 · 8 otherwise |
| Intensity | 30 pts | `min(30, peak_amplitude × 32)` |
| Frequency | 20 pts | `min(20, strikes_per_min × 5)` |
| Trend bonus | ±10 pts | +10 APPROACHING · −5 MOVING_AWAY · 0 STATIONARY |

**Score bands:**

| Score | Label | Colour |
|---|---|---|
| 80–100 | 🔴 EXTREME | `#FF2D55` |
| 60–79 | 🟠 SEVERE | `#FF5500` |
| 40–59 | 🟡 MODERATE | `#FF8C00` |
| 20–39 | 🟡 LOW | `#F5C518` |
| 0–19 | 🟢 MINIMAL | `#00E676` |

---

## Optional API Integrations

### Blitzortung (no key required)

Public lightning strike feed at `data.blitzortung.org`. Returns recent global strikes with lat/lon/timestamp. Cached 30 seconds. Falls back to simulated data automatically if unreachable.

```python
BLITZORTUNG_URLS = [
    "https://data.blitzortung.org/Data/Protected/Strokes/latest.json",
    "https://map.blitzortung.org/GEO/strikes/latest.json",
]
```

### OpenWeatherMap (free API key required)

Get a free key at [openweathermap.org](https://openweathermap.org/api). Enter it in the sidebar under **WEATHER API**. Fetches current conditions: temperature, humidity, wind speed/direction, pressure, visibility, cloud cover. Cached 5 minutes.

### RainViewer (no key required)

```python
RAINVIEWER_API = "https://api.rainviewer.com/public/weather-maps.json"
```

Returns radar tile URLs for the last 6 frames (~30 minutes). Rendered as a Plotly Mapbox raster layer. Cached 2 minutes.

---

## Critical: Thread Safety

`sounddevice` fires `audio_callback()` from a **native C thread**. This thread cannot access `st.session_state` — doing so raises `AttributeError`. The architecture enforces strict separation:

```
✅ CORRECT                          ❌ WRONG
─────────────────────────────────   ─────────────────────────────────
def audio_callback(indata, ...):    def audio_callback(indata, ...):
    _AUDIO_QUEUE.put(indata.copy())     st.session_state.x = ...  # CRASH
```

**Rules:**
- `audio_callback()` writes **only** to `_AUDIO_QUEUE` (module-level `queue.Queue`)
- `process_audio_block()` runs in the main Streamlit thread and reads from `_AUDIO_QUEUE`
- All `st.session_state` access happens only in `process_audio_block()` and `add_event()`
- `_AUDIO_QUEUE` is defined at module level, not inside any function or class

---

## Known Limitations

- **Storm bearing** is estimated from amplitude heuristics. True directional detection requires a microphone array with time-difference-of-arrival processing.
- **Blitzortung** public feed availability varies. The app always falls back to simulated data cleanly with a clear status badge.
- **Browser Geolocation** requires HTTPS in production deployments. Works on `localhost` without HTTPS.
- **Web Speech API** and **Web Audio API** require a modern browser — Chrome 90+, Edge 90+. Firefox has partial support for speech synthesis.
- **PDF export** requires `reportlab`. If not installed, CSV export is always available as a fallback.
- **RainViewer Mapbox** overlay requires internet. Offline mode shows a graceful info message.
- **Arrival time predictor** requires at least 3 events with known distances. Accuracy improves with more data points.
- **Storm decay predictor** requires at least 5 events and detectable rate decay. Shows "INTENSIFYING" if rate is growing.

---

## File Structure

```
app.py                              Main Streamlit application (2,847 lines)
requirements.txt                    Python package dependencies
README.md                           This file

Generated at runtime:
  storm_YYYYMMDD_HHMMSS.json        Saved session files (Tab 8 → Save)
  thunder_YYYYMMDD_HHMMSS.csv       Exported event logs (Tab 4 → CSV)
  thunder_report_YYYYMMDD.pdf       PDF storm reports (Tab 4 → PDF)
```

---

## Function Reference

| Function | Purpose |
|---|---|
| `init_state()` | Initialise all `session_state` keys with safe defaults |
| `butter_bandpass()` | Design order-5 Butterworth bandpass filter coefficients |
| `bandpass_filter()` | Apply IIR filter via `scipy.signal.lfilter` |
| `compute_spectral_centroid()` | Welch PSD centroid in thunder band (Hz) |
| `compute_distance()` | Flash-to-thunder Δt × 0.343 km/s |
| `classify()` | Map distance km → DANGER / WARNING / WATCH / CLEAR / UNKNOWN |
| `level_color()` | Map level string → hex colour code |
| `detect_storm_trend()` | Compute APPROACHING / MOVING_AWAY / STATIONARY from recent events |
| `compute_severity()` | 0–100 composite severity score |
| `severity_label()` | Map score → (label, colour, emoji) |
| `predict_arrival()` | Weighted least-squares ETA to 3 km danger zone |
| `compute_storm_vector()` | Speed + pseudo-bearing from distance-over-time slope |
| `all_clear_status()` | 30-minute countdown and progress since last event |
| `fetch_weather()` | OpenWeatherMap current conditions — cached 5 min |
| `tts_speak()` | Inject browser Web Speech API utterance via JS component |
| `maybe_speak()` | De-duplicated TTS — fires only on alert level change |
| `predict_decay()` | Exponential decay fit on 1-min strike-rate buckets |
| `session_to_json()` | Serialise full session to JSON string |
| `session_from_json()` | Restore session from JSON, recompute all derived state |
| `play_siren()` | Web Audio API tone patterns per alert level |
| `fetch_rainviewer_frames()` | RainViewer radar tile URL list — cached 2 min |
| `fetch_blitzortung()` | Real lightning feed from blitzortung.org — cached 30s |
| `generate_simulated_strikes()` | Exponential-distribution fake strikes around centre point |
| `build_storm_map()` | Plotly geo figure: strikes + range rings + user position |
| `audio_callback()` | sounddevice callback — writes to `_AUDIO_QUEUE` only |
| `process_audio_block()` | Drain queue, filter, detect threshold, update state |
| `add_event()` | Record strike event, update all derived state and alerts |
| `generate_pdf_report()` | ReportLab A4 PDF with session data |
| `build_scenario()` | Generate scripted `(delay_s, km, amp)` step list by scenario name |
| `build_waveform_spike()` | Push thunder amplitude envelope into RMS history |
| `push_ambient_noise()` | Push low-level noise tick into RMS history between strikes |
| `generate_ai_summary()` | Rule-based storm intelligence text: proximity, trend, intensity, frequency |

---

*Thunder Detector Pro · Built with Streamlit + Python · West Bengal, India*
