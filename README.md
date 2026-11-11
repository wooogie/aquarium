# KC868-A6 Interval Timer – ESPHome

Standalone interval timer for **Relay 1 & 2** on a KinCony KC868-A6 (ESP32).  
All parameters are configurable via the built-in web UI at `http://<device-ip>` — no Home Assistant required.

---

## Hardware

| Board | KinCony KC868-A6 |
|---|---|
| MCU | ESP32 (dual-core, 240 MHz) |
| Relays | 6 × relay outputs via PCF8574 I/O expander (I²C 0x24) |
| Inputs | 6 × digital inputs via PCF8574 I/O expander (I²C 0x22) |
| RTC | DS1307 (battery-backed, I²C) |
| 1-Wire | GPIO32 (IO-1), GPIO33 (IO-2) |
| Power | 12 V DC |
| USB | USB-C (flashing / serial) |

### DS18B20 temperature sensors

Connect one sensor per 1-Wire line:

```
DS18B20 pin 1 (GND)  → GND
DS18B20 pin 2 (DATA) → GPIO32 (Sensor 1)  or  GPIO33 (Sensor 2)
DS18B20 pin 3 (VCC)  → 3.3 V

Pull-up: 4.7 kΩ resistor between DATA and 3.3 V on each line
```

---

## Prerequisites

### Install ESPHome

```bash
pip3 install esphome --break-system-packages
# or inside a venv:
python3 -m venv .venv && source .venv/bin/activate
pip install esphome
```

Verify:
```bash
esphome version
```

### Configure credentials

Edit `secrets.yaml` (never commit this file):

```yaml
wifi_ssid: "YourNetworkName"
wifi_password: "YourPassword"
ota_password: "choose-something-random"
```

Adjust the timezone in `kc868-a6-timer.yaml` if needed (default: `Europe/Zurich`):
```yaml
timezone: "Europe/Zurich"   # e.g. Europe/Berlin, America/New_York
```

---

## Build (compile only, no device needed)

```bash
cd /home/silvan/aquarium
esphome compile kc868-a6-timer.yaml
```

Output firmware files:
```
.esphome/build/kc868-a6-timer/build/firmware.factory.bin   ← first flash via USB
.esphome/build/kc868-a6-timer/build/firmware.ota.bin       ← OTA updates
```

---

## Flash via USB (first time)

The KC868-A6 must be put into **download mode** manually:

1. Keep the board **unpowered**
2. Connect USB-C to your computer
3. **Hold the S2 button** on the board
4. While holding S2, attach the **12 V power supply**
5. Release S2 after ~1 second
6. Run:

```bash
esphome upload kc868-a6-timer.yaml
```

ESPHome auto-detects the serial port. If it fails, specify it explicitly:

```bash
esphome upload kc868-a6-timer.yaml --device /dev/ttyUSB0
# list available ports:
ls /dev/ttyUSB* /dev/ttyACM*
```

> **Linux udev note:** if you get a permission error on the serial port, add yourself to the `dialout` group and re-login:
> ```bash
> sudo usermod -aG dialout $USER
> ```

---

## Flash via OTA (subsequent updates)

Once the device is running on WiFi, all future updates can be done wirelessly:

```bash
esphome upload kc868-a6-timer.yaml
```

ESPHome discovers the device via mDNS (`kc868-a6-timer.local`) automatically.  
If mDNS doesn't work on your network, pass the IP directly:

```bash
esphome upload kc868-a6-timer.yaml --device 192.168.1.xxx
```

The OTA password is set in `secrets.yaml` (`ota_password`).

---

## Monitor logs

```bash
# via USB (device must be connected)
esphome logs kc868-a6-timer.yaml

# via network (WiFi must be connected)
esphome logs kc868-a6-timer.yaml --device 192.168.1.xxx
```

---

## Web interface

After boot, open `http://kc868-a6-timer.local` (or `http://<ip>`) in any browser.

| Card | Description |
|---|---|
| **Current Time** | Live clock (NTP-synced, RTC fallback) |
| **Time Source** | Shows `NTP (synced)` or `RTC (no WiFi)` |
| **Timer Enable** | Master on/off toggle for the interval timer |
| **ON Duration** | How long Relay 1 & 2 stay energised (minutes) |
| **OFF Duration** | How long Relay 1 & 2 stay off (minutes) |
| **Daytime Start** | Hour when the timer becomes active (0–23) |
| **Daytime End** | Hour when the timer stops (0–23) |
| **Timer Status** | Live status: `ON – 47 s remaining (2 min total)` |
| **Timer Phase Remaining** | Countdown in seconds |
| **Temperature 1 / 2** | DS18B20 readings (updated every 30 s) |
| **Relay 3–6** | Manual independent switches |
| **WiFi Signal / Uptime** | Diagnostics |

---

## Time & RTC behaviour

| Situation | Clock source |
|---|---|
| Boot, WiFi available | DS1307 RTC seeds clock on boot → NTP overrides within ~5 s |
| Boot, no WiFi | DS1307 RTC provides time (battery-backed, stays accurate for years) |
| WiFi drops mid-run | ESP32 internal timer keeps ticking from last NTP sync |
| NTP syncs | Accurate time written back to DS1307 hardware RTC |

The timer will **always work** as long as the DS1307 has been set at least once (which happens automatically the first time NTP syncs).

---

## Timer logic

```
[Boot / daytime start]
        ↓
┌─── OFF phase ─────────────────────────────┐
│  Relay 1 & 2 OFF  │  count → off_duration  │
│       when done → switch to ON phase       │
└───────────────────────────────────────────┘
        ↓
┌─── ON phase ──────────────────────────────┐
│  Relay 1 & 2 ON   │  count → on_duration   │
│       when done → switch to OFF phase      │
└───────────────────────────────────────────┘
        ↓
   (loop until daytime window ends or timer disabled)
```

Outside the daytime window: relays are forced off and the cycle resets.  
Changing any parameter resets the current cycle.

---

## File overview

```
aquarium/
├── kc868-a6-timer.yaml   main ESPHome configuration
├── secrets.yaml          WiFi & OTA credentials  ← never commit this
└── README.md             this file
```
