# FAKE POD NANO PRO (ESP32-S31) V1 Production: Developer Hardware Guide

[简体中文](hardware_v1.md) | **English**

This document describes the onboard peripheral connections and reference development parameters of the V1 production hardware, for users who want to write their own firmware.

All pin numbers are ESP32-S31 **GPIO numbers**, not chip package pins or display FPC pins. Signal directions are relative to the MCU. This document applies to hardware version **1 (V1 production)** only.

## 1. Hardware Overview

| Item | Specification |
| --- | --- |
| MCU | ESP32-S31 |
| ESP-IDF target | `esp32s31` |
| Display | AMOLED, CO5300 controller, QSPI interface |
| Visible resolution | 460 × 460 pixels |
| Touch controller | CST820, I²C |
| Power management | AXP2101, I²C |
| Audio DAC | CS43131, I²S audio input, I²C control |
| DAC onboard crystal | 24.576 MHz |
| Microphone interface | PDM |
| IMU | QMI8658, I²C |
| RTC | PCF85063, I²C |
| Memory card interface | SDMMC, 4-bit |
| Buttons | One dedicated GPIO button, one AXP2101 power button |

## 2. Display GPIOs and Parameters

| Signal | GPIO | Direction | Notes |
| --- | --- | --- | --- |
| VCI_EN | 44 | Output | AMOLED power enable, active high |
| RESET | 36 | Output | Display reset |
| CS | 37 | Output | QSPI chip select |
| SCK | 38 | Output | QSPI clock |
| DATA0 | 39 | Data | QSPI data line 0 |
| DATA1 | 40 | Data | QSPI data line 1 |
| DATA2 | 42 | Data | QSPI data line 2 |
| DATA3 | 43 | Data | QSPI data line 3 |
| TE | 35 | Input | Tearing-effect sync signal |

| Development parameter | Value |
| --- | --- |
| Interface | 4-line data QSPI, separate CS and SCK |
| Reference SPI controller | SPI2 |
| SPI mode | Mode 0 |
| Reference QSPI clock | 80 MHz |
| Pixel format | RGB565, 16 bit/pixel, RGB order |
| Display window offset | X = 10, Y = 0 |
| Logical coordinate range | X: 0–459; Y: 0–459 |
| Corresponding display RAM window | X: 10–469; Y: 0–459 (inclusive) |
| One RGB565 frame of pixel data | 423,200 bytes (460 × 460 × 2) |

Notes:

- Use a driver that supports the CO5300 QSPI protocol; this interface has no separate D/C GPIO.
- The window offset must be applied every time the draw window is set. Setting it once at initialization does not keep later refreshes in the right position.
- At power-up, you can pull VCI_EN low for 10 ms, then drive it high and wait 120 ms before resetting and initializing the controller. The display module's requirements take precedence for the full init sequence.
- AMOLED brightness is set through controller commands; VCI_EN only enables the supply.
- TE is used to synchronize display writes; choose the trigger edge based on the actual TE waveform.
- 80 MHz is a reference development setting, not a guaranteed value for any wiring length; re-verify if an external cable changes.

## 3. I²C Buses and Device Addresses

### Bus Connections

| Bus | SDA GPIO | SCL GPIO | Reference clock | Onboard devices |
| --- | --- | --- | --- | --- |
| I²C0 | 48 | 47 | 400 kHz | CST820, AXP2101, QMI8658, PCF85063 |
| I²C1 | 55 | 54 | 400 kHz | CS43131 |

### Device Addresses

All addresses below are **7-bit I²C addresses** and can be used directly in ESP-IDF drivers without shifting left.

| Device | Bus | Address | Purpose |
| --- | --- | --- | --- |
| CST820 | I²C0 | `0x15` | Touch data |
| AXP2101 | I²C0 | `0x34` | Power, charging, battery status and power button |
| PCF85063 | I²C0 | `0x51` | Real-time clock |
| QMI8658 | I²C0 | `0x6B` | Acceleration and angular rate data |
| CS43131 | I²C1 | `0x30` | DAC register configuration |

Devices on the same bus share SDA/SCL. Use a single bus instance per bus and coordinate concurrent access; do not initialize the same pins again for each device.

## 4. Touch GPIOs

| Signal | GPIO | Direction | Notes |
| --- | --- | --- | --- |
| SDA | 48 | Bidirectional | Shared I²C0 |
| SCL | 47 | Output | Shared I²C0 |
| INT | 45 | Input | Touch interrupt, active low |
| RESET | 46 | Output | Touch reset, active low |

Touch logical coordinates map to the 460 × 460 display area. When changing the display orientation, rotate or mirror the touch coordinates to match.

## 5. Power Management and Buttons

| Signal or button | GPIO / connection | Direction | Notes |
| --- | --- | --- | --- |
| PMU SDA | GPIO48 | Bidirectional | Shared I²C0 |
| PMU SCL | GPIO47 | Output | Shared I²C0 |
| PMU INT | GPIO49 | Input | AXP2101 interrupt; trigger on falling edge, then read the interrupt status |
| Bottom button | GPIO61 | Input | Low when pressed, pull-up configurable |
| Top power button | AXP2101 power key input | — | Read button events through the PMU; no dedicated MCU GPIO |

Confirmed supply parameters:

| Power output | Voltage |
| --- | --- |
| AXP2101 DCDC1 | 3.3 V by default |
| AXP2101 ALDO1 | 1.8 V |

When writing your own firmware, establish PMU communication and configure the required power outputs before initializing the corresponding peripherals. Short-press events and long-press power on/off behavior of the top power button are configured in the AXP2101; after reading events, clear the interrupt status as the PMU requires.

This document does not provide the full mapping of loads to power rails. Do not change a rail's voltage or on/off state based on the output name alone.

## 6. Audio DAC: CS43131

| Signal | GPIO | Direction | Notes |
| --- | --- | --- | --- |
| BCLK / BCK | 52 | Output | I²S bit clock |
| LRCLK / WS | 51 | Output | I²S word select |
| DIN | 50 | Output | MCU audio data out, to DAC data input |
| RESET | 53 | Output | DAC reset; pull low then high during initialization |
| Control SDA | 55 | Bidirectional | I²C1, address `0x30` |
| Control SCL | 54 | Output | I²C1 |
| MCU MCLK | No GPIO assigned | — | The DAC uses the onboard 24.576 MHz crystal |

| Reference parameter | Value |
| --- | --- |
| I²S controller | I²S1, TX |
| Master/slave | MCU provides BCLK and WS |
| Data format | Standard I²S (Philips), stereo, MSB first |
| Reference sample rate | 48 kHz |
| Reference valid data width | 24 bit |
| Reference slot width | 32 bit |
| Higher sample rate reference | 96 kHz; DAC clocking and I²S must be configured to match |

With two channels and 32-bit slots, BCLK = sample rate × 2 × 32: 3.072 MHz at 48 kHz and 6.144 MHz at 96 kHz.

The DAC control interface and the audio data interface must be initialized separately. 24-bit valid data and 32-bit slots are different parameters; check both the data alignment and the DAC interface format. 96 kHz is a development reference value, not the CS43131 chip's upper limit.

## 7. PDM Microphone

| Signal | GPIO | Direction | Notes |
| --- | --- | --- | --- |
| PDM DATA | 2 | Input | Microphone data |
| PDM CLK | 6 | Output | PDM clock provided by the MCU |

The microphone uses PDM RX on I²S0, separate from I²S1 used by the DAC. Use a PDM RX configuration, not a standard three-wire BCLK/WS/DIN I²S configuration. This document does not specify the microphone part number, the channel-select level or the rated PDM clock range.

## 8. SDMMC Memory Card

| Signal | GPIO | Direction |
| --- | --- | --- |
| D0 | 20 | Bidirectional |
| D1 | 21 | Bidirectional |
| D2 | 22 | Bidirectional |
| D3 | 23 | Bidirectional |
| CLK | 24 | Output |
| CMD | 25 | Bidirectional |

- 4-bit SDMMC interface, 40 MHz reference working clock.
- The reference configuration does not use UHS modes.
- The SDMMC driver performs low-speed identification during initialization, then switches to the working clock.
- No dedicated card-detect GPIO is documented; determine card presence from driver initialization and access results.

## 9. IMU and RTC

The QMI8658 and PCF85063 are both on I²C0 and need no additional SDA/SCL pins.

| Device | Notes |
| --- | --- |
| QMI8658 | Address `0x6B`; provides acceleration and angular rate. No connection from its INT pin to the MCU is documented; use register polling. |
| PCF85063 | Address `0x51`; after first power-up or an oscillator stop, check time validity and set the time again. No connection from its INT/CLKOUT to the MCU is documented. |

## 10. Before You Start

Your development environment must support the `esp32s31` target. The GPIO and peripheral parameters above are for building your own board configuration and do not depend on the preinstalled application.

The actual Flash/PSRAM size and part numbers, USB data line connections, download-mode procedure, display size and FPC pinout, battery specifications and power-rail loads are not defined in this document. For these, refer to the markings on the hardware of your production batch, the schematic or supplementary specifications.

The GPIOs listed here are connected to onboard peripherals; GPIOs not listed should not be assumed to be available for expansion.
