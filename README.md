
# FFmpeg Motion Engine
**Human-Like Native Motion Effects for Production Pipelines**

Author: **Vishnu**  
Senior Backend / Production Engineer  
Focus: FFmpeg motion systems, deterministic rendering, automation reliability

---

## Project Intent

This project implements a **native FFmpeg motion engine** that produces
**cinematic, human-like motion** — shake, zoom punches, reverse bursts, and speed ramps —
using **pure `filter_complex` mathematics**.

The goal is **not visual tricks**.  
The goal is **repeatable, controllable, production-safe motion** that feels like
professional After Effects curves while remaining suitable for
**backend pipelines, batch rendering, and retries**.

---

## Engineering Approach (Why This Works)

I treat motion systems the same way I treat backend systems:

- Explicit math, not hidden presets  
- Deterministic output, not runtime randomness  
- Clear boundaries and failure handling  
- Fast iteration without destabilizing production  

Motion is **compiled**, not animated.

---

## High-Level Flow

```mermaid
flowchart LR
  INTENT[Creative Intent]
  PARAMS[Validated Parameters]
  APD[Attack → Peak → Decay Envelope]
  MATH[FFmpeg Math Expressions]
  FILTER[filter_complex]
  OUTPUT[Rendered Frames]

  INTENT --> PARAMS
  PARAMS --> APD
  APD --> MATH
  MATH --> FILTER
  FILTER --> OUTPUT
````

---

## Motion Model: Attack–Peak–Decay (APD)

```mermaid
graph LR
  A[Attack<br/>Ease-in] --> B[Peak<br/>Stabilize]
  B --> C[Decay<br/>Energy Loss]
```

Why APD:

* Prevents instant velocity jumps
* Avoids jitter at max intensity
* Mimics physical inertia

This is how professional animation tools work — expressed directly in FFmpeg math.

---

## Supported Effects

* **Camera Shake** (micro / medium / heavy, seeded)
* **Zoom Punch / Zoom Out**
* **Reverse Burst**
* **Speed Ramps** (audio-aware, CFR-safe)

---

## Preset Tables

### Camera Shake

| Preset | Amp (px) | Freq (Hz) | Decay | Use           |
| ------ | -------- | --------- | ----- | ------------- |
| Micro  | 2–4      | 10–14     | 0.85  | Subtle energy |
| Medium | 6–10     | 14–18     | 0.75  | Beat hits     |
| Heavy  | 12–18    | 18–24     | 0.60  | Drops         |

---

### Zoom Punch

| Preset | Amp       | Attack | Peak | Decay | Use           |
| ------ | --------- | ------ | ---- | ----- | ------------- |
| Micro  | 0.05–0.08 | 60ms   | 40ms | 220ms | UI hits       |
| Medium | 0.12–0.18 | 80ms   | 60ms | 280ms | Beat drops    |
| Heavy  | 0.22–0.30 | 100ms  | 80ms | 360ms | Strong impact |

---

### Speed Ramps

| Preset | Start | Peak  | Attack | Decay | Use         |
| ------ | ----- | ----- | ------ | ----- | ----------- |
| Micro  | 0.9×  | 1.15× | 120ms  | 220ms | Subtle lift |
| Medium | 0.75× | 1.45× | 160ms  | 320ms | Emphasis    |
| Heavy  | 0.6×  | 1.9×  | 200ms  | 420ms | Drops       |

---

## REAL Demo Clip Commands (Run These)

### 1️⃣ Camera Shake (Medium)

```bash
ffmpeg -y -i input.mp4 \
-filter_complex "
crop=iw:ih:
x='8*sin(2*PI*16*t)*exp(-0.75*t)':
y='8*cos(2*PI*16*t)*exp(-0.75*t)'
" \
-vsync cfr -c:v libx264 demo_shake_medium.mp4
```

---

### 2️⃣ Zoom Punch (Impact)

```bash
ffmpeg -y -i input.mp4 \
-filter_complex "
scale=iw*(1+0.15*sin(PI*t/0.42)*exp(-4*t)):
      ih*(1+0.15*sin(PI*t/0.42)*exp(-4*t)):
      eval=frame
" \
-vsync cfr -c:v libx264 demo_zoom_punch.mp4
```

---

### 3️⃣ Reverse Burst

```bash
ffmpeg -y -i input.mp4 \
-filter_complex "
scale=iw*(1-0.12*sin(PI*t/0.35)*exp(-3.5*t)):
      ih*(1-0.12*sin(PI*t/0.35)*exp(-3.5*t)):
      eval=frame
" \
-vsync cfr -c:v libx264 demo_reverse_burst.mp4
```

---

### 4️⃣ Speed Ramp (Medium, Audio-Safe)

```bash
ffmpeg -y -i input.mp4 \
-filter_complex "
[0:v]setpts='PTS/(0.75+0.7*sin(PI*t/0.9)*exp(-2*t))'[v];
[0:a]atempo=1.1[a]
" \
-map "[v]" -map "[a]" -vsync cfr -c:v libx264 demo_speed_ramp.mp4
```

---

## Audio-Aware Speed Ramp Handling

Rules followed:

* Video time is remapped explicitly
* Audio is never implicitly resampled
* `atempo` is clamped (0.5–2.0)
* No chained aggressive ramps

Production default:

* **Aggressive video ramp**
* **Mild audio stretch**
* Musicality preserved

---

## CFR vs VFR (Critical Notes)

### CFR — Required

```bash
-vsync cfr -r 30
```

Why:

* Stable `t` evaluation
* Predictable envelopes
* No drift

### VFR — Normalize First

Problems with VFR:

* Uneven motion
* Envelope distortion
* Audio desync

Fix:

```bash
-filter_complex "fps=30"
```

**Rule:** Never apply motion math directly to raw VFR input.

---

## Preview Harness

```bash
./render_test.sh shake medium
./render_test.sh zoom_punch heavy
./render_test.sh speed_ramp medium
```

Used for fast iteration without touching production code.

---

## Node.js Backend Wrapper (Example)

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

    p.on("exit", c => c === 0 ? resolve() : reject(new Error("ffmpeg failed")));
  });
}
```

* Stateless
* Deterministic
* Retry-safe
* Backend-friendly

---

## Failure & Edge-Case Notes (Important)

Handled explicitly:

* **Very short clips** → envelope auto-clamps
* **Extreme amplitudes** → rejected or clamped
* **Dropped frames** → CFR normalization
* **Retry renders** → identical output
* **Seeded motion** → no random drift
* **FFmpeg crash** → safe retry, no partial state

Nothing fails silently.

---

## What This Is NOT

* Not FFmpeg vibrate
* Not random jitter
* Not linear keyframes
* Not GUI editing

This is a **production motion engine**, not a demo.

---

## Ideal Use Cases

* Short-form automation (TikTok / Reels / Phonk)
* Programmatic highlight generation
* AI video post-processing
* Batch rendering pipelines

---

## Author

**Vishnu**
Senior Backend / Production Engineer

FFmpeg • Motion Systems • Automation Reliability

---

## License

MIT
