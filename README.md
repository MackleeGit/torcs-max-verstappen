# TORCS AI Driver — Max Verstappen (scr_server 1)

> IBM Bob × TORCS University Challenge  
> Track: **Corkscrew (E Track 2)** · Standing start · 1 lap · Zero damage

## Result

| Metric | Baseline (Alpha) | Final (Iota) |
|---|---|---|
| Laps Completed | 0 (DNF) | **1** |
| Best Lap Time | DNF | **01:58.15** |
| Total Damage | 1,863 | **0** |
| Top Speed | 237 km/h | **235 km/h** |

Broke the sub-2:00 barrier on Corkscrew from a standing start with zero wall contact.

---

## How it works

The driver logic lives entirely in `drive_example()` inside [`snakeoil3_gym.py`](./snakeoil3_gym.py).  
It uses TORCS's 19-sensor track array and vehicle state to make real-time driving decisions at 50 steps/second.

### Core systems

**1. Dual-sensor speed targeting**  
Two separate readings from the forward track sensors drive two separate jobs:
- `straight_dist` = `max()` of sensors 7–11 — sets straight-line target speed; immune to elevation banks clipping a single sensor
- `corner_dist` = `min()` of sensors 8–10 — triggers corner braking only when the road *directly ahead* is genuinely closing in (< 55 m)

When the car is near the track edge (`|trackPos| > 0.5`), diagonal sensors see the barrier wall and lie. In that case `straight_dist` falls back to the dead-ahead sensor only.

**2. Proportional braking**  
No binary brake on/off. Brake pressure scales linearly with how far over the corner target speed Max is:
```
brake = min(1.0, (speed_excess - 10) / 60.0)
```
Soft ramp prevents scrubbing speed unnecessarily through fast sweepers.

**3. Adaptive apex throttle**  
Two-rate throttle build depending on steering angle:
- `|steer| < 0.2` (car pointed at exit): `+0.25` per tick — full launch
- Still cornering: `+0.15` per tick — steady build

**4. Adaptive centering gain**  
When near the edge the centering correction doubles (`0.15` → `0.30`) to snap Max back to the racing line quickly rather than loitering on the boundary.

**5. Traction control**  
If rear wheels spin faster than fronts by more than 5 rad/s, throttle is cut by 0.2 to recover grip.

---

## Optimization history

IBM Bob was used iteratively to analyse the baseline script, identify failure modes from race telemetry, and implement targeted fixes one run at a time.

| Run | Lap Time | Top Speed | Damage | Key Change |
|---|---|---|---|---|
| Alpha | DNF | 237 km/h | 1,863 | Baseline — no brakes, blind throttle, stuck on barrier |
| Beta | 02:24.17 | 196 km/h | 0 | Added braking, smoothed steering, corner-aware speed |
| Gamma | 02:15.40 | 217 km/h | 0 | 5-sensor array, `dist × 2.0` speed multiplier |
| Epsilon | 02:06.71 | 225 km/h | 0 | Multiplier `2.5`, tighter brake ramp `/50` |
| Zeta | 02:04.90 | 228 km/h | 0 | Split `max`/`min` dual-sensor logic, brake ramp `/60` |
| Eta | 02:11.21 | 228 km/h | 0 | ⚠️ Regression — corner threshold `< 45` too aggressive, off-track on final turn |
| Theta | 02:02.42 | 229 km/h | 0 | Calibrated threshold to `< 55`, multiplier `2.8` |
| **Iota** | **01:58.15** | **235 km/h** | **0** | Edge sensor fix, adaptive centering, apex throttle threshold `0.2` |

---

## Setup

### Requirements
- [TORCS](https://ibm.box.com/v/TORCSdownloadzip) with SCR patch
- Python 3

### Run
1. Launch TORCS → Quick Race → Corkscrew → Driver: `scr_server 1` → 1 Lap → New Race
2. In a terminal:
```bash
python snakeoil3_gym.py
```

---

## Credits

- Base client library: Chris X Edwards (`snakeoil3_gym.py`)
- Optimization: IBM Bob (iterative telemetry analysis & code refactoring)
- Challenge: [IBM Bob University TORCS Challenge](https://skills.yourlearning.ibm.com/activity/PLAN-6FA860C53B3A)
