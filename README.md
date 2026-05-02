# Nexsus-EV-Dashboard
---
## Table of Contents

1. [Project Overview](#project-overview)
2. [Live Dashboard](#live-dashboard)
3. [System Architecture](#system-architecture)
4. [Hardware Components](#hardware-components)
5. [How It Works](#how-it-works)
   - [Battery Monitoring](#battery-monitoring)
   - [CAN Bus Communication](#can-bus-communication)
   - [Thermal Management – Three Zone Model](#thermal-management--three-zone-model)
   - [MQTT and EMQX Integration](#mqtt-and-emqx-integration)
   - [NEXUS Web Dashboard](#nexus-web-dashboard)
6. [MQTT Topic Structure](#mqtt-topic-structure)
7. [Firmware Logic Overview](#firmware-logic-overview)
   - [ESP32 Firmware](#esp32-firmware)
   - [STM32 Firmware](#stm32-firmware)
8. [Fault and Emergency Handling](#fault-and-emergency-handling)
9. [Directory Structure](#directory-structure)
10. [Getting Started](#getting-started)
    - [Hardware Setup](#hardware-setup)
    - [ESP32 Firmware Flash](#esp32-firmware-flash)
    - [STM32 Firmware Flash](#stm32-firmware-flash)
    - [Opening the Dashboard](#opening-the-dashboard)
11. [Results](#results)
12. [Future Improvements](#future-improvements)
13. [Team](#team)
14. [References](#references)

---

## Project Overview

This project implements a real-time, embedded Battery Management System (BMS)
for a small electric vehicle prototype powered by a 3S Lithium Polymer (LiPo)
battery pack (11.1 V nominal, 35C discharge rating).

The core problem with most low-cost EV prototypes is that the battery protection
circuit and the motor controller are completely isolated from each other. If the
battery overheats or draws too much current, the motor keeps running at full
power and makes the situation worse. There is also no way for the operator to
watch what is happening to the battery in real time during a test run.

This system solves both problems:

- A CAN bus connects the ESP32 (which reads the battery sensors) to the STM32
  (which controls the motor), so battery state directly and immediately
  influences motor behaviour.

- An MQTT broker (EMQX) connects the ESP32 to a web dashboard called NEXUS,
  so the operator can watch voltage, current, temperature, speed, and RPM live
  from any browser and receive fault alerts the moment a threshold is crossed.

The result is a closed-loop BMS where battery health governs motor output in
hard real time (CAN, < 2 ms response) and the operator sees everything through
a live cloud interface (MQTT, < 500 ms latency).

---

## Live Dashboard

The NEXUS EV Dashboard is hosted publicly on GitHub Pages:

https://anishparnapalli.github.io/Nexsus-EV-Dashboard/

To connect it to live hardware, power on the ESP32 with Wi-Fi credentials and
EMQX broker address configured. The dashboard will begin receiving telemetry
automatically on the vehicle/# topic tree.

To simulate the dashboard without hardware, use any MQTT client (e.g., MQTTX)
to publish JSON payloads to the topics listed in the MQTT Topic Structure
section below.

---

## System Architecture

The system is divided into four functional layers:
SENSING LAYER
3S LiPo Battery (11.1 V / 35C)
├── Voltage Sensor   →  Resistor Divider (10 kΩ / 2.2 kΩ) → ESP32 ADC
├── Current Sensor   →  ACS712-30A Hall Effect             → ESP32 ADC
├── Temp Sensor      →  DS18B20 1-Wire                     → ESP32 GPIO
└── Encoder Motor    →  Quadrature Pulse Output            → ESP32 GPIO INT
Buck Converter: 9–15 V → 5 V / 3.3 V logic rails
PROCESSING LAYER
ESP32 (Dual-core, 240 MHz, Wi-Fi)
├── Reads all 4 sensor channels every 100 ms
├── Computes: pack voltage, current (A), temperature (°C), RPM
├── Builds CAN frame → transmits on CAN1 (MCP2515)
└── Publishes JSON telemetry to EMQX via MQTT over Wi-Fi every 500 ms
COMMUNICATION LAYER
CAN Bus at 500 kbps (MCP2515 transceivers, 120 Ω termination)
Payload per frame: [PWM byte | Freq high byte | Freq low byte | Dir byte]
CONTROL LAYER
STM32 (ARM Cortex-M4)
├── Receives CAN frame on CAN2 (MCP2515)
├── Reads DS18B20 temperature
├── Applies three-zone thermal derating logic
├── Outputs PWM + direction to TB6612FNG H-bridge
└── Drives encoder motor (12 V DC)
IoT CLOUD LAYER
EMQX Public Broker (broker.emqx.io)
├── Receives telemetry published by ESP32
├── Forwards to NEXUS dashboard over WebSocket (port 8084)
└── Forwards dashboard control commands back to ESP32

Data flows in two directions:

  Telemetry:  ESP32 → EMQX → NEXUS Dashboard
  Control:    NEXUS Dashboard → EMQX → ESP32 → CAN → STM32 → Motor

---

## Hardware Components

| Component        | Specification                     | Role                                                       |
|------------------|-----------------------------------|------------------------------------------------------------|
| 3S LiPo Battery  | 11.1 V nominal, 35C discharge     | Primary energy source for the entire system                |
| ESP32 Module     | Dual-core 240 MHz, 802.11n Wi-Fi  | Data acquisition, CAN1 TX, MQTT publish and subscribe      |
| STM32 MCU        | ARM Cortex-M4                     | CAN2 RX, PWM generation, thermal derating and cutoff logic |
| MCP2515 × 2      | SPI-to-CAN, up to 1 Mbps         | CAN physical layer bridge for ESP32 and STM32              |
| ACS712-30A       | 66 mV/A, ±30 A range              | Bidirectional current measurement on the main pack rail    |
| Voltage Divider  | 10 kΩ / 2.2 kΩ resistors          | Scales 12.6 V max pack voltage to 3.3 V ADC range          |
| DS18B20          | 1-Wire, ±0.5 °C, 12-bit           | Cell surface temperature; input to thermal zone logic      |
| TB6612FNG        | 1.2 A continuous, 3.2 A peak      | Dual H-bridge motor driver; STBY pin for emergency cutoff  |
| DC Encoder Motor | 12 V, quadrature encoder          | Drive motor with speed and position feedback to ESP32      |
| Buck Converter   | Input 9–15 V, Output 5 V / 3.3 V | Regulated logic supply for all MCUs and sensors            |

---

## How It Works

### Battery Monitoring

Three sensors feed the ESP32 continuously:

VOLTAGE
  A resistor divider (10 kΩ and 2.2 kΩ) scales the 0–12.6 V pack voltage
  down to the 0–3.3 V ADC range of the ESP32. The firmware reads the raw ADC
  value and multiplies by the inverse scaling factor to recover the true pack
  voltage. A 100 nF decoupling capacitor at the ADC pin suppresses switching
  noise from the motor driver.

  Overvoltage threshold:   > 12.6 V  →  fault
  Undervoltage threshold:  < 9.0 V   →  fault

CURRENT
  The ACS712-30A produces a voltage of (VCC/2) + 66 mV × I, where I is the
  signed current in amperes. The ESP32 reads the ADC value, subtracts the
  zero-current offset (measured at startup with the motor idle), and divides
  by the 66 mV/A sensitivity to get current in amperes. Both charge current
  (negative) and discharge current (positive) are measured without breaking
  the circuit.

  Overcurrent threshold:  > 8 A sustained for 200 ms  →  fault
  The 200 ms debounce prevents motor inrush spikes from triggering false alarms.

TEMPERATURE
  The DS18B20 is mounted directly on the cell surface and communicates over the
  Dallas 1-Wire protocol using a single ESP32 GPIO pin with a 4.7 kΩ pull-up
  resistor. It delivers 12-bit readings (0.0625 °C resolution) every 100 ms
  polling cycle.

SOC ESTIMATION
  State of charge is estimated using a combined approach:
  1. Open-circuit voltage lookup maps pack voltage to SOC:
       12.6 V → 100%,  11.1 V → 50%,  9.6 V → 20%,  9.0 V → 0%
  2. Coulomb counting integrates ACS712 current over time using the trapezoidal
     rule to track charge removed during operation.
  3. When the motor has been idle for more than 5 seconds, the coulomb count is
     reset against the measured open-circuit voltage to correct drift.

  When SOC drops below 30%, the ESP32 firmware caps the commanded PWM at
  180/255 regardless of the drive mode selected on the dashboard.

---

### CAN Bus Communication

The ESP32 (CAN1, transmitter) and STM32 (CAN2, receiver) are connected by a
two-wire differential CAN bus at 500 kbps. Each node uses an MCP2515 SPI-to-CAN
transceiver. A 120 Ω termination resistor sits at each physical end of the bus.

CAN was chosen over UART or SPI for three reasons:
  - Differential signalling gives excellent noise rejection near the switching
    motor driver.
  - Multi-master arbitration means the STM32 can also transmit emergency frames
    back to the ESP32 without bus contention.
  - Guaranteed delivery semantics ensure no control frame is silently lost.

Each CAN data frame carries 4 bytes:

  Byte 0:   PWM duty cycle (0–255)
  Byte 1:   Motor frequency high byte   ┐
  Byte 2:   Motor frequency low byte    ┘  (100 Hz to 30 000 Hz combined)
  Byte 3:   Direction flag (0 = forward, 1 = reverse)

---

### Thermal Management – Three Zone Model

The STM32 reads the DS18B20 temperature on every CAN receive interrupt and
applies one of three actions before forwarding the PWM command to the
TB6612FNG:

ZONE 1 – Normal  (T < 40 °C)
  No intervention. The STM32 passes the commanded PWM duty cycle through to the
  motor driver unchanged.

ZONE 2 – Warning  (40 °C ≤ T ≤ 55 °C)
  Linear derating:
    effective_pwm = commanded_pwm × (1 − 0.03 × (T − 40))
  This reduces PWM by approximately 3% per degree above 40 °C.
  At 50 °C:  effective PWM ≈ 70% of commanded
  At 55 °C:  effective PWM ≈ 55% of commanded
  The vehicle remains driveable but at reduced power, giving the operator time
  to reduce load or stop.

ZONE 3 – Emergency  (T > 55 °C)
  Immediate hard shutdown:
    TB6612FNG STBY pin driven LOW  →  all motor output cut instantly
    STM32 transmits a CAN emergency frame to the ESP32
    ESP32 publishes "OVERHEAT" to the vehicle/emergency MQTT topic
    NEXUS dashboard raises the OVERHEAT DETECTED alert banner
  The motor stays off until a manual reset command is received from the
  dashboard. This prevents automatic restart into a still-hot condition.

---

### MQTT and EMQX Integration

EMQX is an open-source, enterprise-grade MQTT broker supporting MQTT 3.1.1
and 5.0. The project uses the free public cluster at broker.emqx.io.

The ESP32 connects on port 1883 (TCP) using the Arduino PubSubClient library.
The NEXUS dashboard connects on port 8084 (WebSocket over TLS) using the Paho
MQTT JavaScript client.

Telemetry is published every 500 ms. Fault events are published immediately
when detected within the 100 ms sensor polling loop. Control commands from the
dashboard are published at QoS 1 (at-least-once delivery) to ensure broker
acknowledgement before the client considers a command sent.

The publish-subscribe model means:
  - Multiple dashboards can subscribe to the same telemetry simultaneously.
  - Any MQTT client (MQTTX, Node-RED, Python paho, etc.) can receive or
    inject data for testing without changing any firmware.

---

### NEXUS Web Dashboard

URL:  https://anishparnapalli.github.io/Nexsus-EV-Dashboard/

Technology stack:
  - HTML5, CSS3, Vanilla JavaScript
  - Paho MQTT JavaScript client (WebSocket connection to EMQX)
  - HTML Canvas for the speed history chart
  - GitHub Pages hosting (no backend, no server)

Dashboard panels:

  Power Cell Panel
    Animated percentage gauge showing estimated SOC.
    Live voltage reading in volts.
    State indicator: OPTIMAL / WARNING / CRITICAL.

  Live Telemetry Panel
    Current (A), Voltage (V), Motor Temperature (°C) as digital readouts.
    Updated every 500 ms from incoming MQTT messages.

  Speedometer and RPM Gauge
    Animated circular speedometer (0–200 km/h).
    Semi-circular RPM counter derived from encoder pulse count.

  Drive Mode Selector
    ECO:     PWM cap at 100/255  (~39% of max)
    NORMAL:  PWM cap at 150/255  (~59% of max)
    SPORT:   PWM cap at 200/255  (~78% of max)
    RACE:    PWM cap at 255/255  (full power)

  PWM and Frequency Controls
    Manual sliders and text inputs for direct PWM (0–255) and
    frequency (0–50 000 Hz) entry. Published to vehicle/control/pwm
    and vehicle/control/freq.

  Motor Direction Toggle
    FORWARD (▶) and BACKWARD (◀) buttons publish to vehicle/control/direction.

  Vehicle Signal Controls
    LEFT indicator, RIGHT indicator, HEAD light, HAZARD buttons.
    Published as Boolean payloads to vehicle/signals/* topics.

  Speed History Chart
    Scrolling line chart (30 data points) of speed over time.

  MQTT Telemetry Log
    Scrollable log of all MQTT messages, colour-coded by type:
      SUB   incoming telemetry (green)
      PUB   outgoing control commands (blue)
      SYS   broker system messages (grey)

  Emergency Alert Banners
    Three banner elements, hidden by default, shown in red when triggered:
      OVERHEAT DETECTED
      LOW BATTERY WARNING
      OVERCURRENT FAULT

---

## MQTT Topic Structure

TELEMETRY  (ESP32 publishes → Dashboard subscribes)

  Topic                   Payload example            Description
  ──────────────────────────────────────────────────────────────────
  vehicle/telemetry       {"V":11.4,"I":2.1,         Full JSON telemetry bundle
                           "T":38.2,"RPM":820,
                           "SOC":67,"speed":24.3}
  vehicle/emergency       "OVERHEAT"                 Fault code string
                          "OVERCURRENT"
                          "UNDERVOLTAGE"

CONTROL  (Dashboard publishes → ESP32 subscribes → CAN → STM32)

  Topic                       Payload example     Description
  ──────────────────────────────────────────────────────────────
  vehicle/control/pwm         "180"               PWM duty cycle (0–255)
  vehicle/control/freq        "5000"              Motor frequency in Hz
  vehicle/control/direction   "0"                 0 = forward, 1 = reverse
  vehicle/control/reset       "1"                 Clear emergency state

SIGNALS  (Dashboard publishes → ESP32 subscribes → GPIO)

  vehicle/signals/left        "true" / "false"
  vehicle/signals/right       "true" / "false"
  vehicle/signals/head        "true" / "false"
  vehicle/signals/hazard      "true" / "false"

---

## Firmware Logic Overview

### ESP32 Firmware

Language: Arduino C++ (ESP-IDF compatible)
Libraries: PubSubClient (MQTT), OneWire + DallasTemperature (DS18B20),
           MCP_CAN (MCP2515 SPI-to-CAN)

Main loop (runs every 100 ms):

  1. Read voltage ADC  →  scale to pack voltage
  2. Read current ADC  →  subtract offset, scale to amperes
  3. Read DS18B20      →  temperature in °C
  4. Read encoder ISR counter  →  compute RPM
  5. Check fault conditions:
       if V > 12.6 or V < 9.0   →  set voltage_fault flag
       if I > 8.0 for 200 ms    →  set current_fault flag
       if T > 55.0               →  set thermal_fault flag
  6. Build CAN frame from commanded_pwm, freq, direction, fault flags
  7. Transmit CAN frame on CAN1
  8. Every 500 ms: build JSON payload, publish to vehicle/telemetry
  9. If any fault flag set: publish fault string to vehicle/emergency
  10. Call mqttClient.loop()  →  check for incoming control commands
  11. On MQTT receive callback: update commanded_pwm, freq, direction

SOC update (runs every 500 ms):
  coulombs_removed += current × 0.5  (trapezoidal, 0.5 s interval)
  soc = lookup(V_oc) − (coulombs_removed / capacity_As × 100)
  if motor idle > 5 s: reset coulombs_removed using V_oc lookup

### STM32 Firmware

Language: STM32 HAL C (STM32CubeIDE)
Libraries: STM32 HAL CAN driver, STM32 HAL TIM (PWM), MCP2515 SPI driver

CAN receive interrupt callback (fires on every valid CAN frame):

  1. Extract from frame data bytes:
       commanded_pwm  = data[0]           (0–255)
       target_freq    = (data[1]<<8)|data[2]  (Hz)
       direction      = data[3]           (0 or 1)
  2. Read DS18B20 temperature
  3. Apply thermal zone logic:
       if T > 55:
         set TB6612FNG STBY pin LOW        ← hard motor cutoff
         transmit CAN emergency frame back to ESP32
       else if T >= 40:
         derate = 1.0 - 0.03 * (T - 40)
         effective_pwm = commanded_pwm * derate
         set TB6612FNG PWM and direction with effective_pwm
       else:
         set TB6612FNG PWM and direction with commanded_pwm
  4. On MQTT reset command received via CAN relay from ESP32:
       restore TB6612FNG STBY pin HIGH    ← re-enable motor driver

---

## Fault and Emergency Handling

Fault Condition       Trigger                 CAN Response          MQTT Response          Dashboard
─────────────────────────────────────────────────────────────────────────────────────────────────────
Thermal Warning       40 °C ≤ T ≤ 55 °C      PWM derating active   No alert               None
Thermal Emergency     T > 55 °C              STBY pin LOW          "OVERHEAT" published   OVERHEAT DETECTED banner
Overcurrent           I > 8 A for 200 ms     Emergency CAN frame   "OVERCURRENT" published OVERCURRENT FAULT banner
Undervoltage          V < 9.0 V              Emergency CAN frame   "UNDERVOLTAGE" published LOW BATTERY WARNING banner
Overvoltage           V > 12.6 V             Emergency CAN frame   "UNDERVOLTAGE" published LOW BATTERY WARNING banner
SOC < 30%             Coulomb counting        PWM capped at 180     No alert               Battery gauge amber

Recovery: All fault states except thermal warning require a manual reset
command (published to vehicle/control/reset = "1") from the NEXUS dashboard.
The STM32 will not re-enable the motor driver until the reset command is
relayed via CAN and the fault condition has cleared.

---

## Directory Structure
smart-ev-bms/
│
├── firmware/
│   ├── esp32/
│   │   ├── main/
│   │   │   ├── bms_main.ino          Main Arduino sketch
│   │   │   ├── sensor_readings.cpp   Voltage, current, temperature, encoder
│   │   │   ├── can_handler.cpp       MCP2515 CAN frame build and transmit
│   │   │   ├── mqtt_handler.cpp      PubSubClient publish and subscribe
│   │   │   └── soc_estimator.cpp     Coulomb counting and OCV lookup
│   │   └── include/
│   │       └── config.h              Wi-Fi SSID, MQTT broker, thresholds
│   │
│   └── stm32/
│       ├── Core/
│       │   ├── Src/
│       │   │   ├── main.c            HAL init and main loop
│       │   │   ├── can_receiver.c    CAN2 interrupt and frame parsing
│       │   │   ├── thermal_control.c Three-zone thermal logic
│       │   │   └── motor_driver.c    TB6612FNG PWM and direction control
│       │   └── Inc/
│       │       └── config.h          CAN ID, temperature thresholds, PWM limits
│       └── STM32_BMS.ioc             STM32CubeIDE project file
│
├── dashboard/
│   ├── index.html                    NEXUS dashboard entry point
│   ├── style.css                     Dashboard styling
│   ├── app.js                        MQTT connection and UI logic
│   └── assets/                       Icons and fonts
│
├── docs/
│   ├── block_diagram.png             System architecture diagram
│   ├── circuit_schematic.pdf         Full wiring schematic
│   └── report.docx                   Full project report
│
├── hardware/
│   └── bom.csv                       Bill of materials with part numbers
│
└── README.md

---

## Getting Started

### Hardware Setup

1. Wire the voltage divider (10 kΩ + 2.2 kΩ) across the 3S LiPo pack terminals.
   Connect the midpoint to ESP32 GPIO34 (ADC1_CH6).

2. Wire the ACS712-30A in series with the main discharge rail between the battery
   positive terminal and the TB6612FNG VM pin. Connect the ACS712 OUT pin to
   ESP32 GPIO35 (ADC1_CH7).

3. Connect the DS18B20 data line to ESP32 GPIO4 with a 4.7 kΩ pull-up to 3.3 V.
   Mount the DS18B20 body against the LiPo cell surface and secure with tape.

4. Wire MCP2515 module #1 to ESP32 SPI (MOSI=GPIO23, MISO=GPIO19, SCK=GPIO18,
   CS=GPIO5, INT=GPIO2). Set its CAN ID to 0x100.

5. Wire MCP2515 module #2 to STM32 SPI1. Set its CAN ID to 0x200.

6. Connect CAN_H and CAN_L between both MCP2515 modules. Place a 120 Ω
   resistor between CAN_H and CAN_L at each physical end of the bus wire.

7. Connect STM32 PWM output (TIM1_CH1 = PA8) to TB6612FNG PWMA pin.
   Connect STM32 GPIO (PB0) to TB6612FNG AIN1 and (PB1) to AIN2 for direction.
   Connect STM32 GPIO (PB5) to TB6612FNG STBY pin.

8. Connect the encoder motor M+ and M− to TB6612FNG AO1 and AO2.
   Connect encoder channel A to ESP32 GPIO32 and channel B to GPIO33.

9. Power the TB6612FNG VM pin from the battery positive via the ACS712.
   Power all MCUs and sensors from the buck converter 5 V output.

### ESP32 Firmware Flash

Requirements: Arduino IDE 2.x or PlatformIO, ESP32 board package installed.

1. Open firmware/esp32/main/bms_main.ino in Arduino IDE.

2. Edit firmware/esp32/include/config.h:
     WIFI_SSID     →  your Wi-Fi network name
     WIFI_PASSWORD →  your Wi-Fi password
     MQTT_BROKER   →  broker.emqx.io
     MQTT_PORT     →  1883

3. Select board: ESP32 Dev Module.
   Select port: your ESP32 COM/tty port.

4. Click Upload.

5. Open Serial Monitor at 115200 baud to verify sensor readings and MQTT
   connection status.

### STM32 Firmware Flash

Requirements: STM32CubeIDE, ST-Link debugger.

1. Open firmware/stm32/STM32_BMS.ioc in STM32CubeIDE and generate code.

2. Build and flash via ST-Link (Run → Debug or Run → Run).

3. Verify operation by checking the Serial Wire Viewer output or toggling an
   LED on the emergency cutoff callback.

### Opening the Dashboard

No installation required. Open this URL in any modern browser:

  https://anishparnapalli.github.io/Nexsus-EV-Dashboard/

The dashboard will show DISCONNECTED until the ESP32 is powered on and
connected to the same EMQX broker. Once the ESP32 publishes its first message,
the dashboard connects automatically and all panels begin updating.

To test without hardware, use MQTTX (https://mqttx.app) and publish to:
  broker.emqx.io:1883
  Topic: vehicle/telemetry
  Payload: {"V":11.4,"I":2.3,"T":36.5,"RPM":820,"SOC":68,"speed":22.1}

---

## Results

- Zero MQTT message drops recorded in 10-minute test sessions on the telemetry
  topics over a standard home Wi-Fi network.

- Average broker round-trip latency (dashboard command → STM32 acknowledgement)
  was 210 ms over Wi-Fi, adequate for the gradual speed changes of this platform.

- Thermal emergency shutdown (T > 55 °C) triggered motor cutoff within one CAN
  frame period (< 2 ms). NEXUS alert banner appeared within 350 ms.

- Pack voltage gauge on dashboard matched a calibrated multimeter to within
  ±0.3 V across the full operating voltage range.

- During a 60-second drive loop, pack voltage dropped from 11.6 V to 10.9 V and
  motor temperature rose from 27 °C to 43 °C, briefly activating thermal
  derating (15% PWM reduction) before recovering as load was eased.

- Overcurrent fault correctly debounced motor inrush spikes (which exceed 8 A
  for < 100 ms on startup) and only triggered on sustained overcurrent events.

---

## Future Improvements

- Per-cell voltage monitoring using a LTC6803 or BQ76920 multiplexer to detect
  cell imbalance and enable passive balancing.

- TLS-secured MQTT (port 8883) with credential-based authentication to prevent
  unauthorised vehicle control via the public broker.

- Extended Kalman Filter SOC estimation to reduce error from the current
  ±8% (voltage + coulomb counting) to below ±2%.

- GPS module integration for independent speed measurement and location-aware
  alerts on the dashboard.

- Active cooling (small fan triggered at Zone 2 onset) to delay Zone 3 shutdown
  during sustained high-load operation.

- OTA (over-the-air) firmware update capability for the ESP32 to allow
  threshold tuning without physical access to the vehicle.

---

## Team

| Name              | Registration No.    | Contribution                              |
|-------------------|---------------------|-------------------------------------------|
| Ramachandru J     | RA2311004010045     | Hardware assembly, motor control firmware |
| Shibi S           | RA2311004010064     | System design, documentation, reporting   |
| Parnapalli Anish  | RA2311004010117     | NEXUS dashboard, MQTT integration, IoT    |

Supervisor: Dr. T. Rajalakshmi, Associate Professor, Dept. of ECE,
SRM Institute of Science and Technology, Kattankulathur – 603203.

---

## References

[1] C. Yanpreechaset, N. Donjaroennon, S. Nuchkum, and U. Leeton,
    "Development of an EV Battery Management Display with CANopen
    Communication," World Electric Vehicle Journal, vol. 16, no. 7,
    p. 375, 2025. doi: 10.3390/wevj16070375

[2] A. Gozuoglu, "IoT-enhanced battery management system for real-time
    SoC and SoH monitoring using STM32-based programmable electronic load,"
    Internet of Things, vol. 30, p. 101509, 2025.
    doi: 10.1016/j.iot.2025.101509

[3] S. Ebadinezhad et al., "Evaluating the Energy Consumption of ESP32
    Microcontroller for Real-Time MQTT IoT-Based Monitoring System," in
    Proc. 2023 Int. Conf. on Innovation and Intelligence for Informatics,
    Computing, and Technologies (3ICT), pp. 255–261, 2023.
    doi: 10.1109/3ICT60104.2023.10391358

[4] G. Nishanth, M. M. Krishnan, B. Parandhaman et al., "Hardware
    implementation of EKF based SOC estimate for lithium-ion batteries in
    electric vehicle applications," Scientific Reports, vol. 15, p. 15551,
    2025. doi: 10.1038/s41598-025-99931-8

[5] S. Ragul et al., "NANOHEXCOOL: Advanced Thermal Control for Lithium-Ion
    Batteries," in Proc. 2024 8th Int. Conf. on IoT in Social, Mobile,
    Analytics and Cloud (I-SMAC), IEEE, 2024.

---

*SRM Institute of Science and Technology — Department of ECE*
*21IPE306T Battery Management System for E-Mobility — 2025–26*
