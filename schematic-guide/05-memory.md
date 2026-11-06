# Section 5: Memory (8MB PSRAM + microSD)

**What it does:** gives the OS room to grow.
- **PSRAM (U6):** 8 MB of extra RAM on SPI1. The CH32V307 only has 64 KB of its own. Your OS can use it as a heap, or as the memory for loaded programs.
- **microSD (J2):** storage on the chip's SDIO bus (4-bit). A FAT filesystem goes here, so the OS can load programs from files.

Stock re-checked 2026-11-06: APS6404L 2,927 · Molex microSD 48,637 · 10k 16M · 100nF 34M · 10uF 10M.

## Parts in this section

| Ref | Value | Symbol | Footprint | LCSC |
|---|---|---|---|---|
| U6 | APS6404L-3SQR-SN | `Memory_RAM:APS6404L-3SQRx-SN` | `Package_SO:SOIC-8_3.9x4.9mm_P1.27mm` | C5333729 |
| C20 | 100nF | `Device:C` | `Capacitor_SMD:C_0402_1005Metric` | C1525 |
| R7 | 10k (CE# pull-up) | `Device:R` | `Resistor_SMD:R_0402_1005Metric` | C25744 |
| R8, R9 | 10k (SIO2/SIO3 pull-downs) | `Device:R` | `Resistor_SMD:R_0402_1005Metric` | C25744 |
| J2 | Molex 104031-0811 | `Connector:Micro_SD_Card_Det2` | `Connector_Card:microSD_HC_Molex_104031-0811` | C585350 |
| C21 | 10uF | `Device:C` | `Capacitor_SMD:C_0603_1608Metric` | C19702 |
| C22 | 100nF | `Device:C` | `Capacitor_SMD:C_0402_1005Metric` | C1525 |
| R10–R14 | 10k (SD pull-ups ×5) | `Device:R` | `Resistor_SMD:R_0402_1005Metric` | C25744 |

**BOM changes:** +2 10k resistors (R8, R9) and +1 100nF capacitor (C22). All are basic parts, so there's no fee (+$0.01). Section 6's LED resistors are renumbered R15/R16.

## Key decisions (from the datasheets)

1. **PSRAM in plain SPI mode** (1 bit wide). The CH32V307 has no quad-SPI controller, so SIO2 and SIO3 are unused.
2. **CE# pull-up (R7).** The datasheet (§7) says CE# must stay **high** for the first 150 µs after power-up. The MCU's pins float until your code sets them up, so a pull-up holds CE# high during that time.
3. **SIO2/SIO3 pull-downs (R8, R9).** The same section says SIO pins should stay **low** at power-up. They're never driven in SPI mode, so a 10k to GND keeps them from floating. *Not* wired straight to GND: if a stray command ever put the chip into quad mode, it would drive those pins against a dead short.
4. **SD pull-ups on CMD and DAT0–DAT3 (R10–R14).** The SD spec requires them. The card starts up with only CMD active, so the data lines would float without them. **No pull-up on CLK.**
5. **Card detect:** pin 9 → `SD_DET`, pin 10 → GND. Your code turns on the MCU's internal pull-up on PC7, so you don't need another resistor. Card inserted = switch closed = **PC7 reads low**. (If it reads the other way on the real board, flip one line of code.)

## Global labels in this section

Shapes are from **this section's** point of view (Input = comes in from the MCU).

| Label | Shape | Connects to | Other end |
|---|---|---|---|
| `PSRAM_CS` | Input | U6 pin 1 CE# | MCU PA4 |
| `SPI1_SCK` | Input | U6 pin 6 SCLK | MCU PA5 |
| `SPI1_MOSI` | Input | U6 pin 5 SI/SIO0 | MCU PA7 |
| `SPI1_MISO` | **Output** | U6 pin 2 SO/SIO1 | MCU PA6 |
| `SD_CLK` | Input | J2 pin 5 CLK | MCU PC12 |
| `SD_CMD` | Bidirectional | J2 pin 3 CMD | MCU PD2 |
| `SD_D0` | Bidirectional | J2 pin 7 DAT0 | MCU PC8 |
| `SD_D1` | Bidirectional | J2 pin 8 DAT1 | MCU PC9 |
| `SD_D2` | Bidirectional | J2 pin 1 DAT2 | MCU PC10 |
| `SD_D3` | Bidirectional | J2 pin 2 DAT3/CD | MCU PC11 |
| `SD_DET` | **Output** | J2 pin 9 DET_B | MCU PC7 |

Place them on the **left edge of box 5, pointing left** toward the MCU. All of these labels already exist, so pick them from the dropdown.

## Circuit

```
                 +3V3                                       +3V3
                  │                                           │
        R7 10k ───┤                C20 100nF ─┐     R10–R14 10k (one per line)
                  │                           │       │  │  │  │  │
PSRAM_CS ─────────┴── 1 CE#      VDD 8 ───────┴ +3V3  │  │  │  │  │
SPI1_MISO ◄────────── 2 SO/SIO1  SIO3 7 ── R9 10k ─ GND
                 ┌─── 3 SIO2     SCLK 6 ◄──── SPI1_SCK
       R8 10k ───┘    4 VSS      SI   5 ◄──── SPI1_MOSI
          │             │
         GND           GND

        J2 microSD
SD_D2  ◄─► 1 DAT2     ──── R10 ─ +3V3
SD_D3  ◄─► 2 DAT3/CD  ──── R11 ─ +3V3
SD_CMD ◄─► 3 CMD      ──── R12 ─ +3V3
   +3V3 ── 4 VDD      ── C21 10uF + C22 100nF to GND
SD_CLK ──► 5 CLK          (no pull-up)
    GND ── 6 VSS
SD_D0  ◄─► 7 DAT0     ──── R13 ─ +3V3
SD_D1  ◄─► 8 DAT1     ──── R14 ─ +3V3
SD_DET ◄── 9 DET_B
    GND ── 10 DET_A
    GND ── SH (shield, all 4 tabs = 1 symbol pin)
```

## Connections, pin by pin

**U6 (APS6404L PSRAM)**
- Pin 8 `VDD` → +3V3, with C20 (100nF) to GND right at the pin
- Pin 4 `VSS` → GND
- Pin 1 `CE#` → `PSRAM_CS`, **and** R7 10k to +3V3
- Pin 6 `SCLK` → `SPI1_SCK`
- Pin 5 `SI/SIO0` → `SPI1_MOSI` (MCU out → PSRAM in)
- Pin 2 `SO/SIO1` → `SPI1_MISO` (PSRAM out → MCU in)
- Pin 3 `SIO2` → R8 10k → GND
- Pin 7 `SIO3` → R9 10k → GND

*MOSI/MISO don't cross like UART does. MOSI is "master out, slave in" on both chips, so the names match end to end.*

**J2 (microSD)**
- Pin 4 `VDD` → +3V3, with C21 (10uF) and C22 (100nF) to GND. Cards draw current spikes when writing.
- Pin 6 `VSS` → GND
- Pin 5 `CLK` → `SD_CLK`
- Pin 3 `CMD` → `SD_CMD` + R12 10k to +3V3
- Pins 7, 8, 1, 2 (`DAT0`–`DAT3`) → `SD_D0`–`SD_D3`, each with a 10k pull-up (R13, R14, R10, R11)
- Pin 9 `DET_B` → `SD_DET`
- Pin 10 `DET_A` → GND
- `SH` (shield) → GND

## Power budget: unchanged
PSRAM up to 40 mA and the SD card up to 100 mA (short spikes about 200 mA) were already counted in Section 1.

## Bring-up test (after boards arrive)
1. **PSRAM:** send Reset Enable (`0x66`), then Reset (`0x99`), then Read ID (`0x9F`). The first ID byte should be `0x0D` (AP Memory). Then write a pattern to all 8 MB, read it back, and compare. That's your first memory test.
2. **SD card:** with no card inserted, PC7 reads high. Insert a card and it reads low. Then initialize the card over SDIO (CMD0 → CMD8 → ACMD41) and read sector 0.

## Done-check
- [ ] R7 pull-up on CE#; R8/R9 pull-downs on SIO2/SIO3
- [ ] MOSI → pin 5, MISO → pin 2 (not swapped)
- [ ] Pull-ups on CMD and all four DAT lines, **none on CLK**
- [ ] DET_B → `SD_DET`, DET_A → GND, SH → GND
- [ ] C20 at the PSRAM; C21 + C22 at the SD socket
- [ ] `LCSC` field and footprint on every part

Save and tell me. I'll run ERC and check every pin. All the PSRAM and SD "label connected to only one pin" warnings should disappear.
