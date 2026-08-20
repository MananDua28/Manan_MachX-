# Manan_MachX

This repository contains the complete engineering development of two independently developed aerospace payload flight systems:

1. **SUGAR CanSAT** — A competition high-altitude rocket CanSat payload system featuring an STM32-based flight computer, integrated atmospheric and inertial sensor arrays, 433 MHz RF telemetry, onboard SD data logging, and a dedicated ground receiver.
2. **Rideshare Payload** — An autonomous rocket rideshare avionics and imaging system featuring an ESP32-S3 / SX1262 LoRa flight computer, 868 MHz long-range telemetry, local OLED state latching, dedicated ESP32-CAM video/still logging, and a dedicated LoRa ground receiver.

Both flight payloads interface with a unified, dual-receiver ground telemetry server that captures, logs, parses, and visualizes real-time flight metrics during mission operations.

---

## Projects

### [SUGAR CanSAT](CanSAT/)
The SUGAR CanSAT is an aerospace payload designed for high-altitude sounding rocket deployment. It performs autonomous in-flight atmospheric profiling, 6-DOF inertial tracking, multi-point thermal monitoring, and GPS positioning while streaming high-rate binary telemetry packets to a dedicated ground station receiver.
* **Documentation & Details:** [`CanSAT/README.md`](CanSAT/README.md)

### [Rideshare Payload](Rideshare_Payload/)
The Rideshare Payload is an autonomous secondary payload engineered for sounding rocket airframe integration. It features high-link-budget 868 MHz LoRa telemetry, redundant barometric/temperature sensing, continuous SD flight logging, autonomous OLED apogee latching, and an independent camera module for in-flight optical recording.
* **Documentation & Details:** [`Rideshare_Payload/README.md`](Rideshare_Payload/README.md)

### [Shared Ground Station](Shared_Ground_Station/)
The `Shared_Ground_Station/` directory contains the multi-mission ground server infrastructure used to operate both payload systems simultaneously on launch day. It manages concurrent serial communication streams from both the 433 MHz CanSat receiver and the 868 MHz Rideshare LoRa receiver, logs all packets to a persistent SQLite database, executes real-time flight state machine tracking, and provides browser-based telemetry dashboards.
* **Location:** [`Shared_Ground_Station/`](Shared_Ground_Station/)

---

## Repository Structure

```text
Manan_MachX/
├── CanSAT/
│   ├── Dashboard/               # Real-time web telemetry dashboard
│   ├── Documentation/           # Circuit schematics and firmware guides
│   ├── Firmware/
│   │   ├── Flight_Computer/     # STM32F103C8T6 flight computer firmware
│   │   └── Ground_Station/      # Dedicated ESP32 433 MHz RF receiver
│   ├── Flight_Data/             # Flight profiles and simulation datasets
│   └── Testing/                 # Packet framing tests and bench generator
├── Rideshare_Payload/
│   ├── Archive/                 # Historical test firmware (traceability)
│   ├── Dashboard/               # Real-time web telemetry dashboard
│   ├── Documentation/           # Wiring guides, testing procedures, setup
│   ├── Firmware/
│   │   ├── Camera_Module/       # ESP32-CAM autonomous optical logger
│   │   ├── Flight_Computer/     # Heltec V3 (ESP32-S3 + SX1262) flight software
│   │   └── Ground_Station/      # Dedicated Heltec V3 868 MHz LoRa receiver
│   ├── Flight_Data/             # Hardware bench tests and flight datasets
│   └── Testing/                 # Automated serial parser integration tests
├── Shared_Ground_Station/
│   ├── backend/                 # Node.js dual-port telemetry daemon & SQLite DB
│   ├── dashboard/               # Multi-mission ground station portal
│   └── docs/                    # Deployment guides and testing runbooks
├── images/                      # Project imagery and assets
├── .github/workflows/           # Automated CI pipelines for firmware & backend
└── README.md                    # Root repository overview
```

---

## Engineering Scope

The engineering disciplines and domains represented across this repository include:

* **Embedded Systems & Firmware Development:** Bare-metal and RTOS/framework firmware for STM32 ARM Cortex-M3 (PlatformIO / Arduino-STM32) and Espressif ESP32-S3 / ESP32 architectures with hardware watchdog integration.
* **Flight Computer Architecture:** Sensor acquisition pipelines, SPI SD card logging, fault-tolerant state loops, and brownout mitigation.
* **RF Communication & Telemetry Link Design:** Multi-band RF telemetry links utilizing 433 MHz FSK (RFM69HCW) binary-framed packets with CRC16-CCITT and 868 MHz LoRa (SX1262) ASCII-framed packets with checksum validation.
* **Sensor Integration & Bus Interfacing:** Hardware integration of barometric pressure sensors (BMP388, BMP280), 6-DOF IMUs (MPU-6500), multi-node I2C temperature sensors (LM75 arrays), and UART NMEA GPS modules (NEO-6M).
* **Autonomous Optical Subsystems:** Dedicated ESP32-CAM microcontroller firmware for fail-safe JPEG frame acquisition and high-speed SD_MMC logging during dynamic flight conditions.
* **Ground Segment & Mission Control Software:** Asynchronous dual-serial port ground daemon (Node.js / Express / Socket.io) with SQLite persistent storage, automated packet deduplication, and flight phase tracking state machines.
* **Real-Time Data Visualization:** Interactive telemetry dashboards featuring dynamic trajectory charting, 3D attitude visualization, and hardware health diagnostics.
* **Automated Verification & CI/CD:** Automated GitHub Actions workflows for multi-target PlatformIO firmware compilation, Node.js backend validation, and packet framing unit tests.
