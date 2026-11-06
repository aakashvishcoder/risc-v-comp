# Section 1: USB-C Input & Power

## Sheet setup (do this once)

- **File → Page Settings → Paper size: A3** (landscape). Fill in the title block: title "RISC-V Computer", rev "A", today's date.
- Divide the sheet into 6 areas with **Place → Graphic Rectangle** (thin dashed line). Put a **text label** at the top-left of each area:

```
+--------------------------+--------------------------+--------------------------+
| 1  USB-C INPUT & POWER   | 2  USB HUB               | 3  USB-UART CONSOLE      |
+--------------------------+--------------------------+--------------------------+
| 4  MCU CORE (CH32V307)                              | 5  MEMORY (PSRAM + SD)   |
|                                                     +--------------------------+
|                                                     | 6  USER I/O & HEADERS    |
+-----------------------------------------------------+--------------------------+
```

- Sections connect only through **power symbols** (+5V, +3V3, GND) and **net labels** (`USB_DP`, `USB_DM`, ...). No wires cross between boxes.
- **Add an `LCSC` field to every part** (Symbol Properties → + → name `LCSC`, value e.g. `C165948`). JLC reads this field to generate the assembly BOM later. Set test points to **Exclude from BOM**.

## Parts in this section

| Ref | Value | Symbol | Footprint | LCSC |
|---|---|---|---|---|
| J1 | USB-C 16P | `Connector:USB_C_Receptacle_USB2.0_16P` | `Connector_USB:USB_C_Receptacle_HRO_TYPE-C-31-M-12` | C165948 |
| R1, R2 | 5.1k | `Device:R` | `Resistor_SMD:R_0402_1005Metric` | C25905 |
| U1 | USBLC6-2SC6 | `Power_Protection:USBLC6-2SC6` | `Package_TO_SOT_SMD:SOT-23-6` | C7519 |
| F1 | 500mA PTC | `Device:Polyfuse` | `Fuse:Fuse_0805_2012Metric` | C20978 |
| U2 | AMS1117-3.3 | `Regulator_Linear:AMS1117-3.3` | `Package_TO_SOT_SMD:SOT-223-3_TabPin2` | C6186 |
| C1 | 10uF | `Device:C` | `Capacitor_SMD:C_0603_1608Metric` | C19702 |
| C2 | 22uF | `Device:C` | `Capacitor_SMD:C_0805_2012Metric` | C45783 |
| C3 | 100nF | `Device:C` | `Capacitor_SMD:C_0402_1005Metric` | C1525 |
| D1 | Red LED | `Device:LED` | `LED_SMD:LED_0603_1608Metric` | C2286 |
| R3 | 1k | `Device:R` | `Resistor_SMD:R_0402_1005Metric` | C11702 |
| TP1–TP5 | — | `Connector:TestPoint` | `TestPoint:TestPoint_Pad_D1.5mm` | — |

Power symbols (from the `power` library): `VBUS`, `+5V`, `+3V3`, `GND`, `PWR_FLAG`.

## Circuit

```
            VBUS                       +5V                          +3V3
J1 VBUS ────┬────────[F1 500mA]────────┬──────────┬───[U2 VI  VO]───┬──────┬──────┬────────┐
            │                          │          │      AMS1117    │      │      │        │
          U1 pin5                     TP2        C1 10uF   GND     C2     C3    TP3     R3 1k
          (VBUS)                       │          │        │      22uF  100nF   │        │
           TP1                        GND        GND      GND      │      │    GND     D1 ▼ (red)
                                                                  GND    GND             │
                                                                                        GND
J1 CC1 ── R1 5.1k ── GND          J1 D+ (A6 & B6 tied) ── U1 I/O1 ── label USB_DP
J1 CC2 ── R2 5.1k ── GND          J1 D- (A7 & B7 tied) ── U1 I/O2 ── label USB_DM
J1 SBU1, SBU2 ── no-connect (X)   U1 pin 2 GND ── GND
J1 GND, SHIELD ── GND             TP4, TP5 ── GND
```

## Connections, pin by pin

**J1 (USB-C)**
- `VBUS` (all four VBUS pins are one pin on the symbol) → power symbol **VBUS**
- `CC1` → R1 → GND
- `CC2` → R2 → GND. *Each CC line needs its own resistor. One shared resistor breaks C-to-C cables.*
- `D+`: the symbol has **two D+ pins (A6, B6)**. Wire **both** together → net label `USB_DP`
- `D-`: **two D- pins (A7, B7)**, wire both → `USB_DM`. *Missing one of these is the #1 USB-C mistake: the board then works with the plug in only one orientation.*
- `SBU1`, `SBU2` → **No-connect flag** (press `Q`)
- `GND` → GND
- `SHIELD` → GND

**U1 (USBLC6-2SC6 ESD)**
- Pin 1 & pin 6 (`I/O1`) → `USB_DP`
- Pin 3 & pin 4 (`I/O2`) → `USB_DM`
- Pin 5 (`VBUS`) → **VBUS**
- Pin 2 (`GND`) → GND
- On the PCB, place this right next to J1. Data lines flow *through* its pads.

**F1 (polyfuse)**: between **VBUS** and **+5V**.

**U2 (AMS1117-3.3)**
- `VI` (pin 3) → **+5V**
- `VO` (pin 2, also the tab) → **+3V3**
- `GND` (pin 1) → GND
- C1 (10uF) from +5V to GND, close to VI
- C2 (22uF) + C3 (100nF) from +3V3 to GND, close to VO

**D1 + R3 (power LED)**: +3V3 → R3 → D1 anode → D1 cathode → GND. Check the triangle on the symbol points toward GND.

**Test points**

| TP | Net | Expected reading |
|---|---|---|
| TP1 | VBUS | 4.75–5.25 V |
| TP2 | +5V | 4.7–5.2 V (a little below VBUS; the fuse drops ~0.1 V) |
| TP3 | +3V3 | 3.25–3.35 V |
| TP4, TP5 | GND | 0 V reference. Spread these out so a probe clip is always nearby |

**PWR_FLAGs**: place one on each of **VBUS**, **+5V**, and **GND**. The connector and fuse pins are "passive", so ERC can't tell those nets are powered. Without these flags you'll get "power input pin not driven" errors. Don't put one on +3V3: the regulator output already drives it.

## Power budget (the pre-order sanity check)

| Load | Rail | Typical | Worst case |
|---|---|---|---|
| CH32V307 @ 144MHz, USB + peripherals on | +3V3 | 30 mA | 60 mA |
| APS6404L PSRAM (active) | +3V3 | 10 mA | 40 mA |
| microSD card (writing) | +3V3 | 30 mA | 100 mA (short spikes ~200 mA) |
| LEDs (3 × 1.3 mA) | +3V3 | 4 mA | 4 mA |
| **+3V3 total through AMS1117** | | **~75 mA** | **~205 mA** |
| CH334F hub (own internal LDO) | +5V | 40 mA | 85 mA |
| CH343P | +5V | 3 mA | 10 mA |
| **Total from USB** | | **~120 mA** | **~300 mA** |

- USB 2.0 port limit: 500 mA. Fuse hold: 500 mA. **OK**
- AMS1117 heat at worst case: (5.0 − 3.3) V × 0.205 A ≈ **0.35 W**. SOT-223 handles ~1 W with a decent copper pour. **OK**
- AMS1117 dropout ~1.1 V: needs ≥ 4.4 V in. Worst-case input after cable and fuse ≈ 4.6 V. **OK, small margin.** If +3V3 sags under load during testing, swap in AP2112K-3.3 (C51118, 0.25 V dropout).
- VBUS capacitance: USB spec says ≤ 10 uF at plug-in. We'll have ~11 uF (C1 + hub caps). Close enough for a dev board.

## First-power test (after boards arrive)

1. **Before plugging in:** measure resistance from TP2 to GND and from TP3 to GND. Should be > 1 kΩ. Near 0 Ω means a solder short. Find it before powering.
2. Plug into a **USB charger or powered hub first, not your laptop**.
3. D1 lights. TP1, TP2, TP3 read within the table above.
4. Touch U2: warm is fine, too hot to hold means a short on +3V3.
5. Only then plug into your laptop.

## Done-check for this section

- [ ] Both D+ pins and both D- pins on J1 wired
- [ ] Separate 5.1k on CC1 and CC2
- [ ] SBU1/SBU2 have no-connect flags
- [ ] PWR_FLAG on VBUS, +5V, GND
- [ ] Every part has the `LCSC` field filled
- [ ] Footprints assigned (**Tools → Assign Footprints**, or set in each symbol's properties)

When this section is drawn, tell me. I'll check the schematic file, then we'll move on to **Section 2: USB hub**.
