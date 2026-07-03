# Functional Specification & Application Guide
**Customized STM32F103C8Tx Development Board**
**Author:** Vishal Meyyappan R 

---

## 1. System Overview

This customized STM32F103C8Tx development board is architected specifically for robust edge-computing, embedded AI data collection, and autonomous system control. By integrating reliable power steering, high-speed storage, and filtered analog inputs, the board bridges the gap between raw physical environments and downstream data processing. 

This document details the functional capabilities of the hardware subsystems and outlines practical implementation strategies for firmware development.

---

## 2. Integrated MicroSD Storage & Data Logging

The integration of a MicroSD card socket (via the SPI1 peripheral) is the core feature enabling untethered, high-volume data acquisition. This transforms the board from a simple microcontroller into a standalone edge-logging node.

### Functional Mechanics
*   **Protocol:** The board utilizes the SPI (Serial Peripheral Interface) protocol, communicating natively at 3.3V, eliminating the latency and failure points of external level shifters.
*   **Signal Integrity:** 10kΩ hardware pull-up resistors on the CS, DAT1, and DAT2 lines ensure the SD card remains completely stable and deselected during MCU power-on reset, preventing filesystem corruption.
*   **Power Stability:** A dedicated 100nF decoupling capacitor is placed millimeters from the card's VDD pin to absorb the high-current transients that occur during flash memory write cycles.

### Embedded AI & Sensor Applications
*   **Kinematic & Biometric Logging:** The SD card is ideal for sampling high-frequency continuous data streams. When interfacing with analog or I2C peripherals—such as IMU (Inertial Measurement Units), EMG (Electromyography), or Flex sensors—the STM32 can aggregate this data and write it directly to the SD card.
*   **Training Data Generation:** Firmware can be configured via the FatFs middleware to format sensor payloads into standard `.csv` or `.txt` files. This allows the board to operate autonomously in the field, gathering massive localized datasets required for training embedded Machine Learning (TinyML) models or neural networks.
*   **State Machine Logging:** For autonomous systems (e.g., self-regulating thermal systems or robotics), the SD card acts as a "black box," logging state changes, error codes, and telemetry history for post-deployment diagnostics.

---

## 3. Uninterruptible Dual Power Architecture

To support field deployments where power reliability is critical, the board features a "Diode-OR" power steering topology. 

### Functional Mechanics
*   **Automatic Source Selection:** The system accepts power simultaneously from a USB Micro-B connection (5V) and a DC Barrel Jack (e.g., 9V-12V battery pack or wall adapter).
*   **Protection:** Two discrete Schottky diodes block reverse current. The source with the higher voltage automatically assumes the load. 
*   **Hot-Swapping:** A user can plug in a USB cable to extract SD card data or flash firmware, and then disconnect the USB without resetting the MCU, provided the barrel jack remains powered.
*   **Thermal Considerations:** The onboard AMS1117-3.3V LDO regulator features a thermal tab and 22µF input/output capacitance to handle the voltage drop from higher barrel-jack voltages without introducing ripple to the 3.3V rail.

---

## 4. Analog Signal Integrity (VDDA Filtering)

Microcontrollers operating in environments with heavy digital switching (like SPI writing to an SD card) often suffer from electrical noise, which ruins analog sensor readings.

### Functional Mechanics
*   **LC Low-Pass Filter:** The board isolates the analog power rail (`VDDA`) from the main digital power rail (`+3.3V`) using a 120Ω Ferrite Bead (`FB1`).
*   **Capacitance:** The isolated rail is heavily bypassed with 10nF and 1µF capacitors to shunt high-frequency noise to ground.
*   **Application:** This ensures that when utilizing the STM32's internal 12-bit Analog-to-Digital Converter (ADC) for sensitive resistive measurements (like force or flex sensors), the readings remain highly accurate and free of digital jitter, reducing the need for aggressive software filtering.

---

## 5. Expansion and Communication Interfaces

The board breaks out specific peripheral buses to standard 2.54mm pitch headers to interface with external hardware arrays.

### I2C2 Bus (PB10, PB11)
*   **Hardware Pull-ups:** The board includes integrated 1.5kΩ pull-up resistors on the SDA and SCL lines. 
*   **Application:** Ready immediately for multi-device I2C chains, such as connecting multiple OLED displays, IMUs, or environmental sensors without requiring external breadboard resistors.

### UART1 Bus (PA9, PA10)
*   **Application:** Dedicated asynchronous serial communication. Ideal for interfacing with GPS modules, Bluetooth transceivers, or transmitting real-time AI inference results to a host computer.

### SWD Interface (PA13, PA14)
*   **Application:** Standard Serial Wire Debug interface for use with ST-LINK programmers. Allows for live register monitoring, step-through debugging, and direct memory access during complex algorithm development.

---

## 6. Pin Mapping Reference

| Peripheral Function | STM32 Pin | Board Net Label | Notes |
| :--- | :--- | :--- | :--- |
| **SPI1 CS** | PA4 | `SD_CS` | Includes 10kΩ Pull-up |
| **SPI1 SCK** | PA5 | `SD_SCK` | SD Card Clock |
| **SPI1 MISO** | PA6 | `SD_MISO` | Data from SD to MCU |
| **SPI1 MOSI** | PA7 | `SD_MOSI` | Data from MCU to SD |
| **I2C2 SCL** | PB10 | `I2C2_SCL` | Includes 1.5kΩ Pull-up |
| **I2C2 SDA** | PB11 | `I2C2_SDA` | Includes 1.5kΩ Pull-up |
| **UART1 TX** | PA9 | `USART1_TX` | Serial Transmit |
| **UART1 RX** | PA10 | `USART1_RX` | Serial Receive |
| **USB D-** | PA11 | `USB_D-` | |
| **USB D+** | PA12 | `USB_D+` | Includes 1.5kΩ Pull-up for device detection |
| **SWDIO** | PA13 | `SWDIO` | Debug Data |
| **SWCLK** | PA14 | `SWCLK` | Debug Clock |

---

## 7. Firmware Implementation Workflow

To fully utilize the hardware capabilities, developers should follow this standard initialization workflow utilizing STM32 HAL and FatFs:

1.  **System Clock Configuration:** Initialize the HSE using the external 3.2x2.5mm crystal, scaling the PLL to run the core at 72MHz.
2.  **SPI Initialization:** Configure SPI1 in Full-Duplex Master mode. Initialize the bus at a low baud rate (≤400kHz) to meet SD card initialization standards, then dynamically scale the prescaler up (e.g., 9-18MHz) for high-speed data streaming.
3.  **FatFs Middleware:** Enable the FatFs library. Mount the volume using `f_mount()`.
4.  **Logging Loop:**
    *   Open a target file with `FA_OPEN_APPEND | FA_WRITE`.
    *   Sample ADC or I2C sensors.
    *   Format data strings using `sprintf()`.
    *   Write the buffer using `f_write()`.
    *   Critical: Close the file using `f_close()` or sync it using `f_sync()` at regular intervals to guarantee data is flushed from SRAM to physical flash memory in the event of an abrupt power loss.
