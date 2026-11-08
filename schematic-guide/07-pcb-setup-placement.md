# PCB Part 1: Setup, Rules & Placement

The schematic is done (commit `f6fd7f3`). This part gets the board **set up and the parts placed**. **No routing yet.** We route in Part 2, after I check the placement. Placement decides about 80% of how easy routing will be.

## 1. Board size: 50 × 50 mm, 4 layers, 1.6 mm thick

- **Why 50 × 50:** it's JLC's cheapest size bracket, and it fits everything. All the parts take up about 1,000 mm² of the 2,500 mm² board.
- **Why not smaller:** the two 1x10 headers are 25.4 mm long, so they need two 25 mm edges. 45 × 45 is possible, but it makes routing the LQFP-64 painful.
- **Corners:** round them to 1 mm.
- **Mounting holes:** four `MountingHole:MountingHole_2.7mm_M2.5`, one at each corner, centered **3 mm from each edge**. Use M2.5 rather than M3 to save space.

**How:** select the Edge.Cuts layer, then Place → Rectangle, 0,0 → 50,50. Then Edit → Fillet lines → 1 mm. Mounting holes: Place → Footprint.

## 2. Board Setup (File → Board Setup)

### Board Stackup → Physical Stackup
- **Copper layers: 4.** Board thickness: **1.6 mm**. This matches JLC's default 4-layer stackup, **JLC04161H-7628**.

| Layer | Use |
|---|---|
| F.Cu (L1) | Signals + all parts (all on top: JLC assembles one side only, which is cheaper) |
| In1.Cu (L2) | **Solid GND plane.** Never route on it |
| In2.Cu (L3) | **Power plane:** +3V3 for most of the board, and a +5V area under the USB/hub/CH343 corner |
| B.Cu (L4) | Signals that need to cross over others |

In **Board Setup → Board Editor Layers**, set In1.Cu and In2.Cu to type **power**. You can also rename them `GND` and `PWR`.

### Design Rules → Constraints (JLC 4-layer standard, no extra cost)

| Setting | Value |
|---|---|
| Minimum clearance | 0.127 mm |
| Minimum track width | 0.127 mm |
| Minimum annular width | 0.1 mm |
| Minimum via diameter | 0.5 mm |
| Minimum through hole | 0.3 mm |
| Hole to hole clearance | 0.5 mm |
| Copper to edge clearance | 0.3 mm |
| Minimum silkscreen text height / thickness | 0.8 / 0.15 mm |

### Design Rules → Net Classes

| Net class | Track | Clearance | Via (dia / drill) | Diff pair width / gap | Pattern (Netclass assignments) |
|---|---|---|---|---|---|
| Default | 0.2 | 0.15 | 0.5 / 0.3 | — | (everything else) |
| Power | 0.3 | 0.15 | 0.6 / 0.3 | — | `VBUS`, `+5V`, `+3V3`, `GND`, `HUB_3V3`, `CH343_V3` |
| USB | 0.3 | 0.2 | 0.5 / 0.3 | **0.3 / 0.2** | `*_DP`, `*_DN` |

- **USB impedance:** USB needs **90 Ω** between the two wires of each pair. My estimate for JLC04161H-7628 (L1 over the L2 GND plane, about 0.21 mm apart) is **0.3 mm traces with a 0.2 mm gap**, which gives about 89 Ω.
  - **Before ordering, check it** in JLC's online impedance calculator: pick JLC04161H-7628, "differential, outer layer, 90 Ω". Then put its width and gap here.
  - **Where it matters most:** the hub's upstream link to your computer runs at 480 Mbit/s. The MCU's port only runs at 12 Mbit/s, so it's forgiving.
- **Pair names:** KiCad pairs nets whose names differ only in a final P/N. So `USB_DP` and `USB_DN` (also `USB_MCU_*` and `USB_UART_*`) are detected as differential pairs automatically. Route each pair with the **Route Differential Pair** tool (`6`).

## 3. Import the parts

1. Open the PCB editor, then **Tools → Update PCB from Schematic** (F8).
2. Leave "Delete footprints with no symbol" **unticked**, or it removes your mounting holes.
3. Click Update. All 66 footprints drop in a pile, with thin "ratsnest" lines showing what connects to what.

## 4. Placement plan

Coordinates: origin is the top-left corner, x goes right, y goes down. LQFP-64 pin layout: pins 1–16 are on the left side, 17–32 the bottom, 33–48 the right, and 49–64 the top.

```
 y=0  ┌──────────────────────────────────────────────────┐
      │ ○                [J1 USB-C]          [J3 debug]  ○│  ← USB-C overhangs the top edge
      │      U2 AMS1117   U1 ESD   F1                     │
      │      C1 C2 C3                U3 hub + Y1   U4 CH343│
  J4  │ ║                                     D2 R4       ║│  J5
 (left│ ║          Y2 C10 C11                             ║│ (right
 edge,│ ║               ┌──────────────┐                  ║│  edge,
 1x10)│ ║               │   U5 MCU     │  decaps on every ║│  1x10)
      │ ║               │  LQFP-64     │  side, ≤2 mm from║│
      │ ║               │              │  their pins      ║│
      │ ║               └──────────────┘                  ║│
      │ ║        U6 PSRAM                                 ║│
      │      D3 D4                                       │
      │ ○  [SW1 RST] [SW2 BOOT] [SW3 USER]  [ J2 microSD ] ○│  ← SD card slides in from the bottom edge
 y=50 └──────────────────────────────────────────────────┘
      x=0                                              x=50
```

**Placement rules, in order of importance**
1. **J1 USB-C** sits on the top edge, centered. Its front lip lines up with the board edge (the footprint's Edge.Cuts marker shows where). **U1 ESD** goes right behind it, so the USB data lines pass through U1's pads *before* going anywhere else.
2. **U5 MCU** goes in the middle. Start straight (0°), then rotate in 90° steps until the most ratsnest lines are short. **Pin 1 (the dot) top-left** is a good start: then USB (pins 44/45) and UART (42/43) face the hub/CH343 on the right.
3. **Decoupling capacitors:** C12–C17 and C19, **each within 2 mm of its own power pin** (1, 13, 19, 32, 48, 64). C18 goes by pin 7. **This rule matters most for the board working reliably.** Put them on the top side, right beside the pin, with their GND end facing outward for a via down to the GND plane.
4. **Y2 + C10/C11** go within about 5 mm of pins 5/6, with nothing else routed between them and the chip. The same goes for **Y1** next to U3 pins 3/4.
5. **U3 hub** goes between J1 and the MCU, so the 480 Mbit/s upstream pair (`USB_DP/DN`) is **as short as possible**: ideally under 20 mm, and on one layer with no vias.
6. **U4 CH343** goes next to U3, toward the MCU's PA9/PA10 side. C4–C9 go next to their pins on U3/U4.
7. **U2 AMS1117** goes near J1, with C1 at its input and C2/C3 at its output. Leave copper area around its tab: it's what spreads the heat (worst case 0.35 W).
8. **J2 microSD** sits on the bottom edge with the card opening facing out. **U6 PSRAM** goes near MCU pins 20–23 (PA4–PA7, bottom-left). C20 sits on U6 pin 8, and C21/C22 next to J2 pin 4. R7–R14 go between the chips and their pins.
9. **J4** goes along the left edge and **J5** along the right edge. Pin 1 (square pad) at the top, 2.54 mm from the edge.
10. **Buttons** go along the bottom-left edge so you can press them. **LEDs D1–D4** go somewhere visible, not under a header.
11. **Test points TP1–TP7** go where a probe can reach, near what they measure: TP1/TP2 near J1/F1, TP3 near U2, TP6 near U3, and TP7 near U4. Put **TP4/TP5 (GND)** at opposite ends of the board.

**Rotation tip:** keep chips at 0/90/180/270° so JLC's pick-and-place data stays simple.

## Done-check
- [ ] 50 × 50 Edge.Cuts with 1 mm corners, plus 4 M2.5 holes
- [ ] 4 layers, 1.6 mm; In1 = GND, In2 = power
- [ ] Constraints and the 3 net classes entered
- [ ] All parts on **top** (F.Cu), none hanging off the board (except the USB-C lip)
- [ ] Every decoupling capacitor ≤ 2 mm from its pin; crystals next to their chips
- [ ] No courtyard overlaps: run **Inspect → Design Rules Checker** and look for "Courtyards overlap"

Save and tell me. I'll run DRC from the command line and check the placement: decap distances, crystal distances, USB pair lengths, and anything sitting off the board. Then Part 2 is routing.
