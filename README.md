# Customized STM32 Development Board

**Designer:** Vishal Meyyappan R
**Design Tool:** KiCad 9.0
**MCU:** STM32F103C8T6 (LQFP-48)

A compact, breakout-style STM32F103 development board built around a "Blue Pill"-class microcontroller, with dual power input (USB or barrel jack), onboard microSD storage for data logging, and broken-out USART/I2C/SWD headers for peripherals and debugging.

---

## 1. Overview

This board is designed as a general-purpose sensor data acquisition and logging platform. The core idea: read sensor values over I2C or USART, and persist them to a microSD card in real time, so the board can operate as a standalone data logger without needing a permanent connection to a PC. It can be powered either from USB (during development/debugging) or from an external DC supply via the barrel jack (for field/standalone deployment), with both sources safely combined so either one can be present without conflict.

### Key features

- STM32F103C8T6 microcontroller — 72 MHz Cortex-M3, 64KB Flash, 20KB RAM
- Dual power input: USB Micro-B **or** 7–12V DC barrel jack, diode-ORed together
- Onboard AMS1117-3.3 LDO regulator (3.3V logic/power rail)
- microSD card slot (SPI mode) for onboard data storage/logging
- USART1 header for serial communication (e.g. GPS, external logger, PC terminal)
- I2C2 header for sensor peripherals (IMUs, environmental sensors, RTCs, displays, etc.)
- SWD header for programming and debugging (ST-Link compatible)
- Reset button and status LED
- USB D+/D- routed directly to the MCU for native USB device applications (e.g. CDC virtual COM port, DFU bootloader use)
- 4x M2 mounting holes for mechanical enclosure mounting

---

## 2. Functional Blocks

### 2.1 Power Supply

The board accepts power from **either** of two sources, which are combined onto a single protected rail before regulation:

| Source | Connector | Voltage | Notes |
|---|---|---|---|
| USB | J1 (USB Micro-B) | 5V (VBUS) | Also carries D+/D- to the MCU for USB device functionality |
| DC Barrel Jack | J5 | 7–12V (typical wall adapter) | Center pin positive |

Both inputs pass through a dedicated Schottky diode (D2 for the barrel jack, D3 for USB VBUS) before joining onto a common rail. This "power-ORing" arrangement means:
- Either source can be connected and power the board independently.
- If both are connected simultaneously, the board draws from whichever is at the higher potential, without one source backfeeding into the other's connector — protecting a USB host (like a laptop) from being driven by the barrel jack supply, and vice versa.
- Reverse-polarity protection is inherent: if the barrel jack is wired backwards, the diode simply blocks conduction rather than damaging the board.

The combined rail feeds **U2 (AMS1117-3.3)**, a linear regulator that steps the input down to a clean 3.3V rail (`+3.3V`), which powers the MCU core, I/O, and the microSD card. `FB1` (a 120Ω ferrite bead) filters high-frequency noise on the USB VBUS path specifically, and `D1` (LED) with `R3` provides a visual power-on indicator.

A separate `+3.3VA` net is also present for the MCU's analog supply (`VDDA`), isolated with `C7`/`C8` to reduce noise coupling into the ADC/analog peripherals from the digital 3.3V rail.

### 2.2 Microcontroller (U1 — STM32F103C8T6)

The MCU is clocked from an 8 MHz external crystal (`Y1`, `Crystal_GND24`) with `C10`/`C11` (10pF) load capacitors, feeding the HSE (High-Speed External) oscillator input. `SW1` provides a manual reset (NRST), debounced by `C9` and pulled up by `R1`.

Boot mode selection (`BOOT0`) is available on the board for entering the built-in USB/UART bootloader if needed, alongside the SWD interface for hardware debugging.

### 2.3 USB Interface (J1)

A USB Micro-B connector routes `D+`/`D-` directly to the MCU's native USB peripheral pins (`PA11`/`PA12`), enabling USB device applications such as a virtual COM port (CDC), custom HID device, or DFU firmware updates. `R2` (1.5kΩ) pulls `USB_D-` up to 3.3V, a common requirement for the MCU to be recognized as a full-speed USB device without needing to bit-bang the pull-up in firmware.

### 2.4 DC Barrel Jack Input (J5)

For standalone/field operation where USB isn't available (e.g. powered by a wall adapter or battery pack), `J5` accepts a standard center-positive barrel connector. `C14`/`C15` provide bulk and high-frequency decoupling right at the input before the reverse-polarity diode `D2`.

### 2.5 microSD Card Storage (J6)

This is the board's primary logging mechanism. The microSD slot is wired in **SPI mode** (rather than native 4-bit SD mode, which this MCU's hardware doesn't support), using the MCU's SPI1 peripheral:

| J6 Pin | Native SD Function | SPI Mode Function | STM32 Pin |
|---|---|---|---|
| 2 (DAT3/CD) | Data line 3 / Card Detect | Chip Select (CS) | PA4 |
| 3 (CMD) | Command line | MOSI (DI) | PA7 |
| 5 (CLK) | Clock | SCK | PA5 |
| 7 (DAT0) | Data line 0 | MISO (DO) | PA6 |
| 1 (DAT2), 8 (DAT1) | Data lines 1–2 | Unused in SPI mode | Pulled up to 3.3V (R6–R8) |
| 4 (VDD) | Power | Power | +3.3V |
| 6 (VSS) | Ground | Ground | GND |
| 9 (SHIELD) | — | Mechanical shield | GND |

`C16` (100nF) decouples the card's local supply. A pull-up on `CS` (`PA4`) ensures the card remains deselected during MCU power-up/reset, before firmware has configured the pin as an active output — preventing bus contention during boot.

**Intended use case:** the board reads sensor data (e.g. from a device on the I2C2 header) at a fixed interval, formats it (e.g. as a CSV row with a timestamp), and writes it to a file on the SD card using a FAT filesystem library (FatFs) running over this SPI interface. This lets the board function as a self-contained data logger — sensor readings accumulate on the card over time and can be retrieved simply by removing the card and reading it on a PC, without needing a live connection to the board.

### 2.6 Communication Headers

| Header | Signals | Typical Use |
|---|---|---|
| J2 | SWDIO, SWCLK, 3.3V, GND | Programming/debugging via ST-Link or compatible SWD probe |
| J3 | USART1_TX, USART1_RX, 3.3V, GND | Serial communication — external modules (GPS, Bluetooth/BLE, PC terminal), or a second logging path |
| J4 | I2C2_SCL, I2C2_SDA, 3.3V, GND | Sensor peripherals — IMUs, environmental sensors (temperature/humidity/pressure), RTCs, OLED displays, etc. |

`R4`/`R5` (1.5kΩ) provide I2C bus pull-ups on the SDA/SCL lines.

### 2.7 Status & Control

- `D1` (LED) — power-on indicator, active whenever the 3.3V rail is present
- `SW1` — manual reset button (NRST)

### 2.8 Mechanical

Four M2 mounting holes (`H1`–`H4`) are provided at the board corners for enclosure mounting.

---

## 3. Typical Application: Sensor Data Logger

A representative use case this board is designed around:

1. Connect a sensor (e.g. an I2C temperature/humidity sensor, or an IMU) to the **J4 (I2C2)** header.
2. Firmware on the STM32 periodically polls the sensor over I2C.
3. Each reading is timestamped and formatted as a line of text (e.g. CSV: `timestamp,value1,value2`).
4. The line is written to a log file on the microSD card via **SPI1 + FatFs**.
5. The card can be removed and read on any PC, or the board can optionally stream the same data live over **J3 (USART1)** for real-time monitoring during development.

This makes the board suitable for applications like environmental monitoring stations, unattended data collection, portable measurement tools, or any project needing local, non-volatile storage of time-series sensor data without relying on cloud connectivity.

---

## 4. Power Architecture Summary

```
USB Micro-B (5V)  ──▶ D3 ──┐
                            ├──▶ Shared VIN net ──▶ AMS1117-3.3 (U2) ──▶ +3.3V rail ──▶ MCU, SD card, peripherals
Barrel Jack (7-12V) ─▶ D2 ──┘
```

Both sources are diode-ORed so either can power the board independently; the AMS1117-3.3 regulates whichever is present down to a clean 3.3V for the rest of the system.

---

## 5. Bill of Materials

| Reference | Qty | Value | Footprint | Datasheet |
|---|---|---|---|---|
| C1, C2, C3, C4, C9, C15, C16 | 7 | 100n | Capacitor_SMD:C_0402_1005Metric | ~ |
| C5, C14 | 2 | 10u | Capacitor_SMD:C_0603_1608Metric | ~ |
| C6 | 1 | 10n | Capacitor_SMD:C_0402_1005Metric | ~ |
| C7, C8 | 2 | 1u | Capacitor_SMD:C_0402_1005Metric | ~ |
| C10, C11 | 2 | 10p | Capacitor_SMD:C_0402_1005Metric | ~ |
| C12, C13 | 2 | 22u | Capacitor_SMD:C_0805_2012Metric | ~ |
| D1 | 1 | LED | LED_SMD:LED_0603_1608Metric | ~ |
| D2, D3 | 2 | D_Schottky | Diode_SMD:D_SMA | ~ |
| FB1 | 1 | 120R | Inductor_SMD:L_0603_1608Metric | ~ |
| H1, H2, H3, H4 | 4 | MountingHole | MountingHole:MountingHole_2.2mm_M2 | ~ |
| J1 | 1 | USB_B_Micro | Connector_USB:USB_Micro-B_Wuerth_629105150521 | ~ |
| J2, J3, J4 | 3 | Conn_01x04_Pin | Connector_PinSocket_2.54mm:PinSocket_1x04_P2.54mm_Vertical | ~ |
| J5 | 1 | Barrel_Jack | Connector_BarrelJack:BarrelJack_Horizontal | ~ |
| J6 | 1 | Micro_SD_Card | Connector_Card:microSD_HC_Wuerth_693072010801 | [Datasheet](https://www.we-online.com/components/products/datasheet/693072010801.pdf) |
| R1 | 1 | 10K | Resistor_SMD:R_0402_1005Metric | ~ |
| R2, R3, R4, R5 | 4 | 1k5 | Resistor_SMD:R_0402_1005Metric | ~ |
| R6, R7, R8 | 3 | 10k | Resistor_SMD:R_0402_1005Metric | ~ |
| SW1 | 1 | SW_SPDT | Button_Switch_SMD:SW_SPDT_PCM12 | ~ |
| U1 | 1 | STM32F103C8Tx | Package_QFP:LQFP-48_7x7mm_P0.5mm | [Datasheet](https://www.st.com/resource/en/datasheet/stm32f103c8.pdf) |
| U2 | 1 | AMS1117-3.3 | Package_TO_SOT_SMD:SOT-223-3_TabPin2 | [Datasheet](http://www.advanced-monolithic.com/pdf/ds1117.pdf) |
| Y1 | 1 | Crystal_GND24 | Crystal:Crystal_SMD_3225-4Pin_3.2x2.5mm | ~ |

**Total unique references:** 24 | **Total placed components:** ~50 (excluding mounting holes)

---

## 6. Firmware Notes

- **Toolchain:** STM32CubeIDE (or STM32CubeMX + preferred IDE/toolchain)
- **SD card access:** SPI1 peripheral + FatFs middleware (SD-over-SPI driver); recommend low clock speed (~400 kHz) during card initialization, then increasing to a few MHz for normal read/write operations
- **I2C sensor communication:** I2C2 peripheral (`PB10`/`PB11`), 3.3V logic
- **Serial/debug output:** USART1 (`PA9`/`PA10`), standard TTL-level UART, 3.3V logic
- **Debugging:** Standard SWD (`PA13`/`PA14`) — compatible with ST-Link V2/V3 or any SWD-capable probe
- **USB:** Native USB device peripheral (`PA11`/`PA12`) — usable for CDC virtual COM port, custom HID, or DFU bootloader entry

---

<img width="1136" height="670" alt="image" src="https://github.com/user-attachments/assets/540751cd-d8b3-4893-af91-e856eea13220" />

**PCB Editor**

<img width="1154" height="848" alt="image" src="https://github.com/user-attachments/assets/1d5ad453-f448-4b5e-89cd-eae1b3c2d303" />

**3D VIEW**

I have attached Schematic sheet as output.pdf

---

## Designed by Vishal Meyyappan R
