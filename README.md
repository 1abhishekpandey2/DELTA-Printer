# DELTA Printer Firmware

This repository contains the Repetier firmware and the verified configuration
for this custom delta printer.

Start with [PRINTER_SETUP.md](PRINTER_SETUP.md). It records the working
firmware values, endstop wiring and X-endstop remap, calibration safeguards,
EEPROM procedure, and the complete Repetier-Host USB setup and print workflow.

## Key current settings

- Delta printer, RAMPS 1.3/1.4 board profile (`MOTHERBOARD 33`)
- USB serial speed: 115200 baud
- Printable radius: 100 mm
- Configured Z height: 219 mm
- X home switch is intentionally connected to the physical `X_MIN` header;
  firmware maps it as logical `X_MAX`.

Always test switches with `M119` before homing after firmware or wiring work.
