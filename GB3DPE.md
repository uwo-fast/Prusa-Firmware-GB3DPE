# GB3DPE: the GreenBoy3D pellet extruder on a Prusa MK3S

> **Fork addition, not part of upstream Prusa-Firmware.** Everything specific to
> running the GreenBoy3D Pellet Extruder V1 on our MK3S: the hardware, bring-up,
> and what is still open. The firmware's reasoning and calibration log is
> [`GB3DPE_TUNING.md`](GB3DPE_TUNING.md). The bulk hopper that feeds it, with its
> MK3S frame mount and hose outlet, is
> [`uwo-fast/feedstock-hopper`](https://github.com/uwo-fast/feedstock-hopper).
> The mechanical conversion and wiring, step by step with photos, are in the
> Appropedia guide
> [Greenboy3D V1 Pellet Extruder Conversion for the Original Prusa i3 MK3S](https://www.appropedia.org/Greenboy3D_V1_Pellet_Extruder_Conversion_for_the_Original_Prusa_i3_MK3S).
> We are not affiliated with GreenBoy3D or Prusa Research.

**Status.** In service in the lab, printing from virgin pellets and shredded
regrind.

## Why this exists

GreenBoy3D ships no firmware and effectively no documentation: the vendor wiki's
Marlin, Klipper, printing and recycling pages are "Coming Soon" stubs, and the
product page omits the gearbox ratio and any e-steps figure. Everything here is
self-derived.

## Hardware

A gravity-fed auger toolhead: a planetary-geared stepper turns a screw in a
heated barrel and pushes melt out of an M6 nozzle. It replaces the stock hotend
and extruder. From the
[shop page](https://shop.greenboy3d.de/products/greenboy3d-pellet-extruder-v1):

| Parameter | Value |
| --- | --- |
| Supply | 24 V; 70 W cartridge; one thermistor, type unstated; 2 fans |
| Drive | planetary-geared stepper, ratio unpublished |
| Max hotend | 330 °C stock, about 300 °C with PLA-printed parts |
| Nozzle | M6, 0.4–2.5 mm |
| Flow | 125–200 g/h at a 1 mm nozzle |
| Pellets | 0.3–5 mm |
| Retraction | mechanical (reversing screw) |
| Mass | about 700 g |

The kit includes a 1 m pellet conveyor tube, which feedstock-hopper's outlet
threads onto.

**Deviations on our unit:**
- **Adapter mounting:** the Prusa-MK3S adapter bolts straight to the MK3S
  extruder back plate with 3 × M3 × 25 mm socket head screws and M3 nuts,
  trimmed so they clear the coupler and auger. A shim between the gearbox
  faceplate and the adapter corrects the vertical offset between them:
  `/Prusa MK3/ > Motor Faceplate Shim`, 1.5 mm by default, set by the `height`
  variable in its part studio. The adapter itself is unmodified.
- **Fans:** the kit's two 24 V blowers were swapped for identical 5 V units
  (WINSINN 40 × 20 mm), because the Einsy cannot power them.
- **Wiring:** the heater, thermistor, both fans and the extruder motor were
  rewired with new connectors and crimps; the PINDA keeps its stock connector.
  The parts list is linked from the Appropedia guide.
- **PINDA bracket:** GreenBoy3D's probe adapters put the probe outside the X and
  Y travel limits. Our replacement,
  `Pellet-Extruder-Fan-Duct > MK3 PINDA Back Right Mount`, bolts to the existing
  pair of holes on the fan duct and returns the probe near the stock position.
  Its designed offset is X 2.3 / Y 0.86 mm.

Both of our parts are in the Onshape document
[Pellet Extruder - FAST Modified, version v3](https://cad.onshape.com/documents/08ae02778388e363fdd02b30/v/c9830c54e1eb7ccf4e0f1d72/e/b4788436987b1a21b68c4cc7),
alongside the vendor parts. The link is to a released version, so the parts
cannot change under it; newer work may be in the document's main workspace.

**Supplied CAD.** The vendor files come with the kit, with no licence stated.
They are imported into the Onshape document above for convenience; copyright
stays with GreenBoy3D, and the originals are on the
[vendor wiki](https://wiki.greenboy3d.de/). Bounding boxes, measured in those
files:

| Part | Bounding box (mm) |
| --- | --- |
| Pellet-Extruder-Hopper | 43.11 × 40.18 × 45.50 |
| Pellet-Extruder-Hopper-Cap | 43.22 × 54.30 × 6.99 |
| Pellet-Extruder-Fan-Duct | 104.38 × 44.16 × 68.42 |
| Prusa-MK3S+Adapter | 78.37 × 60.45 × 85.23 |
| Proximity-Sensor-Adapter 8/12 | 31.00 × 24.00 × 35.00 |
| Internal-Sliding-Pit | 37.40 × 19.77 × 11.47 |
| External-Sliding-Pit | 46.59 × 50.75 × 12.50 |

The stock hopper holds 20–30 g, under an hour at the quoted flow, which is why
the bulk hopper exists. The toolhead's feed bore is Ø18.30 mm, taken from the
STEP file rather than measured; it is the narrowest opening in the feed path.

## Bring-up

Do these in order.

1. **Build.** `python3 utils/bootstrap.py` once. On Debian its pip step fails
   under PEP 668; supply `pyelftools`, `polib` and `regex` from a virtualenv and
   pass `-DPython3_EXECUTABLE=...`. Then:

   ```bash
   cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release \
         -DCMAKE_TOOLCHAIN_FILE=cmake/AvrGcc.cmake -DFW_VARIANTS="MK3;MK3S"
   ninja -C build MK3S_ENGLISH     # build/build_gen/MK3S/MK3S_ENGLISH.hex
   ```

   Keep a stock `MK3S_ENGLISH.hex` built from upstream `MK3` for rollback.
2. **Flash** from PrusaSlicer. The boot screen must read "Prusa i3 MK3S GB3DPE".
3. **Check the thermistor before heating.** Cold, it must read room
   temperature. Then compare against a thermometer at 180 °C and 240 °C. We run
   table 5, verified against a type-K thermocouple; table 11 over-read by
   11–28 °C.
4. **Turn off what a pellet head does not use**, from the LCD menu (they are
   EEPROM settings, and the variant headers keep them compiled in so the menu
   entries exist): crash detection (the heavy head false-trips StallGuard), fan
   check (the 5 V blowers have no tacho), filament sensor. Read back with
   `D3 Ax0f87 C1` and `D3 Ax0f67 C1`; `00` is off.
5. **Check extrusion direction.** At temperature the auger must push melt out;
   if not, flip `INVERT_E0_DIR`.
6. **Calibrate.** Do not run "Calibrate XYZ": the X travel is too short to reach
   its circles. Send `D10` to mark XYZ as calibrated, then Calibrate Z, Live-Z,
   `G80`. X travel is capped at 210.
7. **Prime, then print** (priming is in `GB3DPE_TUNING.md`). Retraction is set in
   the slicer. Send `M900 K0` before the first-layer calibration.

EEPROM overrides the firmware's defaults on boot, so a reflash does not change
the live motion settings without a factory reset. Check them with `M503`; see
`GB3DPE_TUNING.md`.

**Safety.** `HEATER_0_MAXTEMP` is 305 °C and classic thermal-runaway protection
is active. `THERMAL_MODEL` is off, because Prusa's model is fitted to a 40 W E3D.
Do not print above about 300 °C: the toolhead's printed parts are the limit.
70 W is about 2.9 A against the stock 1.7 A, so watch the heater FET on long
soaks, and raise `TEMP_RUNAWAY_EXTRUDER_TIMEOUT` if the heavy block nuisance-trips
the 45 s window.

## Probe-offset jog test

Settles whether the probe offset is right, or whether X homing shifts under the
heavy head. It has not been run; the offsets are the mount's CAD nominal.

1. `G28`, raise Z, and jog the **nozzle** over a fine mark near the bed centre
   (about X125 / Y105). `M114`: record N.
2. Raise Z and jog the **PINDA** over the same mark. `M114`: record P.
3. The true offset is N − P, in Prusa's convention (X −left +right, Y −front
   +behind). It is independent of homing.

If it matches the configured X 2.3 / Y 0.86, the offset is right and any
misplacement is X homing (StallGuard under load). If not, set
`X/Y_PROBE_OFFSET_FROM_EXTRUDER` to the measured values.

## Open, parked while the printer is in service

- Run the probe-offset jog test above.
- Run `M303 E0 S210 C8` and `M500`. The PID gains are still stock 40 W values,
  and with `THERMAL_MODEL` off, `TEMP_RUNAWAY` is the only hotend protection.
- Re-fork cleanly: land these changes as one commit on a known upstream tag.
- Caliper the toolhead's feed bore (18.30 mm from the STEP file).
- The downstream end of the conveyor tube seats in the vendor hopper cap; a
  purpose-made adapter needs the current vendor CAD, which the wiki serves only
  through its app.
