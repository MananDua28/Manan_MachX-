# SUGAR CanSAT

## Overview

SUGAR is a high-altitude rocket CanSat competition payload developed for autonomous atmospheric sensing, trajectory tracking, thermal monitoring, and live RF telemetry streaming. The payload operates inside a rocket airframe during ascent, transitions through mission modes during flight, and streams telemetry to a dedicated ground station receiver during descent.

This directory contains the complete flight computer firmware, dedicated ground station receiver firmware, web-based mission control dashboard, packet parsers, simulation and testing tools, and flight datasets for the SUGAR CanSat system.

---

## System Architecture

The SUGAR CanSat architecture establishes an end-to-end data pipeline from physical sensor acquisition to ground visualization:

```mermaid
flowchart TD
    subgraph Flight_System["SUGAR CanSat Payload (In Flight)"]
        SENS["Sensors (BMP388, MPU-6500, 4x LM75, GPS NEO-6M)"] -->|I2C / SPI / UART| FC["Flight Computer (STM32F103C8T6 BluePill)"]
        FC -->|SPI| SD["Onboard SPI MicroSD Logging (SdFat)"]
        FC -->|SPI| RF_TX["RFM69HCW 433 MHz Transceiver"]
    end

    RF_TX -.->|433 MHz FSK Binary Frames (0xA55A)| RF_RX["Dedicated CanSat Ground Receiver (ESP32)"]

    subgraph Ground_Segment["Ground Segment & Telemetry Server"]
        RF_RX -->|USB Serial @ 115200| FRAMER["cansat-framer.js / machx-serial.js"]
        FRAMER --> SERVER["Shared Ground Station Backend (Node.js / Express)"]
        SERVER --> DB[(SQLite flight.db)]
        SERVER --> FSM["Flight Phase Tracker"]
        SERVER -->|WebSocket| DASH["CanSat Dashboard (mach-x.html)"]
    end
```

---

## Flight Computer

* **Microcontroller:** STM32F103C8T6 ARM Cortex-M3 (72 MHz, 64 KB Flash, 20 KB SRAM).
* **Firmware Framework:** PlatformIO with Arduino-STM32 (`ststm32` platform).
* **Hardware Watchdog:** Integrated independent hardware watchdog (`IWatchdog`) configured to prevent system lockup.
* **Sensor Array & Bus Interfacing:**
  * **Barometric Pressure & Altitude:** Adafruit BMP388 barometric altimeter and temperature sensor on I2C (`0x76`, `SCL: PB6`, `SDA: PB7`).
  * **Inertial Measurement (6-DOF):** Adafruit MPU-6500 3-axis accelerometer and 3-axis gyroscope on SPI2 (`NCS: PB12`, `SCK: PB13`, `MISO: PB14`, `MOSI: PB15`, `INT: PA8`).
  * **Multi-Point Thermal Array:** 4× LM75 digital temperature sensors on I2C (`0x48`, `0x49`, `0x4A`, `0x4C`, `SCL: PB6`, `SDA: PB7`) distributed across payload structural zones.
  * **GPS Module:** U-Blox NEO-6M GPS receiver on Hardware UART (`RX: PB11`, `TX: PB10`) using `TinyGPSPlus`.
* **Telemetry Radio:** HopeRF RFM69HCW 433.0 MHz transceiver interfaced via SPI (`NSS: PA15`, `DIO0: PB5`, `SCK: PB13`, `MISO: PB14`, `MOSI: PB15`) driven by the `RadioHead` (`RH_RF69`) library.
* **Onboard Storage:** SPI MicroSD card module driven by the `SdFat` library (`CS: PA4`, `SCK: PA5`, `MISO: PA6`, `MOSI: PA7`).
* **Source Code:** Located in [`CanSAT/Firmware/Flight_Computer/`](Firmware/Flight_Computer/).

---

## Ground Station

The SUGAR CanSat utilizes a dedicated ground receiver located in [`CanSAT/Firmware/Ground_Station/`](Firmware/Ground_Station/):

* **Embedded Receiver Hardware:** ESP32 Dev Module with an onboard HopeRF RFM69HCW 433 MHz transceiver.
* **Firmware Operation:** Continuously listens for 433 MHz binary packets, stamps receiver RSSI, and streams intact binary frames over USB Serial at 115200 baud.
* **Ingest Driver (`cansat-framer.js` / `machx-serial.js`):** A stream `Transform` parser that scans the serial stream for the `0xA55A` synchronization word, frames 60-byte packet windows, validates CRC16-CCITT checksums, and forwards verified frames to the central backend.

---

## Telemetry Protocol

The SUGAR CanSat communicates using fixed-size binary telemetry frames to maximize link throughput and reliability over 433 MHz FSK.

### 60-Byte Binary Packet Structure (`Packet v3`)

| Offset (Bytes) | Type | Field Name | Description |
|:---|:---|:---|:---|
| `0..1` | `uint16_t` | `sync` | Packet synchronization word (`0xA55A`) |
| `2` | `uint8_t` | `version` | Protocol version (`0x03`) |
| `3` | `uint8_t` | `source_id` | Source identifier (`0x01` = CanSat / SUGAR) |
| `4` | `uint8_t` | `payload_len` | Payload length (`53` bytes) |
| `5..6` | `uint16_t` | `pkt_id` | Monotonically increasing packet counter |
| `7..10` | `uint32_t` | `timestamp_ms` | Mission elapsed time since boot (ms) |
| `11` | `uint8_t` | `mode` | Mission mode (`0=PRE_DEPLOY`, `1=DEPLOYED_SCIENCE`, `2=GPS_RECOVERY`) |
| `12..15` | `float` | `altitude_m` | Relative barometric altitude (m) |
| `16..19` | `float` | `temp_c` | Temperature measured at BMP388 sensor (°C) |
| `20..23` | `float` | `pressure_hpa` | Barometric pressure (hPa) |
| `24..27` | `float` | `temp_c_1` | External LM75 sensor 1 temperature (°C) |
| `28..31` | `float` | `temp_c_2` | External LM75 sensor 2 temperature (°C) |
| `32..35` | `float` | `temp_c_3` | External LM75 sensor 3 temperature (°C) |
| `36..39` | `float` | `temp_c_4` | External LM75 sensor 4 temperature (°C) |
| `40..43` | `float` | `accel_z` | Vertical axis acceleration (m/s²) |
| `44..47` | `float` | `gyro_x` | Angular velocity (deg/s) |
| `48..51` | `float` | `lat` | GPS Latitude (decimal degrees; active in GPS recovery mode) |
| `52..55` | `float` | `lon` | GPS Longitude (decimal degrees; active in GPS recovery mode) |
| `56` | `int8_t` | `rssi_dbm` | Received Signal Strength Indicator (dBm, stamped by ground receiver) |
| `57` | `uint8_t` | `flags` | Status bitmask (`bit0=launched`, `bit1=apogee`, `bit2=gps_fix`, `bit3=bmp_ok`, `bit4=mpu_ok`, `bit5=sd_ok`, `bit6=stale_sensor`, `bit7=gps_recovery`) |
| `58..59` | `uint16_t` | `crc16` | CRC16-CCITT checksum calculated over bytes `0..57` |

*Note: The ground parser also supports backward compatibility with legacy 43-byte v2 packets.*

---

## Software & Dashboard

* **Telemetry Framing & Parsing:** [`cansat-framer.js`](Firmware/Ground_Station/cansat-framer.js) and [`cansat-hardware.js`](Firmware/Ground_Station/cansat-hardware.js) handle synchronization, byte-window verification, and sensor unit conversions.
* **Mission Dashboard:** [`CanSAT/Dashboard/mach-x.html`](Dashboard/mach-x.html) is an interactive interface displaying live altitude plots, 3D attitude indicators, thermal readouts across 4 zones, GPS coordinates, and raw packet streams.

---

## Testing & Flight Data

* **Framing Stream Unit Tests:** [`CanSAT/Testing/cansat-framer.test.js`](Testing/cansat-framer.test.js) validates packet framing under split byte streams, noisy prefixes, and packet concatenation.
* **Synthetic Telemetry Generator:** [`CanSAT/Testing/generate_fake_machx.py`](Testing/generate_fake_machx.py) generates valid binary packets simulating a parabolic rocket trajectory for bench testing.
* **Flight Dataset:** [`CanSAT/Flight_Data/machx_fake_flight.csv`](Flight_Data/machx_fake_flight.csv) contains simulated flight trajectory logs used for ground station verification.

---

## Documentation

* [`CANSAT_CIRCUIT.md`](Documentation/CANSAT_CIRCUIT.md) — Hardware pin assignments, wiring tables, and power bus routing for the STM32 flight computer.
* [`CANSAT_GROUNDSTATION.md`](Documentation/CANSAT_GROUNDSTATION.md) — Setup and deployment manual for the 433 MHz ESP32 ground station receiver.
* [`2026-04-22-mach-x-cansat-firmware.md`](Documentation/2026-04-22-mach-x-cansat-firmware.md) — Implementation architecture notes for the binary flight firmware.

---

## Repository Structure

```text
CanSAT/
├── Dashboard/
│   └── mach-x.html                      # Real-time web telemetry dashboard
├── Documentation/
│   ├── 2026-04-22-mach-x-cansat-firmware.md # Firmware implementation plan
│   ├── CANSAT_CIRCUIT.md                # Pin assignments & circuit wiring
│   └── CANSAT_GROUNDSTATION.md          # Ground station operational guide
├── Firmware/
│   ├── Flight_Computer/                 # STM32 PlatformIO project
│   │   ├── include/telemetry.h          # Binary packet C struct definitions
│   │   ├── platformio.ini               # PlatformIO build configuration
│   │   └── src/main.cpp                 # Main flight loop & sensor drivers
│   └── Ground_Station/                  # ESP32 433 MHz receiver PlatformIO project
│       ├── cansat-framer.js             # Binary stream framing & sync module
│       ├── cansat-hardware.js           # Packet masks, CRC, & health decoders
│       ├── machx-serial.js              # Ground serial port handler
│       ├── platformio.ini               # PlatformIO build configuration
│       └── src/main.cpp                 # ESP32 RFM69 continuous receiver
├── Flight_Data/
│   └── machx_fake_flight.csv            # Simulated flight trajectory dataset
└── Testing/
    ├── cansat-framer.test.js            # Node.js stream transform unit tests
    └── generate_fake_machx.py           # Synthetic binary packet generator
```
