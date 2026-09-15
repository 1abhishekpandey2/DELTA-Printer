# Delta Printer: Firmware and Repetier-Host Setup

This document records the working configuration for this custom delta printer.
It is intended for anyone who needs to repair, reconfigure, calibrate, or use
the printer.

> **Safety:** Keep a hand near the printer power switch during early homing and
> motion tests. Do not move endstop cables while the controller is powered.

## Hardware and firmware baseline

- Firmware: Repetier Firmware in this repository
- Board profile: `MOTHERBOARD 33` (RAMPS 1.3/1.4 style Arduino Mega profile)
- Kinematics: delta (`DRIVE_SYSTEM 3`)
- Firmware serial speed: `115200` baud
- Usable XY radius: `100 mm`
- Current configured Z height: `219 mm`

The following values are the current known-working values in
[`Configuration.h`](Configuration.h):

```cpp
#define X_MAX_PIN 3
#define INVERT_Y_DIR 1
#define ALWAYS_CHECK_ENDSTOPS 1
#define Z_MAX_LENGTH 219
#define DELTA_MAX_RADIUS 100
#define ROD_RADIUS 201.50
#define PRINTER_RADIUS 180
#define BAUDRATE 115200
```

`PRINTER_RADIUS` is geometry information here; the software travel boundary is
`DELTA_MAX_RADIUS`. Do not increase `DELTA_MAX_RADIUS` beyond the physically
safe radius without testing at low speed.

## Changes made for this printer

### Y motor direction

The Y tower originally moved in the wrong direction. It is corrected with:

```cpp
#define INVERT_Y_DIR 1
```

Change this only if a controlled manual Y move is reversed again.

### X home switch remap

The physical `X_MAX` input/header on the controller did not detect a known-good
switch. The working X switch was therefore moved to the **physical `X_MIN`
header** and is treated by firmware as the logical X maximum endstop:

```cpp
#include "pins.h"

#undef X_MAX_PIN
#define X_MAX_PIN 3
```

Keep the X tower's upper/home endstop plugged into **X_MIN**, not X_MAX. Do not
remove this override unless the controller's X_MAX input has been repaired and
tested.

### Endstops and homing

All three delta carriages home upward, so their switches are logical maximum
endstops:

| Tower | Physical controller header | Firmware signal |
| --- | --- | --- |
| X | `X_MIN` (intentional remap) | `X_MAX` |
| Y | `Y_MAX` | `Y_MAX` |
| Z | `Z_MAX` | `Z_MAX` |

The current endstop settings are:

```cpp
#define ENDSTOP_PULLUP_X_MAX true
#define ENDSTOP_X_MAX_INVERTING true
#define MAX_HARDWARE_ENDSTOP_X true
#define ENDSTOP_PULLUP_Y_MAX true
#define ENDSTOP_Y_MAX_INVERTING true
#define MAX_HARDWARE_ENDSTOP_Y true
#define ENDSTOP_PULLUP_Z_MAX true
#define ENDSTOP_Z_MAX_INVERTING true
#define MAX_HARDWARE_ENDSTOP_Z true
#define ALWAYS_CHECK_ENDSTOPS 1
```

For the normally-open switches used here, wire `COM` to `-`/GND and `NO` to
`S`/signal. Do **not** use the `+`/5 V pin. `ENDSTOP_*_MAX_INVERTING true` is
correct for this wiring.

### Test endstops before homing

Use the serial console (or Repetier-Host G-code input) to send:

```gcode
M119
```

With all upper switches released, expect:

```text
endstops hit: x_max:L y_max:L z_max:L
```

Hold one switch while sending `M119`. Only its matching signal must become
`H`:

| Held switch | Expected result |
| --- | --- |
| X | `x_max:H y_max:L z_max:L` |
| Y | `x_max:L y_max:H z_max:L` |
| Z | `x_max:L y_max:L z_max:H` |

`H` means the endstop is detected as hit; `L` means not hit. Do not home until
these results are correct. A manual switch click must be held while sending
`M119`; a click and release before the command will still show `L`.

## Firmware upload and EEPROM

1. Edit `Configuration.h`.
2. Compile and upload `Repetier.ino` from the Arduino IDE with the correct
   Arduino Mega board/port selected.
3. Reconnect and test endstops with `M119`.
4. After changing EEPROM-backed delta values such as `DELTA_MAX_RADIUS`, load
   the compiled defaults into EEPROM:

   ```gcode
   M502
   M500
   ```

`M502` resets EEPROM settings from `Configuration.h`; it overwrites any
EEPROM-only calibration. Record calibration values before using it.

## Repetier-Host setup (v2.3.2)

Repetier-Host controls the printer by USB. An STL model is loaded into the
host, sliced into G-code, previewed, and streamed directly to the printer. An
SD card is not required while printing from USB.

### Connect to the printer

1. Install and open Repetier-Host.
2. Connect the printer to the computer using USB.
3. Select **Config -> Printer Settings**.
4. In the **Connection** tab, choose the Windows COM port assigned to the
   Arduino and set **Baud rate** to `115200`.
5. Click **Apply**, then **OK**.
6. Click **Connect** in the upper-left corner. Successful connection messages
   appear in the **Log** panel.
7. Open **Manual Control** and send `M119` as the initial connection and
   endstop test.

The COM port differs by computer. In Windows, find it in Device Manager under
**Ports (COM & LPT)** when the printer is connected.

### Printer tab values

In **Config -> Printer Settings -> Printer**, use these current values:

| Setting | Value |
| --- | --- |
| Firmware type | Autodetect |
| Travel feed rate | 4800 mm/min |
| Z-axis feed rate | 100 mm/min |
| Manual extrusion speed | 2 / 20 mm/s (as shown in the current profile) |
| Manual retraction speed | 30 mm/s |
| Default extruder temperature | 200 C |
| Default heated-bed temperature | 55 C |
| Check extruder & bed temperature | Enabled |
| Temperature check interval | 3 seconds |
| Park position | X 0, Y 0, Z min 0 |
| Send ETA to printer display | Enabled |
| Disable extruder after job/kill | Enabled |
| Disable heated bed after job/kill | Enabled |
| Disable motors after job/kill | Enabled |
| Printer has SD card | Enabled |
| Printing-time compensation | 8% |
| Invert control direction (X/Y/Z) | Disabled |
| Flip X and Y | Disabled |

The **Go to Park Position after Job/Kill** and **Remove temperature requests
from Log** options are disabled in the current profile.

### Print an STL over USB

1. Click **Load** and choose an `.stl` file.
2. Position it on the bed if needed.
3. Open **Slicer**, choose the installed slicer profile, and slice the model
   into G-code. A conservative starting profile is 0.20 mm layer height,
   15% infill, and no supports for a simple sphere.
4. Open **Print Preview** and verify that the toolpath stays inside the
   100 mm radius and does not exceed the configured height.
5. Load the correct filament, set temperatures appropriate to that material,
   and home the printer.
6. Click **Print** to stream the G-code through USB.
7. Keep Repetier-Host open, the USB cable connected, and the computer awake
   until printing finishes.

An STL cannot be sent straight to the printer; it must be sliced into G-code.

## Motion-only test

Before a first print, run a no-filament motion test. Home first, then keep all
test moves comfortably within the 100 mm radius and above the bed. Never use
an untested G-code file near the bed without being ready to stop the printer.

## First-print checklist

- `M119` reports correct released and pressed endstop states.
- Homing stops each tower at its upper switch.
- Nozzle height at `Z=0` has been calibrated; paper should lightly drag below
  the nozzle.
- Model bounds fit inside the 100 mm print radius and 219 mm configured height.
- Correct filament temperatures and slicer profile are selected.
- The first layer is observed continuously.
