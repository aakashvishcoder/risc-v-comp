# Section 6: User I/O & Headers

**What it does:** the parts you touch.
- **2 LEDs** your code controls
- **3 buttons:** RESET, BOOT, and USER
- **J3:** a debug header for a WCH-LinkE debugger (optional; USB flashing works without it)
- **J4/J5:** two GPIO headers that bring out 16 spare MCU pins

This is the last schematic section. After it, every label has both ends connected.

Stock re-checked 2026-11-06: LED 6.6M · 1k 34M · TS-1187A switch 968k (basic) · 1x40 header 55k.

## Parts in this section

| Ref | Value | Symbol | Footprint | LCSC |
|---|---|---|---|---|
| D3, D4 | Red LED | `Device:LED` | `LED_SMD:LED_0603_1608Metric` | C2286 |
| R15, R16 | 1k | `Device:R` | `Resistor_SMD:R_0402_1005Metric` | C11702 |
| SW1, SW2, SW3 | RESET / BOOT / USER | `Switch:SW_Push` | `Button_Switch_SMD:SW_Push_1P1T_XKB_TS-1187A` | C318884 |
| J3 | Debug 1x4 | `Connector_Generic:Conn_01x04` | `Connector_PinHeader_2.54mm:PinHeader_1x04_P2.54mm_Vertical` | C2337 (you solder) |
| J4, J5 | GPIO 1x10 | `Connector_Generic:Conn_01x10` | `Connector_PinHeader_2.54mm:PinHeader_1x10_P2.54mm_Vertical` | C2337 (you solder) |

The switch footprint has 4 pads but only 2 numbers: both "1" pads are joined inside the switch, and so are both "2" pads. That matches the 2-pin `SW_Push` symbol. **J3–J5:** in each symbol's properties, tick **Do not populate** so JLC doesn't place them. The footprints stay on the board, and you solder the headers yourself from one 1x40 strip.

**No BOM changes.** The refs are R15/R16 because R7–R14 are now in Section 5.

## Global labels in this section

Shapes are from **this section's** point of view. Point every label **left**, toward the MCU in box 4. All of them already exist, so pick them from the dropdown.

| Label | Shape | Connects to |
|---|---|---|
| `LED1` | Input | R15 → D3 |
| `LED2` | Input | R16 → D4 |
| `NRST` | Output | SW1 |
| `BOOT0` | Output | SW2 |
| `BTN_USER` | Output | SW3 |
| `SWDIO` | Bidirectional | J3 pin 2 |
| `SWCLK` | Output | J3 pin 3 |
| `PA1` … `PB15` (16 labels) | Bidirectional | J4 / J5 (table below) |

## Circuit

```
LED1 ── R15 1k ──►|── GND        (D3; triangle points to GND)
LED2 ── R16 1k ──►|── GND        (D4)

NRST     ── SW1 ── GND           RESET: press = reset (pin has internal pull-up + C18)
BOOT0    ── SW2 ── +3V3          BOOT:  press = BOOT0 high (R5 pulls it low when released)
BTN_USER ── SW3 ── GND           USER:  press = PA0 reads low (code enables internal pull-up)

J3 debug          J4 GPIO              J5 GPIO
1 +3V3            1  +3V3              1  +3V3
2 SWDIO           2  PA1               2  PB6   (I2C1 SCL)
3 SWCLK           3  PA2 (USART2 TX)   3  PB7   (I2C1 SDA)
4 GND             4  PA3 (USART2 RX)   4  PB10  (USART3 TX / I2C2 SCL)
                  5  PA8               5  PB11  (USART3 RX / I2C2 SDA)
                  6  PA15              6  PB12  (SPI2 NSS)
                  7  PB3               7  PB13  (SPI2 SCK)
                  8  PB4               8  PB14  (SPI2 MISO)
                  9  PB5               9  PB15  (SPI2 MOSI)
                  10 GND               10 GND
```

## Connections, pin by pin

**LEDs (active-high: write 1 = on)**
- `LED1` → R15 → D3 anode. D3 cathode → GND.
- `LED2` → R16 → D4 anode. D4 cathode → GND.
- 1k gives ~1.3 mA. That's dim but plenty indoors, and the same as the other LEDs on the board.

**Buttons**
- SW1 between `NRST` and **GND**. No resistor: NRST already has an internal pull-up, and C18 debounces it.
- SW2 between `BOOT0` and **+3V3**. R5 (Section 4) pulls BOOT0 low when it's released.
- SW3 between `BTN_USER` and **GND**. Turn on PA0's internal pull-up in code, then pressed = 0.
- **Entering the bootloader:** hold BOOT, tap RESET, release BOOT.

**Headers**
- Pin 1 of each header → **+3V3** and the last pin → **GND**, so pin 1 is the same on all three. Use the power symbols, not labels.
- J4 and J5 signal pins → the labels from the drawing above. They're named after the MCU pins, so the silkscreen can say exactly which pin is which.
- Anything you power from the header +3V3 pins adds to the AMS1117's load. Keep it under about **150 mA**.

## Power budget: unchanged
Two more LEDs at 1.3 mA each. That's inside the 4 mA already counted for LEDs in Section 1.

## Bring-up test (after boards arrive)
1. Press RESET. The program restarts (an LED blink will restart from the beginning).
2. Hold BOOT, tap RESET, and release. `wchisp info` finds the chip.
3. Flash a program that copies SW3 to LED1. Pressing USER lights D3.
4. Optional: plug in a WCH-LinkE (3V3/SWDIO/SWCLK/GND to J3). `wlink status` connects, and you can debug with GDB.

## Done-check
- [ ] LED triangles point toward GND
- [ ] SW1 → GND, **SW2 → +3V3**, SW3 → GND
- [ ] J3: +3V3, SWDIO, SWCLK, GND (in that order)
- [ ] J4/J5: +3V3 on pin 1, GND on pin 10, 16 labels matching the drawing
- [ ] `LCSC` field on D3, D4, R15, R16, SW1–SW3. J3–J5 set to **Do not populate**

Save and tell me. I'll run the **full-board ERC**. Expected result: **0 errors**. The only warnings left should be the 3 library-mismatch warnings (U1, U6, J2), which are harmless.
