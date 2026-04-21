# Bare-Chip-ESP32-Custom-Development-Board-with-16MB-Flash-and-PCB-Antenna
Custom 4-layer ESP32 bare-chip dev board (no module). Features dual buck power architecture, ideal diode power ORing, USB-C with ESD protection, CP2102N auto-reset, 16MB QSPI flash, PCB MIFA antenna, dual crystal setup, and dedicated decoupling across all rails. Designed with signal integrity and debug access as core priorities.

# Bare Chip ESP32 Custom Development Board

> Hardware design documentation and learning notes for a custom ESP32 development board built around the bare ESP32 chip (no module).

**Author:** Devansh Sharma · 1st Year Integrated MSc, NISER Bhubaneshwar  
**Version:** V2.0 · 19 April 2026

---

## Overview

This board uses the ESP32 IC directly — no pre-certified module like the WROOM. That means power integrity, RF layout, boot config, and clock stability all had to be handled from scratch. It's a system-level design exercise.

**Supports:**
- USB Type-C and external VIN (e.g. 12V) power input
- One-click flashing via automatic reset + boot control
- PCB trace antenna (MIFA) for 2.4GHz Wi-Fi and Bluetooth
- External SPI flash (W25Q128, 16MB) for firmware storage and XIP execution

---

## Table of Contents

- [ESP32 Core](#esp32-core)
- [Flash Interface](#flash-interface)
- [USB-UART Interface](#usb-uart-interface)
- [Auto Reset and Boot Control](#auto-reset-and-boot-control)
- [Power Block](#power-block)
- [PCB Antenna](#pcb-antenna)
- [Design Decisions](#design-decisions)
- [Components](#components)
- [Key Concepts](#key-concepts)
- [Version History](#version-history)
- [References](#references)

---

## ESP32 Core

Unlike simpler MCUs, the ESP32 is a system-level IC — its stable operation depends on power integrity, boot config, clock stability, and PCB layout all working together.

### Power Distribution and Decoupling

The 3.3V supply is distributed across multiple internal domains:

| Pin | Domain |
|-----|--------|
| VDD3P3 | Digital core logic |
| VDD3P3_RTC | RTC subsystem |
| VDD_SDIO | Flash interface |

Each pin is decoupled individually with **100nF** (HF noise) + **1–10µF** (transient support). Caps placed as close as possible to VDD pins with short vias to ground plane. Without this, Wi-Fi current spikes cause voltage dips that trigger the brownout detector.

### Enable (EN) Pin

- Must be HIGH for normal operation
- Pulled up with ~10kΩ resistor
- 100nF cap between EN and GND for noise filtering and stable power-up delay
- Controlled via RTS for auto-reset

### Boot Strapping

| GPIO0 State | Boot Mode |
|-------------|-----------|
| LOW | Flash / programming mode |
| HIGH | Normal execution |

GPIO0 is pulled HIGH by default and driven LOW via DTR during upload. Floating strapping pins = boot failures and unreliable programming.

### Clock System

**40MHz Crystal (Main Clock)** — mandatory for CPU, peripherals, and RF.
- Placed very close to ESP32 crystal pins
- Symmetric routing on both pins (imbalance → phase error)
- Load caps: **8.2pF** (calculated from the specific crystal's spec)
- No routing under the crystal, solid GND plane below

**32.768kHz Crystal (RTC)** — optional, improves sleep accuracy and timekeeping. 22pF load caps.

### VDD_SDIO

Internally regulated by the ESP32 to power the external flash. Routed from ESP32 directly to flash VCC with local 100nF decoupling at both ends. Since the ESP32 runs code directly from flash (XIP), this is a **boot-critical power domain** — instability here causes flash comms failures, boot failures, and reset loops.

### Grounding and Thermal Pad

The ESP32 QFN package has an exposed thermal pad tied to GND via multiple vias to the internal ground plane. Serves as thermal dissipation, central ground reference, and noise return path. A solid GND plane is maintained under the entire chip.

---

## Flash Interface

The bare ESP32 has no internal flash — all firmware lives on an external **W25Q128JVSIQTR** (128Mbit / 16MB, QSPI).

### Why It's Boot-Critical

The ESP32 uses **XIP (Execute-In-Place)**: code runs directly from flash without being copied to RAM. Any flash instability = processor crash.

### Interface Signals

| Signal | Function |
|--------|----------|
| CLK | Clock |
| CS# | Chip Select |
| IO0–IO3 | Data (QSPI) |

### Routing Rules

- Flash placed as close as possible to ESP32 (same PCB side)
- Short, direct routing for all SPI lines
- No vias on CLK line
- Matched trace lengths on CLK and data lines (minimises skew → prevents comms errors)
- 100nF local decoupling on VCC

Test points on CLK, CS, and IO lines for oscilloscope/logic analyser bring-up.

---

## USB-UART Interface

**IC:** CP2102N-A02-GQFN28 (Silicon Labs, 28-pin)

Bridges USB ↔ UART for serial comms, firmware upload, and debug logging. Also drives the auto-reset circuit.

### Signal Chain

USB-C → ESD (USBLC6-2SC6) → 22Ω Series Resistors → CP2102N → ESP32

ESD diode **must** come before the IC. Series resistors match impedance to ~90Ω differential.

### Connections

| CP2102N | ESP32 |
|---------|-------|
| TX | RX |
| RX | TX |

### Power and Key Pins

- Powered from 3.3V rail. Decoupling: 100nF + 4.7µF at VCC
- **VBUS** → 22.1kΩ / 47.5kΩ divider for USB connection detection
- **RSTb** → pulled HIGH via 2kΩ
- **SUSPENDb** → pulled via 10kΩ

Test points on DTR and RTS for debugging upload and reset behaviour.

---

## Auto Reset and Boot Control

Puts the ESP32 into programming mode automatically on upload — no manual BOOT/RESET button required.

### How It Works

Two NPN transistors (**SS8050-G**) translate DTR and RTS signals from the CP2102N into controlled pull-downs on EN and GPIO0.

| Signal | Controls |
|--------|----------|
| DTR | GPIO0 (boot mode) |
| RTS | EN (reset) |

**Sequence during upload:**
1. DTR pulls GPIO0 LOW
2. RTS toggles EN (reset pulse)
3. ESP32 enters bootloader
4. Both return HIGH → ESP32 runs new firmware

10kΩ pull-ups on EN and GPIO0 ensure stable HIGH default. 10kΩ base resistors on DTR/RTS lines limit transistor base current.

---

## Power Block

Dual-input, multi-stage architecture producing a clean 3.3V rail from USB-C or external VIN.

### Power Flow

USB-C (5V VBUS) ──┐
├──► Ideal Diode OR (LM66100 ×2) ──► 5V_rail ──► Buck (TPS62162) ──► 3.3V_rail
Ext. VIN ──────────┘
(e.g. 12V → TPS62163 Buck ──► 5Vin)

### USB-C Path

- Polyfuse for overcurrent protection
- 600Ω @ 100MHz ferrite bead to suppress cable noise
- 100nF + 4.7µF decoupling
- **5.1kΩ on CC1 and CC2** — mandatory for USB-C device detection

### External VIN Path

- SS14 Schottky for reverse polarity protection
- SMBJ15A bidirectional TVS for ESD protection
- TPS62163 buck (VIN → 5V) with 2.2µH inductor
- 47µF bulk decoupling at input and output (handles Wi-Fi current spikes)

### Power Rail Selection

Two **LM66100DCKR** ideal diodes perform power ORing — automatically selects the higher 5V source with near-zero forward voltage drop and no back-feeding.

### 3.3V Rail

**TPS62162** buck (5V → 3.3V):
- 2.2µH inductor, 100kΩ feedback resistors
- 47µF bulk + 100nF + 22µF output filtering
- Buck chosen over LDO for efficiency and better handling of ESP32 Wi-Fi transient spikes

### PCB Power Distribution

- 3.3V as a dedicated power plane (layer 3)
- Continuous GND plane (layer 2)
- Test points on 5V_rail, 3.3V_rail, GND, and Vin

---

## PCB Antenna

**Type:** Meandered Inverted-F Antenna (MIFA) — 2.4GHz Wi-Fi and Bluetooth, on-PCB.

### Structure and Matching

- Meandered copper trace resonant at ~2.4GHz
- Fed from ESP32 RF output through a **pi-network** matching to 50Ω
- Silk screen removed before fabrication to expose copper

### Critical Layout Rules

| Rule | Why |
|------|-----|
| No copper on any layer beneath antenna | Prevents detuning |
| GND/power planes stop at antenna feed boundary (0Ω jumper) | Maintains radiation efficiency |
| Via fence at antenna boundary | Stable RF reference, isolates digital noise |
| Antenna at PCB edge | Reduces signal absorption |

---

## Design Decisions

| Decision | Reason |
|----------|--------|
| **Buck over LDO for 3.3V** | Higher efficiency; better handling of Wi-Fi current spikes; reduces brownout risk |
| **CP2102N over CH340G** | Follows official ESP32 reference design; reliable DTR/RTS auto-reset support |
| **PCB trace antenna over external module** | RF layout learning exercise; lower cost; acceptable 2.4GHz performance |
| **LM66100 over Schottky** | Near-zero forward drop; active back-feed prevention |
| **External SPI flash** | Not a choice — bare ESP32 has no internal flash |

---

## Components

| Ref | Part | Description |
|-----|------|-------------|
| U7 | ESP32 | Dual-core LX6 MCU, Wi-Fi + Bluetooth |
| U10 | W25Q128JVSIQTR | 128Mbit QSPI NOR flash |
| U9 | CP2102N-A02-GQFN28 | USB to UART bridge |
| U2 | TPS62163DSGR | Buck regulator, VIN → 5V |
| U1 | TPS62162DSGT | Buck regulator, 5V → 3.3V |
| U3, U4 | LM66100DCKR | Ideal diode controller ×2 |
| D1 | USBLC6-2SC6 | USB ESD protection |
| D2 | SMBJ15A | TVS diode, VIN ESD protection |
| D3 | SS14 | Schottky, reverse polarity protection |
| Q1, Q2 | SS8050-G | NPN transistors, auto-reset circuit |
| X3 | 40MHz Crystal | Main system clock |
| X1 | 32.768kHz Crystal | RTC clock |
| F1, F2 | Polyfuse | Overcurrent protection |
| L2 | Ferrite Bead 600Ω@100MHz | USB VBUS noise filtering |
| R1, R2 | 5.1kΩ | USB Type-C CC pin resistors |
| — | 270pF | SENSOR_VP / SENSOR_VN filtering |

---

## Key Concepts

### Brownout Reset
The ESP32 resets itself if supply dips below ~2.5–2.7V — even momentarily. Wi-Fi TX bursts cause current spikes that can cross this threshold if decoupling is insufficient.

### XIP (Execute-In-Place)
The ESP32 executes instructions directly from external flash in real time — no copy to RAM first. Flash interface is therefore boot-critical, not just storage.

### USB to UART Conversion
USB uses differential signalling (D+/D-) with packet-based protocol. UART uses simple voltage-level TX/RX at a fixed baud rate. The CP2102N bridges them — USB engine handles protocol complexity, UART engine talks to ESP32. Neither side knows about the other's protocol.

---

## Version History

| Version | Changes |
|---------|---------|
| V1.0 | Initial schematic. Bug: DTR and IO0 nets connected to the same 10kΩ pull-up in auto-reset circuit. |
| V2.0 | Fixed V1.0 bug. Completed PCB layout, improved trace widths (5mil near ESP32, 6mil+ elsewhere). Silk screen removed from antenna area. |

---

## References

1. [ESP32 DevKitC V4 Official Schematic](https://dl.espressif.com/dl/schematics/esp32_devkitc_v4-sch.pdf)
2. [ESP32-WROOM-32 Datasheet](https://documentation.espressif.com/esp32-wroom-32_datasheet_en.pdf)
3. [TPS62160 Datasheet](https://www.ti.com/lit/ds/symlink/tps62160.pdf)
4. [LM66100 Datasheet](https://www.ti.com/lit/ds/symlink/lm66100.pdf)
5. [CP2102N Datasheet](https://www.silabs.com/documents/public/data-sheets/cp2102n-datasheet.pdf)
6. [USBLC6-2SC6 Datasheet](https://www.st.com/resource/en/datasheet/usblc6-2.pdf)
7. [W25Q128JVSIQ Datasheet](https://www.mouser.com/datasheet/2/949/w25q128jv_revf_03272018_plus1489608.pdf)
8. Component datasheets sourced from LCSC via EasyEDA
