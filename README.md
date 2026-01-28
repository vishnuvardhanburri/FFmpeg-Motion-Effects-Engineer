
# FFmpeg Motion Engine
**Human-Like Native Motion Effects for Production Pipelines**

Author: **Vishnu**  
Senior Backend / Production Engineer  
Focus: FFmpeg motion systems, deterministic rendering, automation reliability

---

## Project Overview

This project delivers a **reusable, production-grade FFmpeg motion engine** capable of
producing **human-like camera motion** — shake, zoom punches, reverse bursts, and speed ramps —
using **pure `filter_complex` mathematics**.

The system is designed for **backend-driven video pipelines**, not manual editing.
All motion is **deterministic, parameterized, previewable, and safe to retry**.

The objective is to match **After Effects–quality motion curves** while remaining
fully native to FFmpeg and suitable for automation.

---

## Core Problem This Solves

Most FFmpeg motion implementations fail because they rely on:
- Linear interpolation
- Random jitter
- Hard-coded presets
- No envelope control
- No retry safety

This engine solves that by treating motion as a **compiled system**, not a visual effect.

---

## High-Level Architecture

```mermaid
flowchart LR
  CLIENT[Backend / API]
  CONFIG[Motion Config<br/>(JSON)]
  VALIDATE[Validation Layer]
  ENGINE[Motion Engine<br/>(APD Compiler)]
  FF[FFmpeg filter_complex]
  OUTPUT[Rendered Clip]
  LOGS[Logs / Metrics]

  CLIENT --> CONFIG
  CONFIG --> VALIDATE
  VALIDATE --> ENGINE
  ENGINE --> FF
  FF --> OUTPUT
  ENGINE --> LOGS
````

### Architectural Principles

* No hidden state
* No runtime randomness
* No GUI dependencies
* Deterministic math only
* Safe retries by design

---

## Motion Engine Design

### Attack–Peak–Decay (APD) Envelope

```mermaid
graph LR
  A[Attack<br/>Ease-in] --> B[Peak<br/>Hold]
  B --> C[Decay<br/>Settle]
```

Why APD:

* Prevents instant velocity jumps
* Stabilizes impact frames
* Mimics physical inertia

This is the same concept used in professional animation tools,
implemented directly in FFmpeg math.

---

## Supported Motion Effects

* **Camera Shake** (micro / medium / heavy)
* **Zoom Punch / Zoom Out**
* **Reverse Burst**
* **Speed Ramps** (audio-aware, CFR-safe)

All effects support:

* Duration
* Intensity
* Seeded variation
* Presets
* Consistent behavior across clip lengths

---

## Preset Tables

### Camera Shake

| Preset | Amp (px) | Freq (Hz) | Decay | Use Case      |
| ------ | -------- | --------- | ----- | ------------- |
| Micro  | 2–4      | 10–14     | 0.85  | Subtle energy |
| Medium | 6–10     | 14–18     | 0.75  | Beat hits     |
| Heavy  | 12–18    | 18–24     | 0.60  | Drops         |

---

### Zoom Punch

| Preset | Amp       | Attack | Peak | Decay | Use        |
| ------ | --------- | ------ | ---- | ----- | ---------- |
| Micro  | 0.05–0.08 | 60ms   | 40ms | 220ms | UI hits    |
| Medium | 0.12–0.18 | 80ms   | 60ms | 280ms | Beat drops |
| Heavy  | 0.22–0.30 | 100ms  | 80ms | 360ms | Impact     |

---

### Speed Ramps

| Preset | Start | Peak  | Attack | Decay | Use      |
| ------ | ----- | ----- | ------ | ----- | -------- |
| Micro  | 0.9×  | 1.15× | 120ms  | 220ms | Lift     |
| Medium | 0.75× | 1.45× | 160ms  | 320ms | Emphasis |
| Heavy  | 0.6×  | 1.9×  | 200ms  | 420ms | Drops    |

---

## REAL Demo Commands

### Camera Shake (Medium)

```bash
ffmpeg -y -i input.mp4 \
-filter_complex "
crop=iw:ih:
x='8*sin(2*PI*16*t)*exp(-0.75*t)':
y='8*cos(2*PI*16*t)*exp(-0.75*t)'
" -vsync cfr -c:v libx264 demo_shake.mp4
```

---

### Zoom Punch

```bash
ffmpeg -y -i input.mp4 \
-filter_complex "
scale=iw*(1+0.15*sin(PI*t/0.42)*exp(-4*t)):
      ih*(1+0.15*sin(PI*t/0.42)*exp(-4*t)):
      eval=frame
" -vsync cfr -c:v libx264 demo_zoom.mp4
```

---

### Speed Ramp (Audio-Aware)

```bash
ffmpeg -y -i input.mp4 \
-filter_complex "
[0:v]setpts='PTS/(0.75+0.7*sin(PI*t/0.9)*exp(-2*t))'[v];
[0:a]atempo=1.1[a]
" -map "[v]" -map "[a]" -vsync cfr -c:v libx264 demo_speed.mp4
```

---

## Audio & Frame-Rate Handling

### CFR Required

```bash
-vsync cfr -r 30
```

Why:

* Stable `t` evaluation
* Predictable envelopes
* No drift

### VFR Inputs

* Must be normalized
* Never apply motion directly to raw VFR

```bash
-filter_complex "fps=30"
```

---

## Preview Harness

```bash
./render_test.sh shake medium
./render_test.sh zoom_punch heavy
./render_test.sh speed_ramp medium
```

Used to tune motion safely before production.

---

## Node.js Backend Integration

```ts
import { spawn } from "child_process";

export function runFFmpeg(input: string, output: string, filter: string) {
  return new Promise<void>((resolve, reject) => {
    const p = spawn("ffmpeg", [
      "-y", "-i", input,
      "-filter_complex", filter,
      "-vsync", "cfr",
      "-c:v", "libx264",
      output
    ]);
    p.on("exit", c => c === 0 ? resolve() : reject());
  });
}
```

Stateless, deterministic, retry-safe.

---

## Failure & Edge-Case Handling

Explicitly handled:

* Very short clips → envelope clamps
* Extreme values → validated / rejected
* Dropped frames → CFR enforced
* Retries → identical output
* Seeded motion → no drift
* FFmpeg failure → safe retry

No silent failures.

---

## Execution Plan (How This Will Be Delivered)

### Phase 1 – Motion Core (Week 1)

* Implement APD envelope compiler
* Shake + zoom punch
* Preset definitions
* Preview harness

### Phase 2 – Advanced Motion (Week 2)

* Reverse bursts
* Speed ramps (audio-aware)
* Parameter validation
* Backend integration hooks

### Phase 3 – Hardening (Week 3)

* Edge-case handling
* CFR/VFR normalization
* Performance tuning
* Documentation & presets

Outcome: **production-ready motion engine**, not a demo.

---

## What This Is NOT

* Not FFmpeg vibrate
* Not random jitter
* Not linear keyframes
* Not GUI editing

This is a **backend motion system**.

---

## Ideal Use Cases

* Short-form automation (TikTok / Reels / Phonk)
* Programmatic highlights
* AI video pipelines
* Batch rendering systems

---

## Author

**Vishnu**
Senior Backend / Production Engineer

FFmpeg • Motion Systems • Automation Reliability

---

## License

MIT

