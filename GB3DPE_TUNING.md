# GB3DPE Tuning Notes

> **Fork addition — not part of upstream Prusa-Firmware.** Filename is
> `GB3DPE_`-prefixed so it won't collide on rebase/merge with upstream. This is
> our living reasoning + experimental-iteration log for running the
> **GreenBoy3D pellet extruder (GB3DPE)** on a **Prusa MK3S** (Einsy / atmega2560).
> See `working.tmp/greenboy3d/CONTEXT.md` (untracked) for the hardware research.

## Hardware facts that drive the config

- Extruder: **planetary-geared stepper → auger/screw** pushing molten pellets.
  Gear ratio and auger volume/rev are **not published** — must be calibrated.
- Heater **70 W / 24 V**; single NTC thermistor (≈β4200, use Marlin table `5`,
  verified vs K-type TC); 2 fans; retraction is **mechanical** (reversing screw).
- Board is the stock **Einsy (atmega2560, 8-bit AVR)** → hard step-rate ceiling
  (~40 k steps/s/axis; DEDGE stepping ~2× that). This bounds extrusion flow.

## Extrusion (E axis) — the core relationship

The slicer emits E distances assuming **1.75 mm filament**, so it expects:

```
volume per E-mm = pi * (1.75/2)^2 = 2.405 mm^3 / E-mm
```

`DEFAULT_AXIS_STEPS_PER_UNIT[E]` maps that E-mm to auger rotation:

```
motor_rev / E-mm = steps_per_mm / (200 * microstep)
auger_rev / E-mm = (motor_rev / E-mm) / gear_ratio        # gear_ratio unknown
```

AVR step-rate ceiling sets the max flow:

```
max E speed (mm/s) = ~40000 / steps_per_mm          # x2 with DEDGE
max flow (mm^3/s)  = max E speed * 2.405
```

**Microstepping lever:** for the same auger rotation, µ1 uses 1/4 the steps/mm
of µ4 -> 1/4 the step rate. Lower µsteps = more AVR headroom for a geared auger.
(TMC2130 `INTPOL_E` interpolates to 256 internally, so motion stays smooth.)

## Calibration procedure

**Volumetric (gold standard, needs pellets):**
1. Heat to temp, prime until steady flow.
2. Command a known extrusion (e.g. `G1 E100 F<safe>`), catch the output.
3. Weigh it -> `measured_mass` (g).
4. `expected_mass = 100 * 2.405 * rho`  (PLA rho ~= 0.00124 g/mm^3 -> ~0.298 g)
5. `new_steps = old_steps * (expected_mass / measured_mass)`; repeat until ~1:1.

**Dry (rotational, no pellets):** mark the auger/coupling, command a known E
move, count revolutions -> gives current `auger_rev / E-mm`. Useful for relative
comparison and rate feel; absolute volume still needs the pellet test above.

## Key config levers (Firmware/variants/MK3S.h and MK3.h)

| Macro | Purpose | Current |
|---|---|---|
| `DEFAULT_AXIS_STEPS_PER_UNIT[E]` | E-mm -> steps (delivery) | 4000 |
| `TMC2130_USTEPS_E` | E microstepping | 4 |
| `DEFAULT_MAX_FEEDRATE[E]` / `_SILENT` | E speed ceiling (M203) | 600 |
| `MANUAL_FEEDRATE[E]` | LCD/manual extrude speed | 400 mm/min |
| `TMC2130_CURRENTS_R[E]` | E run current | 40 |
| `EXTRUDE_MINTEMP` | cold-extrude lockout | 175 |

## Priming / first layer (operational)

Pellet extruders need priming before any cal/print:
1. Fill hopper; confirm pellets actually feed (no bridging).
2. Heat to temp and **soak** a few min — barrel thermal mass >> a filament
   hotend; the whole barrel must reach temp to melt through.
3. Manual-extrude until **steady, clean, consistent** flow (first prime from
   empty takes patience). Wipe the purge blob (it drools — no melt retraction).
4. Only then run first-layer / Live-Z. Gaps in the zigzag = not primed / temp
   low: abort, prime more or +5-10 C, retry.

## Slicer settings (PrusaSlicer)

Starting point for the **0.4 mm** nozzle; refine as we calibrate.

- **Nozzle diameter** = your actual size (Printer Settings). Layer height
  0.15-0.20 for 0.4 mm. Bigger nozzle later -> layer ~0.5x nozzle, widen lines.
- **Filament diameter: keep 1.75 mm — never change.** Our e-steps calibration is
  built on the slicer computing E from 1.75 mm stock; changing it breaks it.
- **Linear Advance OFF**: add `M900 K0` to filament Start G-code (auger != spring).
- **Retraction**: start 0.5-1 mm or 0. Reversing the auger barely relieves
  chamber pressure; expect some stringing - don't rabbit-hole early.
- **Flow ceiling ~12 mm^3/s** (AVR at 8000 steps/mm). Max speed ~= 12 /
  (line_width x layer_height). At 0.4/0.2 (~130 mm/s) not limiting, but run the
  first prints slow (20-40 mm/s). Big nozzles: this is the real speed cap.
- **Temp**: start ~215 C PLA (bump +5-10 if under-melted); bed ~60 C.
- **Extrusion multiplier** ~1.0 after e-steps cal; trim from measured wall width.
- First layer: ~0.20 mm and slow for adhesion.

## Iteration log

### Iter 0 — baseline (as-flashed)
- Config: E steps 4000, µstep 4 (= 5.0 motor-rev/E-mm), Emax 600, Imanual 400.
- Observation (dry, no polymer): auger turns **too slow and delivers too little**
  vs expectation. Consistent with steps/mm ~8x low vs slush0's same-extruder value.

### Iter 1 — raise delivery + drop microstepping (APPLIED, pending test)
- Change: `TMC2130_USTEPS_E` 4->1; E steps 4000->8000 (= 40 motor-rev/E-mm,
  ~8x delivery, matches slush0 same-hardware start); cap `DEFAULT_MAX_FEEDRATE[E]`
  and `_SILENT` 600->5 mm/s and `MANUAL_FEEDRATE[E]` 400->250 mm/min
  (AVR-safe at 8000 steps/mm).
- Rationale: fixes "too slow + not enough" (both = low delivery); µ1 keeps step
  rate ~40 k at the new steps/mm. Starting estimate only.
- Expected max flow: ~5 mm/s E -> ~12 mm^3/s (DEDGE gives headroom above this).
- Result: _pending flash + dry test_ (watch: auger rotation rate/amount now ~8x;
  listen for missed-step beeps at MANUAL_FEEDRATE).
- Next: once flowing pellets, run the volumetric calibration to nail steps/mm.
- Note: if high-throughput flow (>~12 mm^3/s) is ever needed, the AVR step-rate
  ceiling is the wall -> a 32-bit control board is the real fix, not more steps/mm.

### Iter 2 — raise delivery ~4x (APPLIED, pending test)
- Prior result (iter1 @ 8000): direction OK (CCW), melt OK at 225 C, manual
  strand clean — but first-layer prints ~1/4 the needed volume (thin/dotting).
  Head speed fine, so it's pure volume-per-E-mm under-delivery, not rate/temp.
- Change: E steps 8000->32000 (~4x, eyeball from "quarter" under; at ustep1
  can't drop microstepping, so all via steps/mm). Feedrate caps dropped to keep
  step rate AVR-safe: `DEFAULT_MAX_FEEDRATE[E]`/`_SILENT` 5->1.5 mm/s,
  `MANUAL_FEEDRATE[E]` 250->75 mm/min. Max flow now ~3.6 mm^3/s (fine for 0.4).
- Result: _pending test before EOD_ (watch line fill; missed-step beeps at prime).
- Next: **volumetric mass calibration tomorrow** (scale on hand):
  `G1 E100 F60`, weigh, `new_steps = 32000 * (0.298 / measured_g)`. Also send
  `M900 K0` before the built-in first-layer cal (LA off for the auger).

<!-- Add Iter 3, 4, ... below as we measure. Keep: Change / Rationale / Result / Next -->
