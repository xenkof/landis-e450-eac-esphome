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
- Reactive power import/export (VAr) and reactive energy totals (kVArh)
- Voltage L1/L2/L3 (V)
- Current L1/L2/L3 (A)
- Power Factor
- Frequency (Hz)

> Single phase installations (most residential EAC customers) can simply remove the L2 and L3 sensors — all sections are clearly marked in the YAML.

**Important prerequisites:**

- Meters replaced by EAC post mid-2025 may have the M-Bus communication port enabled by default. Flash the firmware and check your logs. If you see:
  ```
  AxdrParser: done, 0 objects found, 59/59 bytes consumed
  No COSEM objects found in AXDR payload
  ```
  then you must contact EAC to request activation of the M-Bus data push interface on your specific meter. The port is physically present on all E450 meters but may be disabled by default depending on when it was installed.
- EAC may take several days to push the configuration update remotely to your meter.

---

## Hardware Required

| Component | Notes |
|---|---|
| ESP32-C6 SuperMini | Any ESP32-C6 board works but pin numbers may differ |
| M-Bus Slave to TTL converter | The common Chinese green PCB module with EL817 optocouplers and FC722/TSS721 IC |
| RJ11 cable | Middle two pins wired only (pins 3 and 4) |
| Power supply | USB-C 5V for testing; see [Power Saving](#power-saving-optional) for permanent install |

**M-Bus module notes:**
- The module requires **5V input** on VCC — use the ESP32's 5V pin when powered via USB-C
- M-Bus terminals are polarity-insensitive — if you get no signal, swap the two wires
- The module generates the ~24V M-Bus voltage internally from the 5V input
- The meter's RJ12 socket accepts a standard RJ11 plug — only the middle two pins are needed

---

## Wiring

```
Landis+Gyr E450 Meter
        │
    RJ11 into RJ12 socket
    (middle 2 pins, polarity insensitive)
        │
┌───────────────────┐
│  M-Bus TTL        │
│  Converter Module │
│                   │
│  VCC ─────────── ESP32 5V pin
│  GND ─────────── ESP32 GND
│  TXD ─────────── ESP32 GPIO17 (RX)
│  RXD ─────────── ESP32 GPIO16 (TX)
└───────────────────┘
```

**Pin mapping (ESP32-C6 SuperMini):**

| M-Bus Module | ESP32-C6 SuperMini |
|---|---|
| VCC | 5V |
| GND | GND |
| TXD | GPIO17 |
| RXD | GPIO16 |

> **Note:** TX and RX are swapped relative to standard convention on this module. If you get no data at all, try swapping GPIO16 and GPIO17 in your YAML first before changing anything else.

---

## Software Setup

### 1. ESPHome Version

You need **ESPHome 2026.6.0 or later** which includes the native `dlms_meter` component.

- **ESPHome Device Builder (Desktop app):** Download from [esphome.io](https://esphome.io). If the stable channel still shows an older version, switch to the **beta channel** via the system tray icon.
- **Home Assistant Add-on:** Update via Settings → Add-ons → ESPHome → Update.

### 2. Critical: ESPHome PR Fix

The standard `dlms_meter` component does not correctly parse the specific HDLC frame variant used by the Cyprus E450 meter. Without this fix, all sensor values will read as zero even when the meter is broadcasting correctly. Add the following block to your configuration:

```yaml
external_components:
  - source: github://pr#16942
    components: [dlms_meter]
    refresh: 1s
```

> Once [PR #16942](https://github.com/esphome/esphome/pull/16942) is merged into ESPHome mainline, this `external_components` block can be removed.

### 3. secrets.yaml

Add the following to your `secrets.yaml` file:

```yaml
wifi_ssid: "YourWiFiSSID"
wifi_password: "YourWiFiPassword"
landis_api_encryption_key: "YourGeneratedEncryptionKey"
```

Generate an API encryption key in ESPHome Device Builder when creating the device, or use the one from an existing device.

---

## ESPHome Configuration

Two YAML files are provided in this repository:

| File | Use case |
|---|---|
| `landis-meter-single-phase.yaml` | Single phase installations (most EAC residential customers) |
| `landis-meter-three-phase.yaml` | Three phase installations — includes L2/L3 voltage and current sensors |

Both files use the same architecture: raw internal DLMS sensors with scaling applied via template sensors visible in Home Assistant.

**Confirmed working parameters for EAC Cyprus Landis+Gyr E450:**

| Parameter | Value |
|---|---|
| Baud rate | 2400 bps |
| Parity | NONE |
| Data bits | 8 |
| Stop bits | 1 |
| Frame format | HDLC (Cyprus-specific variant — requires PR #16942) |
| Encryption | None |
| Push interval | ~5 seconds |

**Scaling factors confirmed for this meter:**

| Measurement | Raw unit | Scale factor | Final unit |
|---|---|---|---|
| Energy | Wh | × 0.001 | kWh |
| Current | mA | × 0.01 | A |
| Power Factor | ×1000 | × 0.001 | dimensionless |
| Reactive Power | mVAr | × 0.001 | VAr |
| Power / Voltage / Frequency | — | none | W / V / Hz |

---

## Home Assistant Integration

Once flashed and connected, the device appears automatically in Home Assistant via the ESPHome integration.

**Energy Dashboard setup:**
1. Go to Settings → Dashboards → Energy
2. Under **Grid**, add:
   - `Grid Energy Import Total` as **Grid consumption**
   - `Grid Energy Export Total` as **Return to grid** (for solar/netmetering customers)

---

## Power Saving (Optional)

For battery-powered installations, add deep sleep to reduce average current draw:

- Always on (WiFi active): ~150mA → ~2.3 days on 10,000mAh
- Deep sleep (25s on / 35s off): ~37mA average → ~9 days on 10,000mAh

Add the following to your YAML:

```yaml
deep_sleep:
  id: deep_sleep_control
  run_duration: 25s
  sleep_duration: 35s

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

**OTA update procedure with deep sleep active:**
1. Watch for the device to come online in HA (reconnects every ~60 seconds)
2. Immediately toggle **Prevent Sleep** switch ON during the wake window
3. Flash OTA from ESPHome Device Builder
4. Toggle **Prevent Sleep** OFF when complete

> ⚠️ **Power bank warning:** Most USB power banks have auto-shutoff protection that cuts power when current drops during deep sleep. For permanent installation use a LiPo battery with a TP4056 charger board and MP1584 buck converter set to 3.3V output.

**Permanent battery wiring:**
```
LiPo+  →  TP4056 BAT+
LiPo-  →  TP4056 BAT-
TP4056 OUT+  →  MP1584 IN+
TP4056 OUT-  →  MP1584 IN-
MP1584 OUT+ (3.3V)  →  ESP32 3V3 pin
MP1584 OUT- (GND)   →  ESP32 GND
MP1584 OUT+ (3.3V)  →  M-Bus module VCC (or use separate 5V source for module)
```

---

## Troubleshooting

### Sensors show Unknown
- EAC has not yet activated the M-Bus push interface on your meter
- Contact EAC and request M-Bus data push activation — provide your meter serial number
- Check logs for: `No COSEM objects found in AXDR payload` which confirms port is inactive
- After EAC pushes the update, a meter power cycle by EAC may be required

### Sensors show 0 for everything including voltage
- The `external_components` PR #16942 fix is missing or not applied correctly
- This is the most common issue — the standard `dlms_meter` component cannot parse the Cyprus E450 HDLC frame variant and returns zeros for all values
- Verify `parity: NONE` (not EVEN)
- Verify ESPHome version is 2026.6.0+

### No data at all / complete silence in logs
- Check M-Bus module VCC is receiving 5V (not 3.3V)
- Measure across M-Bus terminals — should read ~24V DC from the meter
- Try swapping TX/RX pins (GPIO16 ↔ GPIO17) in your YAML
- Confirm RJ11 plug is firmly seated in the meter's RJ12 socket
- M-Bus terminals are polarity-insensitive — try swapping the two wires

### WiFi connecting and disconnecting repeatedly
- Add `fast_connect: true` to your wifi config
- On mesh/multi-AP networks, add `bssid: "XX:XX:XX:XX:XX:XX"` to pin to the nearest AP
- Check router logs for roaming loop — the ESP32-C6 can trigger aggressive roaming on some mesh systems

### OTA updates failing with deep sleep enabled
- Turn on the **Prevent Sleep** switch during a wake window before triggering OTA
- Alternatively connect USB-C directly for a physical flash

### Energy totals show Unknown while instantaneous sensors work
- Increase `run_duration` to 25s or more — energy totals may arrive in a later frame
- The meter broadcasts multiple frames per cycle; shorter wake windows may miss them

---

## Known Issues

- **PR #16942 not yet merged:** The Cyprus E450 HDLC frame fix is in a pending ESPHome pull request. The `external_components` workaround is required until it merges into mainline. Track progress at [github.com/esphome/esphome/pull/16942](https://github.com/esphome/esphome/pull/16942).
- **Deep sleep + power banks:** Most power banks auto-shutoff during deep sleep. Use a raw LiPo for permanent installs.
- **EAC activation:** The M-Bus port must be explicitly enabled by EAC on older meter installations. Newer meters (post mid-2025) may have it enabled by default.
- **Three phase L2/L3 sensors:** Not tested by the original author (single phase installation). Community feedback welcome.

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
| Encryption | None |
| Push interval | ~5 seconds |
| ESPHome version | 2026.6.0b1+ |
| ESP32 board | ESP32-C6 SuperMini |

---

## Credits

- [PR #16942](https://github.com/esphome/esphome/pull/16942) by [@PolarGoose](https://github.com/PolarGoose) — Cyprus E450 HDLC frame format fix without which this would not work
- ESPHome `dlms_meter` component contributors
- EAC Cyprus engineering team for technical assistance during testing
- Home Assistant and ESPHome communities

---

## License

MIT — free to use, modify and share with attribution. If this project helped you, consider opening a PR, leaving a ⭐, or posting your experience in the ESPHome community forums.
