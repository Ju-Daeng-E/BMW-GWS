# BMW-GWS: BMW Gear Lever to PiRacer Control System

<div align="center">

**Raspberry Pi-based vehicle control system that interfaces a real BMW F-Series gear lever with a PiRacer through CAN bus communication**

[![Python](https://img.shields.io/badge/Python-3.7+-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![CAN Bus](https://img.shields.io/badge/Protocol-CAN%20Bus-00979D)](https://en.wikipedia.org/wiki/CAN_bus)
[![Raspberry Pi](https://img.shields.io/badge/Platform-Raspberry%20Pi-A22846?logo=raspberrypi&logoColor=white)](https://www.raspberrypi.org/)
[![PyQt5](https://img.shields.io/badge/GUI-PyQt5-41CD52?logo=qt&logoColor=white)](https://www.riverbankcomputing.com/software/pyqt/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

</div>

---

## Overview

BMW-GWS (Gateway System) reads CAN messages from a **BMW F-Series gear lever** (P/R/N/D/M1-M8), processes gear transitions with toggle logic, and translates them into **PiRacer vehicle commands**. A real-time PyQt5 dashboard displays speed, gear status, and system diagnostics.

### Key Features

- **BMW CAN Integration** - Decode gear lever CAN messages (0x197) and control LED indicators (0x3FD)
- **Toggle-based Gear Logic** - Accurate BMW-style gear transition with center-return detection
- **GPIO Speed Sensor** - Real-time speed measurement via rotary encoder on GPIO16
- **Gamepad Control** - ShanWan gamepad for throttle/steering with speed gear adjustment
- **PyQt5 Dashboard** - Full-screen real-time monitoring UI optimized for 1280x400 displays
- **Headless Mode** - Automatic fallback when no display is connected

## Architecture

```
+-------------------+       CAN Bus (500kbps)       +-------------------+
|   BMW F-Series    | -----------------------------> |   Raspberry Pi    |
|   Gear Lever      |  ID: 0x197 (Lever Position)   |                   |
|                   | <----------------------------- |   CAN Controller  |
|   LED Indicator   |  ID: 0x3FD (LED Command)      |                   |
+-------------------+                                +--------+----------+
                                                              |
                          +-----------------------------------+-----------------------------------+
                          |                                   |                                   |
                +---------v---------+             +-----------v-----------+           +-----------v-----------+
                | BMW Lever         |             | Speed Sensor          |           | Gamepad Controller    |
                | Controller        |             | (GPIO16 Polling)      |           | (ShanWan Gamepad)     |
                |                   |             |                       |           |                       |
                | Gear Toggle Logic |             | RPM -> km/h           |           | Throttle / Steering   |
                | P/R/N/D/M1-M8    |             | Pulse Debouncing      |           | Speed Gear (L2/R2)    |
                +---------+---------+             +-----------+-----------+           +-----------+-----------+
                          |                                   |                                   |
                          +-----------------------------------+-----------------------------------+
                                                              |
                                                    +---------v---------+
                                                    |   PyQt5 Dashboard |
                                                    |   (1280 x 400)   |
                                                    |                   |
                                                    | Speedometer | Gear|
                                                    | Logs | Status     |
                                                    +-------------------+
                                                              |
                                                    +---------v---------+
                                                    |     PiRacer       |
                                                    |  Motor + Steering |
                                                    +-------------------+
```

## Project Structure

```
BMW-GWS/
├── main.py                    # Application entry point (GUI / Headless)
├── constants.py               # System configuration and enums
├── data_models.py             # BMWState, PiRacerState dataclasses
├── logger.py                  # Custom logger with multiple handlers
├── crc_calculator.py          # BMW CRC-8 calculation (0x197, 0x3FD)
├── bmw_lever_controller.py    # Gear lever decode and toggle logic
├── can_controller.py          # CAN bus send/receive management
├── speed_sensor.py            # GPIO polling-based speed measurement
├── gamepad_controller.py      # ShanWan gamepad input processing
├── gui_widgets.py             # SpeedometerWidget, GearDisplayWidget
├── main_gui.py                # PyQt5 main window and signal handling
├── tests/
│   └── test_modules.py        # Module import and functionality tests
├── requirements.txt
├── LICENSE
└── .gitignore
```

## Hardware Requirements

| Component | Description |
|-----------|-------------|
| Raspberry Pi 4B+ | Main controller with GPIO and CAN interface |
| BMW F-Series Gear Lever | CAN-connected shifter (P/R/N/D/S/M) |
| CAN Transceiver | MCP2515 or similar SPI-to-CAN module |
| PiRacer | Waveshare PiRacer AI Kit |
| ShanWan Gamepad | USB gamepad for throttle/steering |
| Rotary Encoder | Speed sensor connected to GPIO16 (20 slots) |
| Display (optional) | 1280x400 or larger for dashboard UI |

## Setup

### 1. Install Dependencies

```bash
pip install -r requirements.txt
```

### 2. Configure CAN Interface

```bash
sudo ip link set can0 down
sudo ip link set can0 up type can bitrate 500000
```

Verify with:
```bash
candump can0
```

### 3. Run

```bash
python3 main.py
```

The system automatically detects whether a display is available and switches between GUI and headless mode.

## CAN Protocol

### Gear Lever Message (0x197)

| Byte | Field | Description |
|------|-------|-------------|
| 0 | CRC | CRC-8 checksum (poly: 0x1D, xor: 0x53) |
| 1 | Counter | Rolling counter (0x01-0x0E) |
| 2 | Lever Position | 0x0E=Center, 0x1E=Up(R), 0x3E=Down(D), 0x7E=Side(S) |
| 3 | Buttons | Bit 0: Park, Bit 1: Unlock |

### LED Command Message (0x3FD)

| Byte | Field | Description |
|------|-------|-------------|
| 0 | CRC | CRC-8 checksum (poly: 0x1D, xor: 0x70) |
| 1 | Counter | Rolling counter |
| 2 | LED Code | 0x20=P, 0x40=R, 0x60=N, 0x80=D, 0x81=S/M |
| 3-4 | Reserved | 0x00 |

## Gear State Machine

```
          Unlock Button
    [P] ──────────────> [N]
     ^                 / | \
     |        Up ─────   |   ───── Down
     |              [R]  |  [D]
     |                   |   |
  Park Button        Side Toggle
     |                   |   |
     +───── any ────── [M1] - [M8]
                      Up+ / Down-
```

## Configuration

Key parameters in `constants.py`:

```python
# CAN
BMW_CAN_CHANNEL = 'can0'
CAN_BITRATE = 500000

# Speed Sensor
SPEED_SENSOR_PIN = 16       # GPIO BCM pin
PULSES_PER_TURN = 40        # 20 slots x 2 edges
WHEEL_DIAMETER_MM = 64      # PiRacer wheel diameter

# UI
WINDOW_WIDTH = 1280
WINDOW_HEIGHT = 400
```

## License

This project is licensed under the [MIT License](LICENSE).
