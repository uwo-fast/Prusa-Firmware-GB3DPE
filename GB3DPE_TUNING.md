# GB3DPE firmware reference

What this fork changes in Prusa-Firmware for the GreenBoy3D pellet extruder
(GB3DPE), why, and how the extruder was calibrated. The setup procedure is in
[`GB3DPE.md`](GB3DPE.md).

All changes are in `Firmware/variants/MK3S.h` and `Firmware/variants/MK3.h`.
Changed lines are marked `!!!CHANGED`; comments on a stock value we kept on
purpose are marked `GB3DPE:`.

## Changed settings

| Setting                               | Stock         | GB3DPE        | Why                                                            |
| ------------------------------------- | ------------- | ------------- | -------------------------------------------------------------- |
| `CUSTOM_MENDEL_NAME`                  | Prusa i3 MK3S | MK3S GB3DPE   | Identifies the build (`MK3 GB3DPE` in `MK3.h`). At most 16 characters (see below). |
| `DEFAULT_AXIS_STEPS_PER_UNIT[E]`      | 280           | 1187          | Calibrated by weight (below).                                   |
| `DEFAULT_MAX_FEEDRATE[E]`, `_SILENT`  | 120 mm/s      | 5 mm/s        | Kept from the 8000 steps/mm config; limits flow to about 12 mm³/s (below). |
| `MANUAL_FEEDRATE[E]`                  | 100 mm/min    | 250 mm/min    | LCD extrusion speed, raised for the geared auger.               |
| `TMC2130_CURRENTS_R[E]`               | 30            | 40            | More torque for the geared auger.                               |
| `X_MAX_POS`                           | 255 mm        | 210 mm        | Measured X travel with the wider toolhead.                      |
| `X/Y_PROBE_OFFSET_FROM_EXTRUDER`      | 23 / 5 mm     | 2.3 / 0.86 mm | Design offset of the MK3 PINDA Back Right Mount.                |
| `DEFAULT_Kp`, `Ki`, `Kd`              | 16.13 / 1.1625 / 56.23 | 18.81 / 1.13 / 78.11 | Autotune of our 70 W heater (`M303 E0 S210 C8`). |
| `THERMAL_MODEL`                       | on            | off           | Prusa's model is fitted to the stock 40 W hotend.               |

Kept at stock on purpose:

| Setting                       | Value   | Why                                                                 |
| ----------------------------- | ------- | ------------------------------------------------------------------- |
| `TEMP_SENSOR_0`               | 5       | Matches our thermistor (below). Table 11 over-read by 11–28 °C.      |
| `TMC2130_USTEPS_E`            | 32      | 1187 steps/mm leaves enough step-rate headroom (below).              |
| `FANCHECK`, `FILAMENT_SENSOR` | defined | Turned off at runtime instead (below).                               |
| `EXTRUDE_MINTEMP`             | 175 °C  | Unchanged.                                                           |

**Printer name.** The firmware copies `CUSTOM_MENDEL_NAME` into a 17-byte
EEPROM slot on first boot. A longer name overwrites the next EEPROM settings,
and `M115` then reports it cut off. Our first name, `Prusa i3 MK3S GB3DPE`, was
21 bytes. A printer that already stores a cut-off name keeps it until an EEPROM
erase; it is harmless.

## EEPROM overrides the firmware defaults

The firmware keeps the motion settings and the PID in EEPROM, and on boot the
EEPROM values replace the `#define`s. Flashing new defaults does not change a
printer that already has values stored. Either erase the EEPROM (LCD factory
reset, `All Data`), or set the values live:

```gcode
M92 E1187      ; E steps/mm
M350 E32       ; E microstepping
M203 E5        ; E max feedrate, mm/s
M301 P18.81 I1.13 D78.11
M500           ; save to EEPROM
M503           ; check
```

Compile-time settings take effect on flashing: the travel limits, probe offsets,
thermistor table, `THERMAL_MODEL` and `MANUAL_FEEDRATE`.

## Fan check and filament sensor

`FANCHECK` and `FILAMENT_SENSOR` stay defined. Those `#define`s only compile the
features in; whether they run is a runtime setting in EEPROM, set from the LCD.
Undefining them would also remove the menu items, so they could not be turned
back on if tacho fans or a sensor are fitted later.

| Feature         | LCD item               | EEPROM   | Read with D3   |
| --------------- | ---------------------- | -------- | -------------- |
| Fan check       | Settings > Fans check  | `0x0F87` | `D3 Ax0f87 C1` |
| Filament sensor | Settings > Fil. sensor | `0x0F67` | `D3 Ax0f67 C1` |

`00` is off. The LCD shows the same setting and is easier. D3 needs the address
in lowercase; Pronterface uppercases commands, and `D3 AX0F87` is then read as
a write to address 0. An EEPROM erase turns both features back on, along with
crash detection.

## Extrusion math

The slicer computes E as if extruding 1.75 mm filament, so one E-mm must deliver
the volume of 1 mm of that filament:

$$
A = \pi\left(\frac{1.75}{2}\right)^{2} \approx 2.405\ \text{mm}^3\ \text{per E-mm}
$$

With $s$ steps/mm, $\mu$ microsteps, a 200-step motor and gearbox ratio $G$:

$$
\frac{\text{auger rev}}{\text{E-mm}} = \frac{s}{200\,\mu\,G}
$$

$G$ is unpublished; the weight calibration below does not need it.

The Einsy's 8-bit AVR steps each axis at up to roughly 40,000 steps/s
($f_\text{max}$). That limits E speed and flow:

$$
v_{E,\max} = \frac{f_\text{max}}{s} \approx \frac{40000}{1187} \approx 34\ \text{mm/s},
\qquad
\dot{V}_{\max} = A\,v_{E,\max} \approx 81\ \text{mm}^3/\text{s}
$$

The 5 mm/s E speed limit is lower, so it sets the flow limit:
$5 \times 2.405 \approx 12$ mm³/s. Raise `M203 E` before the AVR becomes the
limit.

## Calibrating E steps by weight

The auger is a positive-displacement pump, so extruded mass is proportional to
steps. Extrude a length $L$, weigh it ($m_\text{meas}$), and compare with

$$
m_\text{exp} = L\,A\,\rho, \qquad \rho_\text{PLA} \approx 1.24 \times 10^{-3}\ \text{g/mm}^3
$$

$$
s_\text{new} = s_\text{old}\,\frac{m_\text{exp}}{m_\text{meas}}
$$

Our calibration (2026-07-22), with $s_\text{old} = 8000$ and $L = 50$ mm:

$$
m_\text{exp} = 50 \times 2.405 \times 1.24 \times 10^{-3} \approx 0.149\ \text{g},
\qquad m_\text{meas} = 1.005\ \text{g},
\qquad s_\text{new} = 8000 \times \frac{0.149}{1.005} \approx 1187
$$

The step-by-step check for a new printer is step 9 in [`GB3DPE.md`](GB3DPE.md).

## Thermistor

GreenBoy3D does not state the thermistor type. Against a type-K thermocouple,
table 11 (100k B3950) read 180 °C at a true 169 °C and 240 °C at a true 212 °C.
Fitting a beta to those points gives about 4200, which matches table 5 (Semitec
104GT-2, the stock table) to within about 6 °C over that range.

## History

Detail is in the git history of this file and the variant headers.

| Date       | Change                                                                                                                     |
| ---------- | -------------------------------------------------------------------------------------------------------------------------- |
| 2026-07-21 | First config: E 4000 steps/mm at microstep 4, interim PINDA mount, `THERMAL_MODEL` off. Table 11 tried, back to table 5.     |
| 2026-07-21 to 22 | E raised to 8000 at microstep 1, then 32000. Neither took effect: EEPROM still held the stock 280 steps/mm (found with `M503` on 2026-07-22). |
| 2026-07-22 | Set 8000 live with `M92`, then calibrated by weight to 1187 at microstep 32; E speed limit 5 mm/s.                            |
| 2026-07-30 | 1187 made the default. Final PINDA mount; probe offset 2.3 / 0.86 mm.                                                       |
| 2026-10-21 | PID autotuned to 18.81 / 1.13 / 78.11 and made the default. Filament sensor found on (unconnected) and turned off. Printer name shortened to fit its EEPROM slot. |
