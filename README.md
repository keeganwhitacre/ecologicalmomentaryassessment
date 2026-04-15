# ePAT: Ecological Phase Assessment Task

A browser-based, mobile-first implementation of the Phase Adjustment Task (PAT) for measuring interoceptive accuracy in ecological settings, paired with an Ecological Momentary Assessment (EMA) protocol for longitudinal affect and body awareness tracking.

## Overview

The Ecological Phase Assessment Task (ePAT) moves cardiac timing research out of the laboratory and into the participant's natural environment. By utilizing standard smartphone hardware, ePAT measures a participant's ability to perceive the timing of an internal physiological signal without the need for proprietary software or external sensors.

Unlike traditional heartbeat counting tasks — which are confounded by prior knowledge of heart rate, estimation heuristics, and time-counting strategies — the PAT is a psychophysical measure that requires participants to judge the synchronicity between their pulse and an external auditory tone. The phase adjustment approach yields a continuous dependent variable (the millisecond-level offset between heartbeat and stimulus) rather than a binary correct/incorrect count, providing richer data on interoceptive precision.

ePAT extends this paradigm by integrating the PAT with a longitudinal EMA protocol, enabling researchers to examine how interoceptive accuracy relates to daily fluctuations in affect, body awareness, and contextual factors — and critically, to test whether interoceptive accuracy functions as a stable trait or a fluctuating state.

## Application Structure

The application is a single HTML file (`index.html`) with all session types routed via URL parameters. A researcher-facing validation tool (`validation.html`) is maintained separately.

| File | Purpose |
|---|---|
| `index.html` | The complete task application. Routes to onboarding, PAT, EMA, or paired sessions via `?session=` parameter. |
| `validation.html` | Researcher-facing tool for validating the browser PPG signal against external hardware (e.g., CNAP, ECG). Records beat timestamps, sync markers, and audio tone events for offline Bland-Altman analysis. |
| `epat-core.js` | Shared signal processing modules: `BeatDetector`, `WabpDetector`, `AudioEngine`, `MotionDetector`, `WakeLockCtrl`. |
| `manifest.json` / `sw.js` | PWA configuration for home-screen installation. |

### Session Types & URL Parameters

All sessions share a single entry point. The `?session=` parameter routes the participant to the appropriate flow:

| Parameter | Value | Flow |
|---|---|---|
| `session` | `onboarding` | Full first-session enrollment: consent, scheduling, device check, task training. Uploads onboarding data, then transitions directly to first PAT session. |
| `session` | `pat` | PAT only: audio check, sensor calibration, 2 practice + 20 experimental trials, EMA confidence ratings. |
| `session` | `emamorning` | Morning EMA check-in (sleep + common affect items). |
| `session` | `emaafternoon` | Afternoon EMA check-in (common affect items only). |
| `session` | `emaevening` | Evening EMA check-in (common affect items only). |
| `session` | `pair_morning` | Paired session: EMA + PAT in the same visit (morning slot). |
| `session` | `pair_afternoon` | Paired session: EMA + PAT (afternoon slot). |
| `session` | `pair_evening` | Paired session: EMA + PAT (evening slot). |
| `id` | Any string | Pre-fills and hides the participant ID field. |
| `day` | Integer | Displays a "Day N" label on the start screen for longitudinal tracking. |
| `cb` | `pat_first` or `ema_first` | Counterbalance order for paired sessions (default: `ema_first`). |

**Example URLs:**
```
index.html?session=onboarding&id=42
index.html?session=pair_morning&id=42&day=4&cb=pat_first
index.html?session=emaevening&id=42&day=7
```

## Onboarding Session

`?session=onboarding` runs participants through a five-screen enrollment flow before their first PAT session:

1. **Informed Consent** — Scrollable consent document. Checkbox and initials field are locked until the participant scrolls to the bottom. Consent metadata (initials, timestamp) is included in the onboarding upload.
2. **Scheduling** — Participant selects available days and sets preferred morning/afternoon/evening EMA time windows.
3. **Device Check** — Automated checks for camera access, torch availability, audio output, and PPG signal quality. The PPG check runs a live camera preview so participants can confirm correct finger placement before proceeding.
4. **Task Training** — A simulated rhythm trains participants on the dial mechanics. Two sets of scrolling lines (grey = rhythm markers, coral = tone markers) scroll left at the same rate. The participant turns the dial to shift the coral lines onto the grey lines. A soft gate requires 2 seconds of maintained alignment before Continue unlocks — confirming understanding without punishing slow learners. No real cardiac data is used here.
5. **Setup Complete** — Summary screen. Onboarding data uploads, then participant proceeds directly to their first PAT session.

The cover story presented throughout onboarding and the task describes the study as a timing and affect study. Internal body signal perception is not named or foregrounded until after the task is complete, to avoid demand characteristics.

## UI Design

ePAT uses a purpose-built OLED dark theme optimized for iPhone screens (the primary deployment target):

- **Palette**: Pure black background (`#000000`) with layered surface greys, warm coral accent (`#e8716a`), and system-native font stack (`-apple-system`, SF Pro Display).
- **EMA layout**: One question at a time with animated transitions. Progress is shown as a thin fill bar (no item count visible, to reduce perceived length).
- **PAT trial screen**: Full-screen rotary dial with neumorphic styling and tick ring. Sensor warning overlays with live camera preview on finger loss or low SQI.
- **End screen**: Beat count message ("N beats contributed to science.") computed from baseline + trial data. Suppressed for EMA-only sessions.
- **Attrition mitigations**: Pre-filled participant ID (via `?id=`), day label (via `?day=`), hidden ID field for returning participants, and beat count reciprocity message.

## Technical Implementation

### PPG Signal Processing

The heart rate detection pipeline operates on frames from the device's rear camera with the torch (flashlight) active, creating a photoplethysmography (PPG) signal from light transmitted through the fingertip.

1. **Red channel extraction.** Each frame is drawn to a 40×40 canvas. The mean red channel intensity is computed across all pixels. Red dominance and overall brightness are used to determine whether a finger is present (with an 8-frame debounce to avoid flicker).

2. **Bandpass filtering.** The raw red channel signal passes through a custom IIR bandpass filter (high-pass at 0.67 Hz, low-pass at 3.33 Hz) to isolate the pulsatile cardiac component and reject DC drift and high-frequency noise. This passband corresponds to approximately 40–200 BPM. Filter coefficients are recomputed dynamically based on the actual camera framerate.

3. **WABP onset detection.** Beat detection uses a JavaScript port of the WABP algorithm (Zong, Heldt, Moody & Mark, 2003; *Computers in Cardiology* 30:259–262), adapted from the [PhysioNet reference implementation](http://www.physionet.org/physiotools/wfdb/app/wabp.c) for camera-rate PPG. The algorithm operates on the first derivative of the filtered signal:
   - The sample-to-sample difference is half-wave rectified (negative slopes zeroed) to isolate rising edges.
   - A moving sum over a 130ms window produces a "slope energy" signal with a bump at each pulse upstroke.
   - An adaptive dual-threshold system (Ta / T1 = Ta/3) self-calibrates during an 8-second learning period, tracking the running amplitude of detected pulses and decaying toward a minimum if no pulse is found for 2.5 seconds.
   - When the slope energy crosses the threshold, the algorithm searches backward from the crossing point to locate the pulse wave **onset** (the foot of the upstroke), not the peak.
   - A 250ms eye-closing period prevents within-beat retriggering. An additional 400ms minimum IBI guard at the BeatDetector layer rejects dicrotic notch reflections.
   - All time constants rescale dynamically based on the actual camera framerate.

4. **iOS multi-camera handling.** On multi-lens iPhones, virtual camera devices (labeled "Dual" or "Triple") trigger automatic macro lens switching when the finger is close to the lens, destabilising the signal. The camera initialisation routine enumerates all physical cameras, skips virtual composites and individual tele/ultra-wide lenses, and targets a single lens with confirmed torch capability.

### Dynamic Framerate Adaptation

- **High framerate request.** All `getUserMedia()` calls request `frameRate: { ideal: 120, min: 30 }`.
- **Actual FPS readback.** The negotiated framerate is read from `track.getSettings().frameRate` and used to recompute all framerate-dependent parameters.
- **Per-frame processing.** Uses `requestAnimationFrame` for the processing loop. (`requestVideoFrameCallback` was evaluated but caused waveform stuttering on iPhone under ISP framerate reduction and was reverted.)
- **Temporal resolution.** ±33ms at 30fps → ±17ms at 60fps → ±8ms at 120fps.

### Filter State Continuity

The IIR bandpass filter runs continuously across trials — the detector is never stopped between trials, only its callbacks are swapped via `BeatDetector.setCallbacks()`. This preserves filter state and avoids the 8-second WABP learning period re-running on every trial.

### Beat Detection Hardening

Several layers beyond the base WABP algorithm:

- **Dicrotic notch rejection**: Two-condition check — IBI < 75% of running average period AND the pair sum is within 15% of the expected period. A `DICROTIC_MIN_PERIODS = 4` guard ensures sufficient history before the check activates.
- **Frozen baseline period**: An EMA-updated floor ratio replaces the earlier fixed `DICROTIC_ABSOLUTE_FLOOR` approach, which failed to handle sustained half-cycle WABP lock-on.
- **IBI plausibility flagging**: Successive-difference check at a 30% threshold flags beats without rejecting them, enabling post-hoc QC without data loss.

### Signal Quality Index (SQI)

A per-frame SQI is computed via perfusion index (AC/DC ratio of the red channel). Per-trial SQI time series, total seconds below warning threshold, and final SQI are logged. Trials can be flagged post-hoc using `sqiGoodThreshold` and `sqiWarnThreshold` from the trial's `qualitySummary`.

### Tone Scheduling

Auditory tones are beat-triggered (not predictive), consistent with PAT 2.0 (Palmer, Murphy, Bird et al., 2025). Each detected beat schedules a tone at `delay = (averagePeriod / 2) * knobValue` seconds after detection.

Tones are scheduled via the Web Audio API's `AudioContext` clock on a dedicated high-priority audio thread, providing sub-millisecond scheduling precision. Each tone event records both the `AudioContext.currentTime` at scheduling and the corresponding `performance.now()` value for post-hoc timeline alignment.

A 350ms audio-side refractory gate (`MIN_TONE_SPACING`) prevents double-firing. `resetSchedulerState()` and `clearDropLog()` are called at each trial boundary via `endTrial()`.

### Motion Detection

The accelerometer (via DeviceMotion API) monitors for excessive movement during trials and triggers a "Keep still" pill notification. In `validation.html`, it additionally functions as a tap detector for sync marker insertion.

## Data Output

Each session generates a JSON file uploaded to the configured endpoint (with localStorage retry for offline resilience) and optionally downloaded locally.

### Onboarding (`onboarding_{id}_{date}.json`)

```json
{
  "type": "onboarding",
  "participantId": "42",
  "startedAt": "2025-04-01T10:00:00.000Z",
  "completedAt": "2025-04-01T10:12:00.000Z",
  "consent": {
    "agreed": true,
    "initials": "KW",
    "timestamp": "2025-04-01T10:01:30.000Z"
  },
  "schedule": {
    "availableDays": ["Mon","Tue","Wed","Thu","Fri","Sat","Sun"],
    "windows": {
      "morning":   { "start": "08:00", "end": "10:00" },
      "afternoon": { "start": "13:00", "end": "15:00" },
      "evening":   { "start": "19:00", "end": "21:00" }
    }
  },
  "deviceChecks": {
    "camera": true,
    "torch": true,
    "audio": true,
    "signal": true
  }
}
```

### PAT Session

Top-level fields: `sessionId`, `participantId`, `day`, `type` (`pat_only` or `paired`), `phases`, `counterbalance`, `startedAt`, `completedAt`, `wallClockAnchor`, `perfAnchor`, `device`, `status`, `data[]`.

**`type: "baseline"`**
- `recordedHR`: Array of instantaneous BPM values per detected beat.
- `totalBeats`: Total beat count (minimum 80 required to proceed; silent restart on failure).
- `ppgSampleRate`: Effective camera framerate.
- `ppgDiagnostics`: Frame timing, clipping rate, group delay estimate.
- `finalSqi`: SQI at end of baseline period.

**`type: "trial"`**
- `isPractice`: Boolean.
- `initialKnobValue`: Randomised starting dial position (±1.0, drawn from random visual offset).
- `instantPeriods` / `averagePeriods`: IBI arrays (seconds) at each beat.
- `knobScales`: Dial position at each beat.
- `currentDelays`: Computed tone delay (seconds) at each beat.
- `instantErrs`: Beat-to-beat period change.
- `recordedHR` / `instantBpms`: Instantaneous and smoothed BPM.
- `toneTimings[]`: Per-tone record — `beatTime`, `intendedDelay`, `scheduledAt` (AudioContext time), `perfNow`.
- `fingerLossEvents[]`: `lostAt` / `restoredAt` timestamps (performance.now).
- `ibiFlags[]`: Per-beat plausibility flags with magnitude and prev/current IBI.
- `dicroticRejections[]`: Per-rejection log.
- `sqiTimeSeries[]`: Per-SQI-update `{time, sqi}` records.
- `confidence`: 0–9 post-trial rating.
- `bodyPos`: Body region code (1–7), 8 ("did not feel it"), or -1 (not collected this trial).
- `qualitySummary`: `totalBeats`, `cleanBeats`, `flaggedBeats`, `flagRate`, `dicroticRejectsCount`, `sqiBadSeconds`, `sqiFinalValue`.
- `audioDropLog`: Per-trial audio scheduling drop events.
- `audioCtxState`: AudioContext state at trial boundary.
- `ppgDiagnostics`: Frame timing, clipping, group delay for this trial.

### EMA Session

Each EMA response is pushed to `sessionData.data[]` as:

```json
{
  "type": "ema_response",
  "session": "morning",
  "submittedAt": "2025-04-01T08:34:12.000Z",
  "sleep_hours": 7.5,
  "sleep_quality": 7,
  "restfulness": 6,
  "valence": 6,
  "arousal": 4,
  "maia_noticing": 5,
  "maia_emoawareness": 4,
  "ctx_activity": "Working / Studying",
  "ctx_social": "Alone",
  "ctx_location": "Home"
}
```

Morning sessions include `sleep_hours`, `sleep_quality`, and `restfulness`. Afternoon and evening sessions include only the common items (`valence` through `ctx_location`).

### Validation (`{id}_CNAP_SYNC.json`)

- `beatTimestamps[]`: `performance.now()` values for every detected PPG onset.
- `syncMarkers[]`: Manual or tap-triggered markers with timestamps and type labels.
- `toneTimestamps[]`: Timestamps of test tones for audio latency measurement.
- `pulseBeepTimestamps[]`: Timestamps of pulse-triggered beeps.

## Analysis

The data output is structured for compatibility with the PAT 2.0 analysis pipeline. Two scoring approaches are supported:

- **Phase-based consistency** (PAT 1.0): Selected delay expressed as proportion of IBI. Consistency measured across trials.
- **Delay-based consistency** (PAT 2.0, recommended): Raw delay in milliseconds, without IBI normalisation. Captures fixed neural propagation delays that should not vary with heart rate.

PAT 2.0 recommends classifying participants using individualised thresholds (simulating random responders with the participant's own IBI distribution) rather than continuous consistency scores. Example R analysis code: [osf.io/fp5sq](https://osf.io/fp5sq/).

For Bland-Altman validation against CNAP or ECG ground truth, `validation.html` provides contralateral-hand recording with tap markers for timeline alignment. Systematic offset of ~10–30ms from pulse transit time is expected and should be modelled as a fixed covariate.

## Browser and Device Compatibility

| Platform | Browser | Status |
|---|---|---|
| iOS 15+ | Safari | Fully supported (primary target). |
| iOS 15+ | Chrome / Firefox / Edge (iOS) | Functional — all use WebKit on iOS. |
| Android 8+ | Chrome | Fully supported. |
| Android | Firefox | Torch API support inconsistent; may fail silently. |
| Desktop | Any | Camera/torch unavailable. EMA check-ins work. |

The `DeviceMotionEvent.requestPermission()` flow (required on iOS 13+) is handled automatically.

## Known Limitations

- **No ISP control.** Browser APIs cannot disable auto-exposure, auto-white-balance, or auto-focus. Mitigated by SQI diagnostics enabling post-hoc QC.
- **Audio latency.** Web Audio API introduces 5–20ms non-deterministic latency vs. CoreAudio/AAudio in native apps. Constant within a session; does not affect consistency scores but limits absolute delay comparisons across platforms.
- **Variable framerate.** Mobile ISPs can drop frames or reduce framerate under occlusion. Mitigated by frame timing diagnostics.
- **IIR filter group delay.** ~30–80ms at cardiac frequencies depending on framerate. Constant across beats within a session; cancels in consistency scoring. Analytically estimated and logged in `ppgDiagnostics.estimatedGroupDelayMs`.

## Pending Implementation

- PAT 2.0 feature set: lock-then-slide-to-tick confirmation, unified retry budget, two-phase practice (tone-to-tone then tone-to-body), passive peripheral SQI indicator, dual-method classification framework.
- Formal CNAP validation (Bland-Altman, contralateral hand, tap markers).
- R scoring pipeline for PAT 2.0 delay-based consistency and individualised classification thresholds.
- Body map replacement: tappable SVG body outline in place of text grid.

## Acknowledgments

This project was developed at **The Ohio State University — The Affective Science Lab** under the supervision of **Dr. Kristen Lindquist**.

The ePAT is built upon and inspired by the following research and open-source software:

- **The Phase Adjustment Task (PAT):** The core psychophysical task logic is adapted from the PAT developed by Plans, Ponzo, Morelli, Cairo, Ring, Keating, Cunningham, Catmur, Murphy & Bird.
    - Original paper: [Measuring interoception: the phase adjustment task (2021)](https://www.sciencedirect.com/science/article/pii/S0301051121001642). *Biological Psychology*, 165, 108171.
    - PAT 2.0 refinements: Palmer, Murphy, Bird et al. (2025). Refinements of the Phase Adjustment Task (PAT 2.0). Preprint, [doi:10.31219/osf.io/4qtwv](https://doi.org/10.31219/osf.io/4qtwv).
    - Original source code (Swift/iOS): [huma-engineering/Phase-Adjustment-Task](https://github.com/huma-engineering/Phase-Adjustment-Task)

- **WABP Beat Detection Algorithm:** JavaScript port of the WABP arterial blood pressure pulse detector.
    - Zong, W., Heldt, T., Moody, G.B. & Mark, R.G. (2003). An open-source algorithm to detect onset of arterial blood pressure pulses. *Computers in Cardiology*, 30, 259–262.
    - PhysioNet source: [wabp.c](http://www.physionet.org/physiotools/wfdb/app/wabp.c)

- **Web-Based PPG Detection:** Camera-based heart rate monitoring approach informed by the open-source work of Richard Moore.
    - Source: [richrd/heart-rate-monitor](https://github.com/richrd/heart-rate-monitor)

- **Conceptual Framework:** The application of ePAT for ecological momentary assessment is grounded in the Theory of Constructed Emotion and research into interoceptive predictive processing.

---

**Maintained by:** Keegan Whitacre — The Ohio State University, Affective Science Lab
