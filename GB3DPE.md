# GB3DPE: GreenBoy3D pellet extruder on a Prusa MK3S

This fork of Prusa-Firmware runs the GreenBoy3D Pellet Extruder V1 on an
Original Prusa i3 MK3S. This page is the procedure for setting up a converted
printer. What the firmware changes, and why, is in
[`GB3DPE_TUNING.md`](GB3DPE_TUNING.md).

- Mechanical conversion and wiring, with photos:
  [Appropedia guide](https://www.appropedia.org/Greenboy3D_V1_Pellet_Extruder_Conversion_for_the_Original_Prusa_i3_MK3S)
- CAD for our parts and the vendor parts:
  [Pellet Extruder - FAST Modified, version v3](https://cad.onshape.com/documents/08ae02778388e363fdd02b30/v/c9830c54e1eb7ccf4e0f1d72/e/b4788436987b1a21b68c4cc7)
  (Onshape)
- Bulk hopper: [`uwo-fast/feedstock-hopper`](https://github.com/uwo-fast/feedstock-hopper).
  The toolhead's own hopper holds 20–30 g, under an hour of printing.

We are not affiliated with GreenBoy3D or Prusa Research. GreenBoy3D publishes no
firmware or setup guide for this extruder; everything here was worked out on our
printer.

## The extruder

A planetary-geared stepper turns a screw in a heated barrel, which melts pellets
and pushes them out of an M6 nozzle. It replaces the MK3S hotend and extruder.
From the
[shop page](https://shop.greenboy3d.de/products/greenboy3d-pellet-extruder-v1):

| Parameter  | Value                                                  |
| ---------- | ------------------------------------------------------ |
| Supply     | 24 V; 70 W heater cartridge; one thermistor; two fans  |
| Drive      | planetary-geared stepper, ratio unpublished            |
| Max hotend | 330 °C; about 300 °C with PLA-printed parts            |
| Nozzle     | M6, 0.4–2.5 mm                                         |
| Flow       | 125–200 g/h at a 1 mm nozzle                           |
| Pellets    | 0.3–5 mm                                               |
| Retraction | mechanical (the screw reverses)                        |
| Mass       | about 700 g                                            |

The vendor CAD and STL files are on the
[GreenBoy3D wiki](https://wiki.greenboy3d.de/). The kit comes with no licence
stated; copyright stays with GreenBoy3D.

## What differs from a stock MK3S

Hardware (the Appropedia guide has the steps):

- **Mounting:** the kit's Prusa-MK3S adapter bolts to the MK3S extruder back
  plate with 3 × M3 × 25 mm socket head screws and M3 nuts, trimmed to clear the
  coupler and auger. The `Motor Faceplate Shim` (1.5 mm) sits between the
  gearbox faceplate and the adapter.
- **Fans:** the kit's 24 V blowers are replaced with 5 V 40 × 20 mm blowers
  (WINSINN), because the Einsy's fan outputs are 5 V.
- **Wiring:** the heater, thermistor, both fans and the extruder motor are
  re-terminated for the Einsy; the PINDA keeps its stock connector.
- **PINDA:** the `MK3 PINDA Back Right Mount` bolts to the fan duct and puts the
  probe at X 2.3 / Y 0.86 mm from the nozzle. The vendor's probe adapters put it
  outside the printer's travel.

Firmware: E steps/mm 1187, E speed limit 5 mm/s, X travel 210 mm, the probe
offset above, a tuned hotend PID, and Prusa's model-based thermal check turned
off. The full list is in [`GB3DPE_TUNING.md`](GB3DPE_TUNING.md).

## What you need

- A printer converted per the Appropedia guide.
- A computer with PrusaSlicer, and Pronterface or another serial terminal
  (115200 baud).
- A thermometer or thermocouple that reads to at least 250 °C.
- A scale that reads to 0.01 g.
- Pellets, and the density of the material (PLA is about 1.24 g/cm³).

## Setup

Do the steps in order. Each one ends with what you should see.

### 1. Build the firmware

On Linux, from the repository root:

```bash
python3 -m venv ~/.venvs/gb3dpe-fw
~/.venvs/gb3dpe-fw/bin/python utils/bootstrap.py
.dependencies/cmake-3.22.5/bin/cmake -S . -B build -G Ninja \
    -DCMAKE_MAKE_PROGRAM=$PWD/.dependencies/ninja-1.12.1/ninja \
    -DCMAKE_BUILD_TYPE=Release -DCMAKE_TOOLCHAIN_FILE=cmake/AvrGcc.cmake \
    -DFW_VARIANTS="MK3;MK3S" -DPython3_EXECUTABLE=$HOME/.venvs/gb3dpe-fw/bin/python
.dependencies/ninja-1.12.1/ninja -C build MK3S_ENGLISH
```

`bootstrap.py` downloads the pinned toolchain into `.dependencies/` and installs
its Python packages into the virtualenv.

**Check:** `build/build_gen/MK3S/MK3S_ENGLISH.hex` exists. Keep a stock
`MK3S_ENGLISH.hex`, built from upstream Prusa-Firmware, for rolling back.

### 2. Flash

In PrusaSlicer: Configuration > Flash printer firmware, then choose the `.hex`.

**Check:** the LCD status line reads `MK3S GB3DPE OK.`

### 3. Reset the EEPROM

The printer keeps its settings in EEPROM, and on boot those override the
firmware's defaults. A printer that has run stock firmware still holds stock
values, such as 280 E steps/mm. Erasing the EEPROM loads our defaults.

1. Hold the knob pressed while you switch the printer on.
2. When the screen shows `Factory RESET` and the printer beeps, release it.
3. Choose `All Data`.

This also erases the calibration and the print statistics. After the reset the
printer asks whether to guide you through setup. Answer No: its wizard runs
Calibrate XYZ, which cannot work on this printer (step 10).

**Check:** connect with Pronterface and send `M503`. These lines must match:

```text
M92 X100.00 Y100.00 Z400.00 E1187.00
M350 X16 Y16 Z16 E32
M203 X200.00 Y200.00 Z12.00 E5.00
M301 P18.81 I1.13 D78.11
```

### 4. Turn off the checks a pellet head cannot pass

In the LCD Settings menu, set each of these to `[Off]`. The EEPROM erase turns
all three on.

| LCD item      | Why it must be off                                      |
| ------------- | ------------------------------------------------------- |
| `Crash det.`  | The heavy toolhead trips the stall detection falsely.   |
| `Fans check`  | The 5 V blowers have no speed signal.                   |
| `Fil. sensor` | There is no filament. It is in its own submenu.         |

With no sensor connected, `Fil. sensor` can show `[On]` while runout shows
`[N/A]`. Turn it off anyway.

**Check:** all three read `[Off]`.

### 5. Check the thermistor

1. Cold, the hotend must read room temperature.
2. Heat to 180 °C, then 240 °C, and compare each with your thermometer at the
   nozzle.

**Check:** the readings agree to within about 10 °C. Ours was checked against a
type-K thermocouple and fits the firmware's table to within about 6 °C. If yours
is further off, stop: the sensor may not match the table (see
[`GB3DPE_TUNING.md`](GB3DPE_TUNING.md)).

### 6. Tune the hotend PID

With the hotend empty or a dish under the nozzle (it may ooze):

```gcode
M303 E0 S210 C8
```

When it finishes it prints Kp, Ki and Kd for each cycle. Apply the last set and
save:

```gcode
M301 P<Kp> I<Ki> D<Kd>
M500
```

**Check:** `M503` shows your values on the `M301` line. Ours were P18.81 I1.13
D78.11; yours should be close.

### 7. Check the extrusion direction

Fill the hopper, heat to 215 °C, and extrude a little from the LCD.

**Check:** the auger pushes melt out of the nozzle. If it pulls back, set
`INVERT_E0_DIR` to 1 in `Firmware/variants/MK3S.h` and rebuild.

### 8. Prime

1. Check that pellets feed from the hopper and do not bridge.
2. Hold at 215 °C for a few minutes. The barrel holds far more heat than a
   filament hotend, and all of it must be hot before melt flows through.
3. Extrude from the LCD until the flow is steady and clean. The first prime from
   empty takes several minutes. The nozzle drools between moves, because
   reversing the screw relieves little pressure.

**Check:** a steady, even strand.

### 9. Check the E steps by weight

One E-mm must deliver the volume of 1 mm of 1.75 mm filament, 2.405 mm³. Weigh
what the printer delivers:

1. Primed and at 215 °C, cut the strand flush at the nozzle.
2. Send:

   ```gcode
   M83
   G1 E200 F120
   ```

3. When it stops, cut the strand at the nozzle and weigh it.

Expected mass = 200 × 2.405 mm³ × density. For PLA that is
200 × 2.405 × 0.00124 = 0.60 g.

**Check:** the mass is within 5 % of expected. If not, set
`new steps/mm = 1187 × expected / measured`, then send `M92 E<new>` and `M500`.
Repeat until it is within 5 %.

### 10. Calibrate the bed

1. Send `D10`. This marks the XYZ calibration as done. Do not run Calibrate XYZ:
   with X travel at 210 mm the nozzle cannot reach its calibration points.
2. Run Calibration > Calibrate Z.
3. Send `M900 K0` (Linear Advance off), then run Calibration > First layer cal.
   and set Live-Z.

**Check:** the first-layer pattern is a continuous line. Gaps mean the
extruder is not primed or too cold: prime more or raise the temperature by
5–10 °C and repeat.

## Slicer settings (PrusaSlicer)

Start from the Original Prusa i3 MK3S profile and change:

- **Nozzle diameter:** the nozzle actually fitted.
- **Filament diameter:** leave it at 1.75 mm. The E steps assume it.
- **Filament start G-code:** add `M900 K0`. Linear Advance models filament
  compression, which an auger does not have.
- **Retraction:** 0.5–1 mm, or none. Expect some stringing.
- **Temperature:** about 215 °C for PLA; raise 5–10 °C if under-melted. Bed about
  60 °C.
- **Layer height:** 0.15–0.20 mm for a 0.4 mm nozzle; about half the nozzle
  diameter for larger nozzles.
- **Speed:** start at 20–40 mm/s. Flow is capped at about 12 mm³/s by the 5 mm/s
  E speed limit, so the speed limit is about 12 / (line width × layer height)
  mm/s.
- **Extrusion multiplier:** about 1.0; trim it from measured wall width.

## Safety

- Do not print above about 300 °C. The toolhead's printed parts are the limit.
  The firmware cuts out at 305 °C (`HEATER_0_MAXTEMP`).
- Prusa's model-based thermal check is off, because its model is fitted to the
  stock 40 W hotend. Classic thermal-runaway protection is on and is the only
  hotend protection.
- The 70 W heater draws about 2.9 A, against 1.7 A for the stock 40 W one. Watch
  the heater connector on the Einsy during long heat-ups.
- If the heavy heater block trips thermal runaway while heating normally, raise
  `TEMP_RUNAWAY_EXTRUDER_TIMEOUT` (45 s) in `Firmware/variants/MK3S.h`.
- Do not leave the printer heating unattended.

## Checking the probe offset

The offset X 2.3 / Y 0.86 mm is the mount's design value. To measure it:

1. `G28`, raise Z, and jog the nozzle over a fine mark near the bed centre
   (about X125 / Y105). Send `M114` and note X and Y.
2. Raise Z and jog the PINDA over the same mark. Send `M114` again.
3. Offset = nozzle position − PINDA position. Signs: X negative is left,
   Y negative is front.

If it differs from X 2.3 / Y 0.86, set `X_PROBE_OFFSET_FROM_EXTRUDER` and
`Y_PROBE_OFFSET_FROM_EXTRUDER` in `Firmware/variants/MK3S.h` and rebuild.

## Open items

- Run the probe-offset check above on our printer.
- Measure the toolhead's feed bore with calipers (18.30 mm in the STEP file).
- Design an adapter for the conveyor tube where it enters the vendor hopper cap.
- Re-base the fork as one commit on a tagged upstream release.
