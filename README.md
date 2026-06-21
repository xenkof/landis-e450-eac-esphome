# Landis+Gyr E450 Smart Meter Reader for Home Assistant
### ESPHome DLMS/M-Bus integration for Cyprus (EAC) smart meters

![ESPHome](https://img.shields.io/badge/ESPHome-2026.6.0+-blue)
![ESP32-C6](https://img.shields.io/badge/ESP32-C6%20SuperMini-green)
![License](https://img.shields.io/badge/license-MIT-brightgreen)

> **Community contribution** — This guide documents a working integration of the Landis+Gyr E450 electricity meter (as deployed by EAC in Cyprus) with Home Assistant via ESPHome. It is the result of extensive trial and error and is shared to save others the same effort.

---

## Table of Contents
- [Overview](#overview)
- [Hardware Required](#hardware-required)
- [Wiring](#wiring)
- [Software Setup](#software-setup)
- [ESPHome Configuration](#esphome-configuration)
- [Home Assistant Integration](#home-assistant-integration)
- [Power Saving (Optional)](#power-saving-optional)
- [Troubleshooting](#troubleshooting)
- [Known Issues](#known-issues)
- [Credits](#credits)

---

## Overview

The Landis+Gyr E450 smart meter deployed by EAC (Electricity Authority of Cyprus) has a wired M-Bus interface that broadcasts DLMS/COSEM data. This project reads that data using an ESP32-C6 and the ESPHome `dlms_meter` component, publishing live readings to Home Assistant.

**What you get in Home Assistant:**
- Live grid power import/export (W)
- Total energy import/export (kWh) — feeds HA Energy Dashboard
- Reactive power import/export (VAr)
- Reactive energy totals (kVArh)
- Voltage L1 (V)
- Current L1 (A)
- Power Factor
- Frequency (Hz)

**Important prerequisites:**
- Provided your meter was replaced by a Landis-Gyr one post mid-2025 the communication port should be enabled by default. If you test this code and it's logs are like this:
                AxdrParser: done, 0 objects found, 59/59 bytes consumed
                No COSEM objects found in AXDR payload
then you must contact EAC to request activation of the M-Bus data push interface on your specific meter. The port is physically present but disabled.
- EAC may take several days to push the configuration update remotely.

---

## Hardware Required

| Component | Notes |
|---|---|
| ESP32-C6 SuperMini | Any ESP32-C6 board works but pin numbers may differ |
| M-Bus Slave to TTL converter | The common Chinese green PCB module with EL817 optocouplers and FC722/TSS721 IC |
| RJ11 cable | Middle two pins wired only (pins 3 and 4) |
| Power supply | USB-C 5V for testing; see Power Saving section for battery |

**M-Bus module notes:**
- The module requires **5V input** on VCC (not 3.3V) — use the ESP32's 5V pin when powered via USB
- M-Bus terminals are polarity-insensitive — swap if no signal
- The module generates the ~24V M-Bus voltage internally from 5V input

---

## Wiring

```
Landis+Gyr E450 Meter
        │
    RJ11/RJ12
    (middle 2 pins, polarity insensitive)
        │
┌───────────────┐
│  M-Bus TTL    │
│  Converter    │
│               │
│  VCC ─────── ESP32 5V pin
│  GND ─────── ESP32 GND
│  TXD ─────── ESP32 GPIO17 (RX)
│  RXD ─────── ESP32 GPIO16 (TX)
└───────────────┘
```

**Pin mapping (ESP32-C6 SuperMini):**

| M-Bus Module | ESP32-C6 SuperMini |
|---|---|
| VCC | 5V |
| GND | GND |
| TXD | GPIO17 |
| RXD | GPIO16 |

> **Note:** TX/RX are swapped relative to what you might expect. If you get no data, swap GPIO16 and GPIO17.

---

## Software Setup

### 1. ESPHome Version

You need **ESPHome 2026.6.0 or later** which includes the native `dlms_meter` component.

- **ESPHome Device Builder (Desktop app):** Download from [esphome.io](https://esphome.io). Switch to the **beta channel** in the system tray if the stable channel hasn't updated yet.
- **Home Assistant Add-on:** Update via Settings → Add-ons → ESPHome.

### 2. Critical: ESPHome PR Fix

The standard `dlms_meter` component does not correctly parse the specific HDLC frame format used by the Cyprus E450 meter. You must include the fix from ESPHome PR #16942 in your configuration:

```yaml
external_components:
  - source: github://pr#16942
    components: [dlms_meter]
    refresh: 1s
```

> Once PR #16942 is merged into ESPHome mainline, this `external_components` block can be removed.

### 3. secrets.yaml

Add to your `secrets.yaml`:
```yaml
wifi_ssid: "YourWiFiSSID"
wifi_password: "YourWiFiPassword"
landis_api_encryption_key: "YourGeneratedEncryptionKey"
```

Generate an encryption key in ESPHome Device Builder when creating the device.

---

## ESPHome Configuration

Save as `landis-meter.yaml`:

```yaml
esphome:
  name: landis-meter
  friendly_name: Landis+Gyr E450

esp32:
  board: esp32-c6-devkitc-1
  framework:
    type: esp-idf

# Required fix for Cyprus E450 HDLC frame format
external_components:
  - source: github://pr#16942
    components: [dlms_meter]
    refresh: 1s

logger:
  level: DEBUG

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  power_save_mode: none
  fast_connect: true

api:
  encryption:
    key: !secret landis_api_encryption_key

ota:
  - platform: esphome

uart:
  id: mbus_uart
  tx_pin: GPIO16
  rx_pin: GPIO17
  baud_rate: 2400
  rx_buffer_size: 2048
  data_bits: 8
  parity: NONE
  stop_bits: 1

dlms_meter:
  uart_id: mbus_uart

sensor:
  - platform: dlms_meter
    obis_code: "1-0:1.7.0"
    name: "Grid Power Import"
    unit_of_measurement: W
    device_class: power
    state_class: measurement
    accuracy_decimals: 1

  - platform: dlms_meter
    obis_code: "1-0:2.7.0"
    name: "Grid Power Export"
    unit_of_measurement: W
    device_class: power
    state_class: measurement
    accuracy_decimals: 1

  - platform: dlms_meter
    obis_code: "1-0:1.8.0"
    name: "Grid Energy Import Total"
    unit_of_measurement: kWh
    device_class: energy
    state_class: total_increasing
    accuracy_decimals: 3

  - platform: dlms_meter
    obis_code: "1-0:2.8.0"
    name: "Grid Energy Export Total"
    unit_of_measurement: kWh
    device_class: energy
    state_class: total_increasing
    accuracy_decimals: 3

  - platform: dlms_meter
    obis_code: "1-0:3.7.0"
    name: "Reactive Power Import"
    unit_of_measurement: VAr
    device_class: reactive_power
    state_class: measurement
    accuracy_decimals: 1

  - platform: dlms_meter
    obis_code: "1-0:4.7.0"
    name: "Reactive Power Export"
    unit_of_measurement: VAr
    device_class: reactive_power
    state_class: measurement
    accuracy_decimals: 1

  - platform: dlms_meter
    obis_code: "1-0:3.8.0"
    name: "Reactive Energy Import Total"
    unit_of_measurement: kVArh
    device_class: reactive_energy
    state_class: total_increasing
    accuracy_decimals: 3

  - platform: dlms_meter
    obis_code: "1-0:4.8.0"
    name: "Reactive Energy Export Total"
    unit_of_measurement: kVArh
    device_class: reactive_energy
    state_class: total_increasing
    accuracy_decimals: 3

  - platform: dlms_meter
    obis_code: "1-0:32.7.0"
    name: "Voltage L1"
    unit_of_measurement: V
    device_class: voltage
    state_class: measurement
    accuracy_decimals: 1

  - platform: dlms_meter
    obis_code: "1-0:31.7.0"
    name: "Current L1"
    unit_of_measurement: A
    device_class: current
    state_class: measurement
    accuracy_decimals: 2
    filters:
      - multiply: 0.01

  - platform: dlms_meter
    obis_code: "1-0:13.7.0"
    name: "Power Factor"
    device_class: power_factor
    state_class: measurement
    accuracy_decimals: 3
    filters:
      - multiply: 0.001

  - platform: dlms_meter
    obis_code: "1-0:14.7.0"
    name: "Frequency"
    unit_of_measurement: Hz
    device_class: frequency
    state_class: measurement
    accuracy_decimals: 2
    filters:
      - multiply: 0.01
```

---

## Home Assistant Integration

Once flashed and connected, the device will appear automatically in Home Assistant via the ESPHome integration.

**Energy Dashboard setup:**
- Go to Settings → Dashboards → Energy
- Add `Grid Energy Import Total` as grid consumption
- Add `Grid Energy Export Total` as grid return (for solar/netmetering)

---

## Power Saving (Optional)

For battery-powered installations, add deep sleep to reduce average current draw from ~150mA to ~37mA:

```yaml
deep_sleep:
  id: deep_sleep_control
  run_duration: 25s
  sleep_duration: 35s

# Add this switch to allow OTA updates without physical access
switch:
  - platform: template
    name: "Prevent Sleep"
    id: prevent_sleep
    optimistic: true
    restore_mode: ALWAYS_OFF
    turn_on_action:
      - deep_sleep.prevent: deep_sleep_control
    turn_off_action:
      - deep_sleep.allow: deep_sleep_control
```

**OTA update procedure with deep sleep enabled:**
1. Watch for device to come online in HA (every 60 seconds)
2. Immediately turn on **Prevent Sleep** switch
3. Flash OTA from ESPHome Device Builder
4. Turn **Prevent Sleep** off when done

**Battery life estimates (10,000mAh power bank):**
- Deep sleep enabled: ~9 days
- Always on: ~2.3 days

> ⚠️ Power banks with auto-shutoff will cut power during deep sleep. Use a LiPo battery with TP4056 charger + MP1584 buck converter (set to 3.3V) for permanent installation.

**Permanent battery wiring:**
```
LiPo+ → TP4056 BAT+
LiPo- → TP4056 BAT-
TP4056 OUT+ → MP1584 IN+
TP4056 OUT- → MP1584 IN-
MP1584 OUT+ (3.3V) → ESP32 3V3 pin
MP1584 OUT- → ESP32 GND
MP1584 OUT+ (3.3V) → M-Bus module VCC (adjust module to accept 3.3V or use separate 5V source)
```

---

## Troubleshooting

### All sensors show Unknown
- EAC has not yet activated the M-Bus push interface on your meter
- Contact EAC and request M-Bus data push activation for your meter serial number
- After activation, a meter power cycle may be required

### All sensors show 0
- The `external_components` PR fix is not applied — the Cyprus E450 uses a specific HDLC frame format not supported by the standard component
- Ensure your ESPHome version is 2026.6.0+
- Check that `parity: NONE` is set (not EVEN)

### No data / complete silence in logs
- Check M-Bus module VCC is receiving 5V
- Measure M-Bus terminals — should be ~24V from meter
- Try swapping TX/RX pins (GPIO16 ↔ GPIO17)
- Verify RJ11 is firmly seated in meter RJ12 socket

### WiFi disconnecting frequently
- Add `fast_connect: true` to wifi config
- Pin to a specific AP using `bssid:` if you have a mesh network
- Avoid deep sleep if power bank cuts out during sleep

### OTA updates failing with deep sleep
- Enable the Prevent Sleep switch before attempting OTA
- Or connect USB-C directly for physical flash

### Energy totals show Unknown while other sensors work
- Increase `run_duration` to 25s+ to catch all frame sequences
- Energy totals are sometimes in a separate frame broadcast

---

## Known Issues

- **PR #16942 not yet merged:** The Cyprus E450 HDLC frame fix is in a pending ESPHome pull request. The `external_components` workaround is required until it merges.
- **Deep sleep + power banks:** Most power banks auto-shutoff when current drops during deep sleep. Use a raw LiPo for permanent installs.
- **EAC activation required:** The M-Bus port must be explicitly enabled by EAC per meter. This is not automatic.

---

## Confirmed Working Configuration

| Parameter | Value |
|---|---|
| Meter | Landis+Gyr E450 (EAC Cyprus) |
| Protocol | DLMS/COSEM over wired M-Bus |
| Baud rate | 2400 bps |
| Parity | NONE |
| Data bits | 8 |
| Stop bits | 1 |
| Frame format | HDLC (Cyprus-specific variant) |
| Encryption | None (EAC Cyprus does not encrypt) |
| Push interval | ~5 seconds |
| ESPHome version | 2026.6.0+ |
| ESP32 board | ESP32-C6 SuperMini |

---

## Credits

- ESPHome PR #16942 by [@PolarGoose](https://github.com/PolarGoose) — Cyprus E450 HDLC frame format fix
- ESPHome `dlms_meter` component team
- EAC Cyprus engineering team for technical assistance
- Home Assistant community

---

## License

MIT License — free to use, modify and share. If this helped you, consider opening a PR or leaving a star ⭐
