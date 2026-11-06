# Section 3: USB-UART Console (CH343P)

**What it does:** shows up on your computer as a serial port. It connects to the CH32V307's **USART1** (PA9/PA10), which gives you:
1. a `printf` console from your very first program, and
2. a backup way to flash: the CH32V307's built-in bootloader also listens on USART1.

Stock re-checked 2026-11-06: CH343P 7,306 · LED 6.6M · 1k 34M · 100nF 34M.

## Parts in this section

| Ref | Value | Symbol | Footprint | LCSC |
|---|---|---|---|---|
| U4 | CH343P | `Interface_USB:CH343P` | `Package_DFN_QFN:WCH_QFN-16-1EP_3x3mm_P0.5mm_EP1.8x1.8mm` | C2846043 |
| C7, C8, C9 | 100nF | `Device:C` | `Capacitor_SMD:C_0402_1005Metric` | C1525 |
| D4 | Red LED | `Device:LED` | `LED_SMD:LED_0603_1608Metric` | C2286 |
| R14 | 1k | `Device:R` | `Resistor_SMD:R_0402_1005Metric` | C11702 |
| TP7 | CH343_V3_TP | `Connector:TestPoint` | `TestPoint:TestPoint_Pad_D1.5mm` | — (untick **Include in BOM**) |

**New since the original BOM:** D4 + R14, a "console ready" LED. Both are basic parts already used elsewhere, so there's no extra fee (~$0.01). It lights once your computer has set up the serial port, so you can tell at a glance whether the CH343 is working during bring-up.

*Reference numbers:* I'm using D4/R14 so they don't clash with the user LEDs planned for Section 6 (D2, D3, R12, R13). If KiCad annotates them differently, that's fine.

## Global labels in this section

Label **shapes describe the signal direction from this section's point of view**. UART names are from the **MCU's** point of view (`UART1_TX` = the MCU transmits).

| Label | Shape | Placed at | Points | Other end |
|---|---|---|---|---|
| `USB_UART_DP` | Bidirectional | Left edge of box 3 | ← left | Section 2, hub port 2 |
| `USB_UART_DN` | Bidirectional | Left edge of box 3 | ← left | Section 2, hub port 2 |
| `UART1_TX` | **Input** (comes into the CH343) | Bottom edge of box 3 | ↓ down | Section 4, CH32V307 **PA9** (USART1_TX) |
| `UART1_RX` | **Output** (leaves the CH343) | Bottom edge of box 3 | ↓ down | Section 4, CH32V307 **PA10** (USART1_RX) |

TX and RX cross: the CH343's **RXD** listens to the MCU's **TX**, and the CH343's **TXD** talks to the MCU's **RX**. Getting this backwards is the classic UART mistake, and the label names handle it for you:

```
CH343 pin 5 RXD  ◄── UART1_TX ◄── MCU PA9  (TX)
CH343 pin 4 TXD  ──► UART1_RX ──► MCU PA10 (RX)
```

## Circuit

```
           +5V                                  +3V3
            │                                    │
  C7 100nF ─┤                                    ├─ C9 100nF
            │ 3 VDD5                     1 VIO   │
            │      ┌───────────────────────┐     │
            ├──────┤ 9 VBUS                │     ├──── R14 1k ──► D4 ──┐
                   │                       │                          │ (cathode)
 USB_UART_DP ◄─► 7 UD+       U4      ~ACT 10 ─────────────────────────┘
 USB_UART_DN ◄─► 8 UD-     CH343P          │
                   │                  TXD 4 ──► UART1_RX  (Output)
                   │                  RXD 5 ◄── UART1_TX  (Input)
   CH343_V3 ──── 6 V3_OUT                  │
      │            │   ~DTR 12, ~RTS 13 ──── X
   C8 100nF, TP7   │   ~DCD 11, ~DSR 14, ~CTS 15, ~RI 16 ──── X
      │            │                       │
     GND           └── 2 GND, 17 EPAD ─────┴── GND
```

## Connections, pin by pin

**Power**
- Pin 3 `VDD5` → **+5V**. C7 (100nF) from +5V to GND, at pin 3.
- Pin 9 `VBUS` → **+5V**. Tells the CH343 the USB cable is present. Rated to 6.5V, so 5V direct is fine (datasheet §5.2).
- Pin 6 `V3_OUT` → local net **`CH343_V3`**: C8 (100nF) to GND and TP7. This is the chip's own internal 3.3V for its USB circuitry.
  - ⚠️ **Do NOT connect it to +3V3.** Same rule as the hub's HUB_3V3 in Section 2. Use a **local** label (press `L`) or just a short wire.
- Pin 1 `VIO` → **+3V3**, the same supply as the MCU, so the UART signal levels match. C9 (100nF) to GND.
- Pin 2 `GND` and pin 17 `GND_EPAD` (pad under the chip) → GND.

**USB**
- Pin 7 `UD+` → `USB_UART_DP`; pin 8 `UD-` → `USB_UART_DN`
- No resistors: the datasheet says UD+/UD- connect directly, and the pull-up is built in.

**UART**
- Pin 4 `TXD` → `UART1_RX`
- Pin 5 `RXD` → `UART1_TX`

**Console-ready LED**
- **+3V3** → R14 (1k) → D4 anode → D4 cathode → pin 10 `~ACT`
- `~ACT` is active-low: it pulls the LED's cathode to GND (LED on) once the computer has configured the device. The datasheet suggests exactly this hookup (§5.3).

**Unused pins → no-connect flag**
- Pin 12 `~DTR`. ⚠️ Never add a pull-down here: at power-up the CH343 reads this pin, and a pull-down switches it into RS-485 half-duplex mode (datasheet §5.3).
- Pin 13 `~RTS`
- Pins 11 `~DCD`, 14 `~DSR`, 15 `~CTS`, 16 `~RI`. These inputs have internal pull-ups.

*Skipped on purpose:* wiring DTR/RTS to the MCU's RESET/BOOT0 for button-free serial flashing. The USB bootloader plus the BOOT/RESET buttons (Section 6) already cover flashing. If you want it later, it's 2 transistors + 2 resistors.

**Test point**

| TP | Net | Expected reading |
|---|---|---|
| TP7 | CH343_V3 | ~3.3 V |

## Power budget: unchanged
CH343: ~3 mA from +5V plus µA on VIO. The new LED adds ~1.3 mA on +3V3. Section 1's totals still hold.

## Bring-up test (after boards arrive)
1. Plug in. TP7 ≈ 3.3 V.
2. **D4 lights** within a second or two. If not, the CH343 isn't being set up: check the hub (TP6) and the USB_UART lines.
3. macOS: `ls /dev/cu.*`. A new `/dev/cu.usbmodem…` (or `/dev/cu.wchusbserial…`) appears. macOS has a built-in driver for it.
4. Open it: `screen /dev/cu.usbmodemXXXX 115200` (quit with `Ctrl-A` then `K`). Once the MCU runs code, this is your console.

## Done-check
- [ ] CH343_V3 **not** connected to +3V3 (local label or short wire)
- [ ] VDD5 and VBUS both to +5V; VIO to +3V3
- [ ] TXD → UART1_RX, RXD → UART1_TX (crossed)
- [ ] No pull-down on DTR; no-connect flags on DTR, RTS, DCD, DSR, CTS, RI
- [ ] LED: +3V3 → R14 → D4 → ~ACT (triangle points toward ~ACT)
- [ ] Four global labels with the shapes above
- [ ] `LCSC` field on U4, C7–C9, D4, R14; TP7 excluded from BOM

Save and tell me. I'll run ERC and check every net. The `USB_UART` warnings from Section 2 should disappear.
