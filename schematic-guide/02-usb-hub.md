# Section 2: USB Hub (CH334F)

**What it does:** splits the single USB-C port into two USB devices: port 1 goes to the CH32V307 (flashing over USB / your own USB code later), port 2 goes to the CH343P (serial console). Ports 3 and 4 are unused.

Stock re-checked 2026-11-06: CH334F 2,441 · 12MHz crystal 254k · capacitors millions.

## Parts in this section

| Ref | Value | Symbol | Footprint | LCSC |
|---|---|---|---|---|
| U3 | CH334F | `Interface_USB:CH334F` | `Package_DFN_QFN:QFN-24-1EP_4x4mm_P0.5mm_EP2.7x2.7mm` | C5187527 |
| Y1 | 12MHz | `Device:Crystal_GND24` | `Crystal:Crystal_SMD_3225-4Pin_3.2x2.5mm` | C9002 |
| C4 | 1uF | `Device:C` | `Capacitor_SMD:C_0402_1005Metric` | C52923 |
| C5 | 10uF | `Device:C` | `Capacitor_SMD:C_0603_1608Metric` | C19702 |
| C6 | 100nF | `Device:C` | `Capacitor_SMD:C_0402_1005Metric` | C1525 |
| TP6 | HUB_3V3_TP | `Connector:TestPoint` | `TestPoint:TestPoint_Pad_D1.5mm` | — (untick **Include in BOM**) |

Add the `LCSC` field to each real part, as in Section 1.

## Global labels in this section

| Label | Shape | Placed at | Points | Other end |
|---|---|---|---|---|
| `USB_DP` | Bidirectional | Left edge of box 2 | ← left | Section 1 (J1 / U1) |
| `USB_DN` | Bidirectional | Left edge of box 2 | ← left | Section 1 (J1 / U1) |
| `USB_UART_DP` | Bidirectional | Right edge of box 2 | → right | Section 3, CH343P UD+ |
| `USB_UART_DN` | Bidirectional | Right edge of box 2 | → right | Section 3, CH343P UD- |
| `USB_MCU_DP` | Bidirectional | Bottom edge of box 2 | ↓ down | Section 4, CH32V307 PA12 (USB D+) |
| `USB_MCU_DN` | Bidirectional | Bottom edge of box 2 | ↓ down | Section 4, CH32V307 PA11 (USB D-) |

Each label points toward the section it connects to. Once you place `USB_DP` / `USB_DN` here, Section 1's labels have their other end.

## Circuit

```
                           +5V                      net HUB_3V3
                            │                           │
                     C4 1uF ┤                  ┌────────┼────────┬────────┐
                            │ 19 V5            │ 20     │        │        │
  USB_DP ◄─► 15 DPU+ ┌──────┴──────────────────┴─┐     C5 10uF  C6 100nF  TP6
  USB_DN ◄─► 14 DMU- │          U3 CH334F         │      │        │
                     │                            │     GND      GND
  USB_MCU_DP ◄─► 12 DP1+                 DP2+ 10 ◄─► USB_UART_DP
  USB_MCU_DN ◄─► 11 DM1-                 DM2- 9  ◄─► USB_UART_DN
                     │                            │
          X ──── 8 DP3+  7 DM3-  6 DP4+  5 DM4- ──── X   (unused ports: no-connect)
                     │                            │
         Y1 ┌─ 4 XI                     PSELF 18 ─── GND   (bus-powered)
     12MHz  └─ 3 XO                    ~RESET 16 ─── X
     (G pins → GND)                    ~OVCUR  1 ─── X
                     │                  ~PWREN 24 ─── X
                     │       LED1 22, LED2 23, LED3 13, LED4 21 ─── X
                     │       NC 2, NC 17 (already no-connect type)
                     └──── 25 GND (exposed pad) ─── GND
```

## Connections, pin by pin

**Power**
- Pin 19 `V5` → **+5V**. C4 (1uF) from +5V to GND, right at pin 19.
- Pin 20 `VDD33_LDO` → new net **`HUB_3V3`** (use a local label or just wire it). C5 (10uF) and C6 (100nF) from HUB_3V3 to GND. TP6 on HUB_3V3.
  - ⚠️ **Do NOT connect pin 20 to the board's `+3V3`.** This is the hub's *own* internal regulator output. Tying it to +3V3 would make two regulators fight each other.
  - The pin is "power output", so ERC is satisfied. No PWR_FLAG needed on HUB_3V3.
- Pin 25 `GND` (the big pad under the chip) → GND.

**Upstream (to the computer)**
- Pin 15 `DPU+` → `USB_DP`
- Pin 14 `DMU-` → `USB_DN`
- No resistors needed: the CH334 has the 1.5k pull-up built in.

**Downstream**
- Pin 12 `DP1+` → `USB_MCU_DP`; pin 11 `DM1-` → `USB_MCU_DN`
- Pin 10 `DP2+` → `USB_UART_DP`; pin 9 `DM2-` → `USB_UART_DN`
- Pins 5, 6, 7, 8 (ports 3 and 4) → **no-connect flag** on each. The hub has built-in pull-downs on unused ports.
- *If D+ and D- ever look swapped during routing: you may swap which port goes where (port 1 ↔ 2), but never swap D+ with D- within a port.*

**Crystal**
- Y1 pin 1 → pin 4 `XI`; Y1 pin 3 → pin 3 `XO`
- Y1 pins 2 and 4 (marked G) → GND
- **No load capacitors.** The CH334 has ~16pF built into XI/XO (datasheet §6.1).

**Configuration pins**
- Pin 18 `PSELF` → **GND**. Tells the computer this is a *bus-powered* hub (powered from the USB cable), which is the truth. It has a built-in pull-up, so a direct wire to GND is fine.
- Pin 16 `~RESET/CDP` → **no-connect**. The datasheet says to leave it floating; it has an internal ~25k pull-up.
- Pin 1 `~OVCUR` → **no-connect**. It has an internal weak pull-up, and our fuse in Section 1 handles overcurrent.
- Pin 24 `~PWREN` → **no-connect**. It's for switching power to external USB ports, which we don't have.
- Pins 13, 21, 22, 23 (LEDs) → **no-connect**. Pins 13/21 double as EEPROM SCL/SDA during reset, and their built-in pull-ups handle that.
- Pins 2 and 17 are `NC` in the symbol already. Nothing to do.

**Test point**

| TP | Net | Expected reading |
|---|---|---|
| TP6 | HUB_3V3 | 3.15–3.5 V (datasheet LDO range) |

## Power budget: unchanged
The hub draws up to 85 mA from +5V through its own regulator. This was already counted in Section 1 (~300 mA worst case from USB, within the 500 mA limit).

## Bring-up test (after boards arrive)
1. Plug in. TP6 should read ~3.3 V.
2. **macOS:** `system_profiler SPUSBDataType`. You should see a "USB 2.0 Hub" with the WCH vendor ID **0x1A86**.
3. When Section 3 is working, the CH343 appears under the hub as a serial port (`/dev/cu.usbmodem…` or `/dev/cu.wchusbserial…`).

## Done-check
- [ ] HUB_3V3 is **not** connected to +3V3
- [ ] PSELF to GND
- [ ] No-connect flags on: RESET, OVCUR, PWREN, LED1–LED4, ports 3 & 4 (DP3/DM3/DP4/DM4)
- [ ] No load capacitors on the crystal; Y1 G pins to GND
- [ ] Six global labels placed with the shapes and directions above
- [ ] `LCSC` field on U3, Y1, C4, C5, C6; TP6 excluded from BOM

Save and tell me. I'll run ERC and check every net.
