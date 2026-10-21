# GB3DPE tuning notes

> Fork addition, not part of upstream Prusa-Firmware. This is the reasoning and
> calibration log for the firmware config that runs the GreenBoy3D pellet
> extruder (GB3DPE) on our Prusa MK3S (Einsy board, ATmega2560). The hardware,
> bring-up and open items are in [`GB3DPE.md`](GB3DPE.md). The `GB3DPE_` prefix
> keeps the file from colliding with upstream on a merge.

## Hardware facts that drive the config

- The extruder is a planetary-geared stepper turning an auger that pushes molten
  pellets. GreenBoy3D does not publish the gear ratio or the auger's volume per
  revolution, so both have to be calibrated.
- 70 W / 24 V heater; one NTC thermistor (beta about 4200, Marlin table 5,
  checked against a type-K thermocouple); two fans. Retraction is mechanical:
  the screw reverses.
- The stock Einsy is an 8-bit AVR, which limits the step rate to roughly
  40,000 steps/s per axis (about twice that with the TMC2130's double-edge
  stepping, DEDGE). That limit bounds extrusion flow.

## Extrusion math

The slicer emits E distances as if it were extruding 1.75 mm filament, so one
E-mm has to deliver the volume of 1 mm of that filament:

$$
A = \pi\left(\frac{d_\text{fil}}{2}\right)^{2}
  = \pi\left(\frac{1.75}{2}\right)^{2}
  \approx 2.405\ \text{mm}^3\ \text{per E-mm}
$$

Steps/mm maps commanded E-mm to auger rotation, where $s$ is steps/mm, $\mu$ the
microstep setting, $200$ the full steps per revolution of a $1.8^\circ$ motor,
and $G$ the gearbox ratio:

$$
\frac{\text{motor rev}}{\text{E-mm}} = \frac{s}{200\,\mu}
\qquad\qquad
\frac{\text{auger rev}}{\text{E-mm}} = \frac{s}{200\,\mu\,G}
$$

$G$ is unpublished, but the volumetric calibration below cancels it.

The AVR step-rate limit $f_\text{max}\approx 40{,}000$ steps/s bounds feedrate
and volumetric flow:

$$
v_{E,\max} = \frac{f_\text{max}}{s}
\qquad\qquad
\dot{V}_{\max} = A\,v_{E,\max} = \frac{A\,f_\text{max}}{s}
$$

A lower $s$ leaves more headroom. Our calibrated $s$ is low enough that the
stock $\mu = 32$ is fine.

## Volumetric calibration

The auger is a positive-displacement pump, so extruded volume is proportional to
commanded steps. Extrude a known length $L$, weigh the result $m_\text{meas}$,
and compare it with the expected mass:

$$
m_\text{exp} = L\,A\,\rho,
\qquad \rho_\text{PLA} \approx 1.24\times10^{-3}\ \text{g/mm}^3
$$

$$
s_\text{new} = s_\text{old}\,\frac{m_\text{exp}}{m_\text{meas}}
$$

Repeat until $m_\text{meas}\approx m_\text{exp}$, then trim with the slicer's
extrusion multiplier from measured wall widths.

Our calibration on 2026-07-22, with $s_\text{old}=8000$ and $L=50$ mm:

$$
m_\text{exp} = 50 \times 2.405 \times 1.24\times10^{-3} \approx 0.1491\ \text{g},
\qquad m_\text{meas} \approx 1.005\ \text{g}
$$

$$
s_\text{new} = 8000 \times \frac{0.1491}{1.005} \approx 1187\ \text{steps/mm}
$$

As a cross-check, the stock $s = 280$ predicts

$$
m = 1.005 \times \frac{280}{8000} \approx 0.035\ \text{g} \approx \tfrac{1}{4}\,m_\text{exp},
$$

which matches the under-extrusion of about a quarter that we saw while the stock
value was still live (Iter 2). At $s\approx 1187$ and $\mu = 32$,
$v_{E,\max} \approx 40000/1187 \approx 34$ mm/s and
$\dot{V}_{\max} \approx 81$ mm³/s, so the AVR is not the bottleneck and stock
microstepping stays.

Without polymer, you can mark the auger or its coupling, command a known E move
and count revolutions to get auger rev per E-mm. That checks the rate, but the
absolute volume still needs the weigh test.

## Config values (`MK3S.h` and `MK3.h`)

| Macro                                 | Purpose                  | Value        |
| ------------------------------------- | ------------------------ | ------------ |
| `DEFAULT_AXIS_STEPS_PER_UNIT[E]`      | E-mm to steps (delivery) | 1187         |
| `TMC2130_USTEPS_E`                    | E microstepping          | 32 (stock)   |
| `DEFAULT_MAX_FEEDRATE[E]` / `_SILENT` | E speed limit (M203)     | 5 mm/s       |
| `MANUAL_FEEDRATE[E]`                  | LCD/manual extrude speed | 250 mm/min   |
| `TMC2130_CURRENTS_R[E]`               | E run current            | 40           |
| `EXTRUDE_MINTEMP`                     | cold-extrude lockout     | 175 (stock)  |

## Priming and first layer

A pellet extruder has to be primed before any calibration or print:

1. Fill the hopper and check that pellets actually feed and are not bridging.
2. Heat to temperature and let it soak for a few minutes. The barrel has far
   more thermal mass than a filament hotend, and all of it has to reach
   temperature before the melt runs through.
3. Extrude manually until the flow is steady and clean. The first prime from
   empty is slow. Wipe off the purge blob: the nozzle drools, because reversing
   the auger relieves little pressure.
4. Only then run the first-layer calibration or Live-Z. Gaps in the zigzag mean
   it is not primed or too cold: stop, prime more or raise the temperature by
   5–10 °C, and retry.

## Slicer settings (PrusaSlicer)

Starting points for a 0.4 mm nozzle.

- **Nozzle diameter:** the real one, in Printer Settings. Layer height
  0.15–0.20 mm at 0.4 mm. For larger nozzles use about half the nozzle diameter
  and widen the lines.
- **Filament diameter:** leave it at 1.75 mm. The E-steps calibration assumes
  the slicer computes E for 1.75 mm filament, and changing it breaks the
  calibration.
- **Linear Advance:** off. Add `M900 K0` to the filament start G-code; LA models
  filament compression, which an auger does not have.
- **Retraction:** 0.5–1 mm, or none. Reversing the auger relieves little chamber
  pressure, so expect some stringing.
- **Flow:** the AVR limit is about 81 mm³/s at 1187 steps/mm, but the 5 mm/s
  `M203 E` cap limits flow to about 12 mm³/s, and that is the real limit today.
  Maximum print speed is roughly flow / (line width × layer height), which is
  not limiting at 0.4/0.2. Run the first prints at 20–40 mm/s. For large
  nozzles, raise `M203 E` before blaming the AVR.
- **Temperature:** start at about 215 °C for PLA and raise it 5–10 °C if
  under-melted. Bed about 60 °C.
- **Extrusion multiplier:** about 1.0 after the E-steps calibration; trim it
  from measured wall width.
- **First layer:** about 0.20 mm, printed slowly.

## EEPROM overrides the firmware defaults

Prusa firmware stores motion settings in EEPROM, and on boot the EEPROM values
win over the `#define`s. Reflashing a new `DEFAULT_AXIS_STEPS_PER_UNIT`,
`TMC2130_USTEPS_E` or `DEFAULT_MAX_FEEDRATE` does not change the live value
unless you also factory-reset. We found this on 2026-07-22 with `M503`: after
flashing iterations 1 and 2, the live E axis was still at the stock 280 steps/mm,
microstep 32 and 120 mm/s, so none of our E changes had applied. Compile-time
settings did apply: probe offsets, travel limits, thermistor table,
`THERMAL_MODEL` and `MANUAL_FEEDRATE`.

To tune E, change it live instead of reflashing:

```gcode
M350 E32       ; E microstepping
M92 E1187      ; E steps/mm
M203 E5        ; E max feedrate (mm/s)
M500           ; save to EEPROM
M503           ; check that E reads 32 / 1187 / 5
```

Step `M92 E<n>` and `M500` to dial a value in; there is no need to reflash for
each step. If `M350` does not stick, a factory reset loads all the `#define`s at
once; afterwards send `D10` again and redo Live-Z and the mesh. Keep the
`#define`s at the final values so a fresh flash reproduces the printer, but on
this printer the values that count are the ones in EEPROM.

## Runtime-gated features (fan check, filament sensor)

`FANCHECK` and `FILAMENT_SENSOR` stay defined in our variant headers on purpose.
The `#define`s only compile the feature in; whether it runs is a runtime setting
in EEPROM, set from the LCD:

| Feature         | LCD menu                | EEPROM   | Read it            |
| --------------- | ----------------------- | -------- | ------------------ |
| Fan check       | Settings > Fan check    | `0x0F87` | `D3 Ax0f87 C1`     |
| Filament sensor | Settings > Fil. sensor  | `0x0F67` | `D3 Ax0f67 C1`     |

`00` means off. `Firmware/fancheck.cpp` reads `EEPROM_FAN_CHECK_ENABLED` into
`fans_check_enabled` and gates the error on it; the filament sensor is toggled
through `lcd_fsensor_enabled_set` in `Firmware/ultralcd.cpp`.

Both are off on our printer. The 5 V blowers that replaced the kit's 24 V ones
have no tacho signal, and a pellet toolhead has no filament to sense.
Undefining them would remove the menu options along with the checks, leaving no
way to turn them back on if we ever fit tacho fans.

This is the same trap as the E axis: before deciding whether a setting applied,
check whether it lives in EEPROM. A factory reset restores the `#define`
defaults and turns both checks back on.

## Iteration log

Each entry records what we knew at the time. Later entries correct earlier ones.

### Iter 0: baseline as flashed

- Config: E 4000 steps/mm, microstep 4 (5 motor rev per E-mm), E max feedrate
  600 mm/s, manual E feedrate 400 mm/min.
- Dry test, no polymer: the auger turned too slowly and delivered too little.
  That fits steps/mm being about 8× lower than slush0's config for the same
  extruder.

### Iter 1: raise delivery, drop microstepping

- Change: `TMC2130_USTEPS_E` 4 to 1; E steps 4000 to 8000 (40 motor rev per
  E-mm, about 8× the delivery, matching slush0's config). `DEFAULT_MAX_FEEDRATE[E]`
  and `_SILENT` capped from 600 to 5 mm/s and `MANUAL_FEEDRATE[E]` from 400 to
  250 mm/min, to stay within the AVR step rate at 8000 steps/mm.
- Rationale: "too slow" and "not enough" are both low delivery. Microstep 1
  keeps the step rate near 40,000 steps/s at the new steps/mm. A starting
  estimate only.
- Expected maximum flow: 5 mm/s of E, about 12 mm³/s. At 8000 steps/mm, more
  than that would have needed a 32-bit board rather than more steps/mm.
- Result: never applied; EEPROM kept the stock values (see Iter 2).

### Iter 2: raise delivery about 4×

- Iter 1 as we read it then: direction correct (CCW), melt fine at 225 °C, a
  clean manual strand, but first-layer prints had about a quarter of the needed
  volume (thin, dotted lines). Head speed was fine, so it looked like
  under-delivery per E-mm rather than a rate or temperature problem.
- Change: E steps 8000 to 32000 (about 4×, estimated from the "quarter"). Already
  at microstep 1, so the whole increase went into steps/mm. Feedrate caps lowered
  to keep the step rate within the AVR limit: `DEFAULT_MAX_FEEDRATE[E]` and
  `_SILENT` 5 to 1.5 mm/s, `MANUAL_FEEDRATE[E]` 250 to 75 mm/min. Maximum flow
  about 3.6 mm³/s, enough for 0.4 mm.
- Result: invalid, never applied. `M503` showed the live E axis at the stock
  280 steps/mm and microstep 32, held by EEPROM. Iterations 0 to 2 all ran at
  the stock filament ratio, so their extrusion observations do not count.

### Iter 3: apply the config live with M-codes

- Cause: EEPROM overrode every E `#define`. Reverted the `#define` to 8000 at
  microstep 1 (32000 came from the invalid stock-ratio reading) and switched to
  live tuning.
- Applied live: `M350 E1`, `M92 E8000`, `M203 E5`, `M500`; checked with `M503`.
- Rationale: 8000 at microstep 1 is slush0's value for the same extruder, the
  only valid reference we had, since our own readings were at the stock ratio.
- Next: weigh test with `G1 E100 F60`, then
  `new_steps = 8000 × 0.298 / measured_g` and `M500`. Send `M900 K0` before the
  built-in first-layer calibration. Once settled, update the `#define` to match.
- Result: see Iter 4.

### Iter 4: 1187 steps/mm at microstep 32 (current)

- Volumetric calibration, live via `M92` and `M500`, confirmed with `M503`: E50
  at 8000 steps/mm weighed 1.005 g against 0.149 g expected, so
  $s_\text{new} = 8000 \times 0.149 / 1.005 \approx 1187$ steps/mm.
- Microstepping back to the stock 32. Microstep 1 was only ever needed because
  of the stock-ratio confusion; at 1187 steps/mm the AVR allows about 34 mm/s
  of E. E max feedrate stays at 5 mm/s.
- Ran live for about a week and printed well. `M503` shows `M92 E1187`,
  `M350 E32`, `M203 E5`.
- Written into the `#define`s on 2026-07-30 so a fresh flash or factory reset
  reproduces the printer: `DEFAULT_AXIS_STEPS_PER_UNIT[E]` 8000 to 1187,
  `TMC2130_USTEPS_E` 1 to 32. The feedrate was already 5.

### Iter 5: final PINDA mount

- Printed and fitted the final PINDA Back Right Mount. Its designed offset from
  the nozzle is X 2.3 / Y 0.86 mm, so `X/Y_PROBE_OFFSET_FROM_EXTRUDER` went from
  20.4/8.6 to 2.3/0.86 (compile-time, applies on flash). The probe now sits much
  closer to the nozzle, so more of the mesh is reachable within the shorter X
  travel.
- Calibration is unchanged: `D10` to mark XYZ calibrated, then Calibrate Z,
  Live-Z, `G80`.
- The offset is the CAD value, not a measurement. The jog test in `GB3DPE.md`
  checks it.

<!-- New entries go below, as Change / Rationale / Result / Next. -->
