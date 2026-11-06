# Section 4: MCU Core (CH32V307RCT6)

**What it does:** the brain. This section powers the chip, gives it a clock, sets how it boots, and assigns **every one of its 64 pins**: to USB, the console, the PSRAM, the SD card, the LEDs/buttons, or the GPIO headers. Sections 5 and 6 only need to pick up the labels placed here.

Stock re-checked 2026-11-06: CH32V307RCT6 1,877 · 8MHz crystal 511k · 20pF 922k · 100nF 34M · 10uF 10M · 10k 16M.
Pinout cross-checked: KiCad symbol `CH32V30xRxTx` = datasheet Figure 3-1 (CH32V307RCT6, LQFP64M).

## Parts in this section

| Ref | Value | Symbol | Footprint | LCSC |
|---|---|---|---|---|
| U5 | CH32V307RCT6 | `MCU_WCH_RiscV:CH32V30xRxTx` | `Package_QFP:LQFP-64_10x10mm_P0.5mm` | C2988083 |
| Y2 | 8MHz | `Device:Crystal` | `Crystal:Crystal_SMD_5032-2Pin_5.0x3.2mm` | C115962 |
| C10, C11 | 20pF | `Device:C` | `Capacitor_SMD:C_0402_1005Metric` | C1554 |
| C12–C18 | 100nF (×7) | `Device:C` | `Capacitor_SMD:C_0402_1005Metric` | C1525 |
| C19 | 10uF | `Device:C` | `Capacitor_SMD:C_0603_1608Metric` | C19702 |
| R5, R6 | 10k | `Device:R` | `Resistor_SMD:R_0402_1005Metric` | C25744 |

**BOM changes vs. the original plan:** 100nF caps 8 → **7** and 10uF caps 2 → **1**, to match the datasheet's power circuit (Figure 4-1-1: one 0.1uF per supply pin). Section 5's caps renumbered to C20/C21.

## Key decisions (from the datasheet)

1. **USB = PA11/PA12 (USBFS, 12 Mbps).** The built-in bootloader accepts USB on PA11/PA12 *or* PB6/PB7, and serial on PA9/PA10 ([WCH CH32V307 EVT manual](https://www.wch.cn/uploads/file/20220327/1648348521180114.pdf)). We wire only one USB pair to the hub, so the bootloader appears on exactly one port. PB6/PB7 go to a GPIO header instead.
2. **Boot pins:** BOOT0 = 0 → run your program from flash (normal). BOOT0 = 1 and BOOT1 (PB2) = 0 → built-in bootloader. Both get 10k pull-downs; the BOOT button in Section 6 pulls BOOT0 high.
3. **8 MHz external crystal (HSE).** Needed for accurate USB timing. The 20pF caps put the crystal within ~50 ppm of 8 MHz; USB full-speed allows 2,500 ppm.
4. **No 32.768 kHz crystal.** It's only needed for the real-time clock; the OS timer runs off the main clock. PC14/PC15 are left unused.
5. **PSRAM on SPI1, SD card on SDIO.** The CH32V307 has no quad-SPI controller, so the PSRAM runs in plain SPI mode. The SDIO pins are fixed by the chip (PC8–PC12, PD2).

## Power & clock connections

| Pin | Name | Connect to | Cap (to GND, close to the pin) |
|---|---|---|---|
| 1 | VBAT | +3V3 | C12 100nF |
| 13 | VDDA | +3V3 | C13 100nF |
| 19 | VDD | +3V3 | C14 100nF |
| 32 | VIO | +3V3 | C15 100nF |
| 48 | VDD | +3V3 | C16 100nF |
| 64 | VIO | +3V3 | C17 100nF |
| 12 | VSSA | GND | — |
| 18, 31, 47, 63 | VSS | GND | — |
| — | — | C19 10uF between +3V3 and GND (one bulk cap for the whole chip) | — |
| 5 | OSC_IN | Y2 pin 1, C10 (20pF) to GND | |
| 6 | OSC_OUT | Y2 pin 2, C11 (20pF) to GND | |
| 7 | ~RST | label `NRST` + C18 100nF to GND | |
| 60 | BOOT0 | label `BOOT0` + R5 10k to GND | |
| 28 | PB2 (BOOT1) | R6 10k to GND (no label) | |

Schematic tip: draw each 100nF cap right next to its power pin with a short wire to the pin and a GND symbol. That shows which cap belongs to which pin, and you'll mirror that placement on the PCB.

## Signal pins: global labels

Shapes are from the **MCU's point of view** (Output = the MCU drives it). Point each label toward the box it connects to.

**USB & console (to Sections 2 & 3, up)**

| Pin | Name | Label | Shape |
|---|---|---|---|
| 44 | PA11 | `USB_MCU_DN` | Bidirectional |
| 45 | PA12 | `USB_MCU_DP` | Bidirectional |
| 42 | PA9 | `UART1_TX` | **Output** |
| 43 | PA10 | `UART1_RX` | **Input** |

*These four labels already exist. Pick them from the dropdown so the names match exactly.*

**Debug, reset, boot (to Section 6)**

| Pin | Name | Label | Shape |
|---|---|---|---|
| 46 | SWDIO (PA13) | `SWDIO` | Bidirectional |
| 49 | SWCLK (PA14) | `SWCLK` | Input |
| 7 | ~RST | `NRST` | Input |
| 60 | BOOT0 | `BOOT0` | Input |

**PSRAM on SPI1 (to Section 5)**

| Pin | Name | Label | Shape |
|---|---|---|---|
| 20 | PA4 | `PSRAM_CS` | Output |
| 21 | PA5 | `SPI1_SCK` | Output |
| 22 | PA6 | `SPI1_MISO` | Input |
| 23 | PA7 | `SPI1_MOSI` | Output |

**microSD on SDIO (to Section 5)**

| Pin | Name | Label | Shape |
|---|---|---|---|
| 39 | PC8 | `SD_D0` | Bidirectional |
| 40 | PC9 | `SD_D1` | Bidirectional |
| 51 | PC10 | `SD_D2` | Bidirectional |
| 52 | PC11 | `SD_D3` | Bidirectional |
| 53 | PC12 | `SD_CLK` | Output |
| 54 | PD2 | `SD_CMD` | Bidirectional |
| 38 | PC7 | `SD_DET` | Input (card-detect switch) |

**LEDs & user button (to Section 6)**

| Pin | Name | Label | Shape |
|---|---|---|---|
| 26 | PB0 | `LED1` | Output |
| 27 | PB1 | `LED2` | Output |
| 14 | PA0 | `BTN_USER` | Input (PA0 can also wake the chip from standby) |

**GPIO headers (to Section 6). Label = the pin's own name, shape Bidirectional**

| Header | Pin → label | Useful for |
|---|---|---|
| J4 | 15 `PA1` · 16 `PA2` · 17 `PA3` · 41 `PA8` · 50 `PA15` · 55 `PB3` · 56 `PB4` · 57 `PB5` | USART2 (PA2/PA3), timer PWM, general GPIO |
| J5 | 58 `PB6` · 59 `PB7` · 29 `PB10` · 30 `PB11` · 33 `PB12` · 34 `PB13` · 35 `PB14` · 36 `PB15` | I2C1 (PB6/PB7), USART3 / I2C2 (PB10/PB11), SPI2 (PB12–PB15) |

**Unused → no-connect flag (12 pins)**
2 PC13 · 3 PC14 · 4 PC15 · 8 PC0 · 9 PC1 · 10 PC2 · 11 PC3 · 24 PC4 · 25 PC5 · 37 PC6 · 61 PB8 · 62 PB9

Pin count check: 11 power + 2 crystal + RST + BOOT0 + BOOT1 + 4 USB/UART + 2 SWD + 4 SPI + 7 SD + 3 LED/button + 16 header + 12 unused = **64** ✓

## Power budget: unchanged
~30 mA typical, 60 mA worst case at 144 MHz. Already counted in Section 1.

## Bring-up test (after boards arrive)
1. TP3 (+3V3) still 3.25–3.35 V with the MCU fitted.
2. **Bootloader:** hold BOOT, tap RESET, release BOOT. Then:
   ```bash
   wchisp info
   ```
   It should report `CH32V307`. This proves power, reset, and USB through the hub all work, before you've written a line of code.
3. Flash a blink program for LED1 (PB0). If USB flashing misbehaves, the same bootloader also answers over the CH343 serial port (USART1).

## Done-check
- [ ] All 6 supply pins on +3V3, each with its own 100nF; all 5 ground pins on GND; one 10uF bulk
- [ ] Crystal on pins 5/6 with 20pF to GND on each side
- [ ] NRST: 100nF to GND + `NRST` label. BOOT0: 10k to GND + `BOOT0` label. PB2: 10k to GND
- [ ] `UART1_TX` on **PA9**, `UART1_RX` on **PA10** (opposite of the CH343 side, which is correct)
- [ ] `USB_MCU_DN` on PA11, `USB_MCU_DP` on PA12
- [ ] All labels in the tables above with the listed shapes; no-connect flags on the 12 unused pins
- [ ] `LCSC` field on U5, Y2, C10–C19, R5, R6

Save and tell me. I'll run ERC and check all 64 pins. The Section 2/3 warnings (`USB_MCU_*`, `UART1_*`) and the "RXD not driven" error should disappear. New "label connected to only one pin" warnings for the Section 5/6 labels are expected until those sections are drawn.
