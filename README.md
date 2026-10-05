# 24 Wires Away

A small keyboard controller board (2 layers, ~29 × 57 mm) that drives a
full-size keyboard matrix over a single 24-wire connector: **16 rows + 8
columns = up to 128 keys**.

A Pro Micro–sized controller does not have enough free GPIO for a matrix
that size on its own, so the board expands its pins: 4 MCU pins are decoded
into 16 row lines, and 8 MCU pins read the 8 columns through a buffer.

The board is designed to work with both **5 V** controllers (Elite-C /
ATmega32U4) and **3.3 V** controllers with the same pinout (e.g. RP2040-based
boards like the Elite-Pi or SparkFun Pro Micro RP2040).

## Voltage compatibility

The matrix itself always runs at 5 V (`VBUS`). Each side of it is designed
to tolerate either MCU voltage:

- **Rows (MCU → matrix):** the 74HCT138 decoders run from `VBUS`. HCT inputs
  read anything above 2.0 V as high, so they accept both 3.3 V and 5 V logic.
  Their 5 V outputs only go to the matrix, never back to the MCU.
- **Columns (matrix → MCU):** the 74LVC245 runs from the MCU's own `VCC`, so
  its outputs follow the controller's voltage. Its inputs tolerate 5 V at
  either supply voltage.

**Requirement:** the controller's pin 24 must provide 5 V from USB (`VBUS`),
because that pin powers the decoders and pull-ups. This is true for the
Elite-C and RP2040 Pro Micro–style boards. On a nice!nano, pin 24 is the
battery input, so the board would get no 5 V there.

## Main ICs

### U1 — Pro Micro–compatible controller (Elite-C in the schematic)
Keebio's Elite-C: a Pro Micro–compatible module with USB-C, built around the
Microchip **ATmega32U4**, an 8-bit AVR running at 5 V / 16 MHz with native
USB. It runs the keyboard firmware (e.g. QMK) and shows up on the host as a
USB HID keyboard. It also powers the rest of the board from USB (`VBUS`).
3.3 V boards with the same pinout fit the same footprint.

### U3, U4 — 74HCT138 (3-to-8 line decoder/demultiplexer)
Each '138 takes a 3-bit address and pulls exactly one of its 8 outputs low
(outputs are active-low). The two chips are cascaded into a **4-to-16
decoder** that drives the 16 matrix rows:

- `PD1`, `PD0`, `PD4` → address inputs `A0`–`A2` on both chips
- `PC6` → acts as the 4th address bit: it goes to U3's active-low enable
  (`E0`) and U4's active-high enable (`E2`), so `PC6 = 0` selects U3
  (rows 1–8) and `PC6 = 1` selects U4 (rows 9–16)

The firmware scans the matrix by selecting one row at a time (driving it
low) and reading the columns: a column reading low means the key at that
row/column is pressed.

### U2 — 74LVC245APW (octal bus transceiver, used as a one-way buffer)
Sits between the 8 matrix columns (B side) and 8 MCU pins (A side).
`DIR` is tied low (data flows B → A, matrix → MCU), and `OE` is tied low
(always enabled). It is powered from the MCU's `VCC`, which is what makes the
column side work at 3.3 V or 5 V. In the schematic it uses KiCad's `74HC245`
symbol, which has the same pinout.

| Net | J1 pin | 74LVC245 | MCU pin |
|-----|--------|----------|---------|
| COL1 | 24 | B0 → A0 | PF4 (A3) |
| COL2 | 23 | B1 → A1 | PF5 (A2) |
| COL3 | 22 | B2 → A2 | PF6 (A1) |
| COL4 | 21 | B3 → A3 | PF7 (A0) |
| COL5 | 20 | B4 → A4 | PB1 (15) |
| COL6 | 19 | B5 → A5 | PB3 (14) |
| COL7 | 18 | B6 → A6 | PB2 (16) |
| COL8 | 17 | B7 → A7 | PB6 (10) |

## Passives

- **RN1, RN2** — 2 × 4-resistor 10 kΩ arrays (`R_Array_Convex_4x0603`) that
  pull `COL1`–`COL8` up to `VBUS`. They're required because the 74LVC245's
  inputs must not float, and the MCU's internal pull-ups can't reach the
  matrix through the buffer.
- **C1–C3** — 100 nF 0603 decoupling capacitors: C1 for U2 (`VCC`), C2 for
  U3 and C3 for U4 (`VBUS`). On the PCB, place each one next to its chip's
  supply pin.

## Connectors

- **J1** — JST PHD 2×12 (2.00 mm), the 24-wire keyboard matrix connector.
  Pins 1–16 are the rows (decoder outputs, U3 `Y0–Y7` then U4 `Y0–Y7`), and pins
  17–24 are the columns (`COL8`–`COL1`, through U2).
- **J3** — 6 solder-wire pads: `VBUS`, `GND`, and 4 spare MCU pins
  (`PD7`, `PE6`, `PB4`, `PB5`). Their purpose isn't labeled in the schematic.
  They might be meant for lock LEDs or other extras.

Unused controller pins: `TX0/PD3`, `RX1/PD2`, `RST`, and the extra Elite-C
pads `B7`, `D5`, `C7`, `F0`, `F1`.

## Status / to-do

- **PCB is out of date.** The schematic was changed (TXS0108E → 74LVC245,
  plus RN1, RN2 and C1–C3), but the board hasn't been updated yet. Run
  *Update PCB from Schematic*, reroute U2 (same TSSOP-20 footprint, different
  pinout), and place the new parts.
- **Firmware:** columns are active-low with external pull-ups, so don't
  enable the MCU's internal pull-ups on the column pins. They're harmless,
  but they do nothing here.
- **Ghosting:** with 16 × 8 lines and no diodes on this board, pressing
  several keys at once can cause ghosting unless the keyboard matrix itself
  has diodes.

## History

- **Oct 2026:** replaced the TXS0108E level translator with a 74LVC245. The
  TXS0108E's A side is limited to 3.6 V, so it couldn't run from a 5 V
  controller, and its `OE` pin was tied to GND, which disabled it. Added
  column pull-ups and decoupling capacitors.
