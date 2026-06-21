# esphome-deye-lite

> **Draft** — work in progress, not ready for use.

**Local Modbus monitoring and smart export control for Deye hybrid inverters. No cloud, no subscription.**

> ESP32-based module that connects to your Deye inverter's RS485 Modbus port and reads inverter data every 5 seconds. The built-in web interface is accessible from any browser on your local network. Connecting to Home Assistant adds live dashboards and energy monitoring.

![Module photo](docs/module.jpg)

---

## Contents

- [What it does](#what-it-does)
- [Hardware](#hardware)
- [Software](#software)
- [Setup](#setup)
- [How to use](#how-to-use)
- [Sensors reference](#sensors-reference)
- [Troubleshooting](#troubleshooting)
- [Compatibility](#compatibility)
- [License](#license)

---

## What it does

The module reads your inverter's internal registers over Modbus RS485 and makes the data available locally via a built-in web interface. No Home Assistant, no MQTT broker, no internet connection required.

It reports:

- **Solar production:** PV1 and PV2 power, voltage, current; daily and total yield
- **Grid:** power (import/export), voltage; daily and total bought/sold energy
- **Battery:** state of charge, power, current, voltage, temperature; daily and total charge/discharge energy
- **House load:** real-time consumption power; daily and total energy
- **Inverter health:** DC and AC temperatures

Fast sensors update every 5 seconds. Energy totals and temperatures update every 60 seconds to avoid saturating the Modbus bus.

The module also includes **SmartInject**, an automatic export voltage control algorithm that prevents grid overvoltage faults during solar export. See [Software](#software) for details.

---

## Hardware

### What's in the box

- ESP32 module PCB with RS485 transceiver
- 2-pin JST connector for Modbus connection (A / B)
- 2-pin JST connector for 12 V DC power input

![Hardware photo](docs/hardware.jpg)

### Wiring

Connect the module to your Deye inverter's **Modbus RS485 port**. This is typically a 4-pin connector on the inverter's communication board labelled **RS485** or **Modbus**.

| Module | Inverter Modbus port |
|--------|----------------------|
| RS485 A | A (or +) |
| RS485 B | B (or −) |
| 12 V | 12 V auxiliary output |
| GND | GND |

> **Cable:** use twisted-pair cable for the A/B signal lines. Standard Cat5e with one pair for A/B and another pair for power works well for runs up to 10 m.

![Wiring diagram](docs/wiring.jpg)

---

## Software

### SmartInject: export voltage control

#### The problem it solves

When your solar panels produce more than your home can consume, your inverter pushes the surplus to the grid. If many homes in your street do the same simultaneously, the local grid voltage rises. When it rises above the inverter's protection threshold, your inverter trips an **overvoltage fault** and stops exporting, sometimes repeatedly throughout the day.

The inverter has a built-in **Max Sell Power** setting that caps grid export, but it is a fixed value and cannot react to what the grid is actually doing at any moment. SmartInject makes this limit dynamic.

#### How it works

SmartInject watches the grid voltage reading (updated every 5 seconds) and adjusts the Max Sell Power limit automatically:

- **No PV (< 1 W):** nothing to regulate; algorithm skips
- **Low PV (< 100 W):** limit resets to 1000 W so the system starts fresh each morning
- **Voltage above threshold (≥ 253 V) and currently exporting:** limit steps down proportionally
  - 253–254 V: −250 W
  - 254–255 V: −500 W
  - above 255 V: −1000 W
  - Floor: 500 W (never cuts export entirely)
- **Voltage OK and export near the limit:** limit raises by 250 W (up to 6000 W)

After every adjustment an **injection ceiling clamp** runs: the limit is capped to `actual export + 500 W` to prevent voltage spikes when the battery finishes charging and the inverter suddenly has extra power available.

#### What SmartInject does not do

SmartInject is a protective mechanism, not a power optimiser. It only ever writes to the **Max Sell Power** register (register 245). It does not touch battery charge/discharge settings, time-of-use schedules, or any other inverter configuration.

When voltage is high it will reduce your export. This is intentional: the alternative is a fault trip that stops export entirely.

---

## Setup

### 1. First boot and Wi-Fi

1. Power the module from the inverter's 12 V auxiliary output (or a bench supply during setup).
2. Within about 10 seconds the module broadcasts a Wi-Fi hotspot named **`Deye-Lite Fallback Hotspot`**.
3. Connect your phone or laptop to the hotspot. The **hotspot password** is printed on the label on the module.
4. A captive portal opens automatically. If it doesn't, navigate to `192.168.4.1` in your browser.
5. Enter your home Wi-Fi network name and password and tap **Save**.
6. The module reboots and connects to your network. The hotspot disappears.

> The fallback hotspot reappears any time the module cannot reach a known Wi-Fi network, for example after a router change. Re-run the captive portal to reconfigure.

### 2. Adding to Home Assistant *(optional)*

Once the module is on your network, Home Assistant will detect it automatically via mDNS.

1. In Home Assistant, go to **Settings → Devices & Services**.
2. You should see a new ESPHome device discovered. Click **Configure**.
3. Enter the **API encryption key** printed on the label on the module.
4. Click **Submit**. All entities are added immediately.

> If the device is not discovered automatically after a minute or two, go to **Settings → Devices & Services → Add Integration → ESPHome** and enter the module's IP address manually.

### 3. OTA firmware updates *(optional)*

All firmware updates are pushed over Wi-Fi.

**Via ESPHome dashboard** (recommended):
- The [ESPHome device builder](https://esphome.io/guides/getting_started_hassio.html) is available as a Home Assistant add-on or as a [standalone desktop application](https://esphome.io/guides/getting_started_command_line.html). The module advertises itself via mDNS and will appear automatically as an adopted device. Click **Install** to compile and push the update wirelessly.

**Via the web interface:**
- The built-in web server provides a firmware upload page accessible from your browser. No ESPHome installation required.

**Via CLI:**
```bash
esphome run deye-lite.yaml --device <IP address>
```

---

## How to use

### Web interface

The module runs a local web server on port 80. Access it from any browser on your network:

```
http://deye-lite-<last 6 digits of MAC>.local
```

Or use the module's IP address directly if mDNS is not available on your network.

**Username:** printed on the label on the module
**Password:** printed on the label on the module

The web interface shows all live sensor values and lets you:
- Toggle the SmartInject switch
- Adjust the Max Sell Power limit manually
- Trigger a remote restart

This works entirely without Home Assistant.

### Home Assistant entities

Once adopted, all sensors appear automatically as entities named `deye-lite-<MAC>_<sensor>`. The MAC-based suffix ensures no conflicts if you have multiple modules on the same network.

#### Energy dashboard

The module maps directly into the **Home Assistant Energy dashboard** (**Settings → Energy**). Recommended slots for a typical setup:

| HA Energy slot | Entity | Notes |
|---|---|---|
| Solar production | `..._Total_PV_Prod` | |
| Grid consumption | `..._Total_Energy_Bought` | |
| Grid return | `..._Total_Energy_Sold` | |
| Battery in *(optional)* | `..._Total_Batt_Chg` | only if battery fitted |
| Battery out *(optional)* | `..._Total_Batt_Dchg` | only if battery fitted |

> Entity names above use `deye-inverter` as the example hostname prefix. Your prefix will include your module's MAC suffix; check your device page in Home Assistant for the exact names.

### SmartInject switch

SmartInject is **on by default**. Toggle the `SmartInject` switch entity in Home Assistant or the web interface to enable or disable it at any time. When disabled, the Max Sell Power value stays at whatever it was last set to. It is not reset automatically.

### Max Sell Power

The `Max_Sell_Power` number entity shows the export limit currently set on the inverter (in watts) and lets you override it manually. SmartInject writes to this same value. If SmartInject is on, any manual value you set will be overwritten on the next voltage reading.

---

## Sensors reference

| Entity | Unit | Update interval |
|---|---|---|
| PV1 Power | W | 5 s |
| PV2 Power | W | 5 s |
| PV Power (total) | W | 5 s |
| PV1 / PV2 Voltage | V | 60 s |
| PV1 / PV2 Current | A | 60 s |
| Daily PV Production | kWh | 60 s |
| Total PV Production | kWh | 60 s |
| Grid Power | W | 5 s |
| Grid Voltage | V | 5 s |
| Daily Energy Bought / Sold | kWh | 60 s |
| Total Energy Bought / Sold | kWh | 60 s |
| House Power | W | 5 s |
| Daily / Total House Energy | kWh | 60 s |
| Battery SOC | % | 60 s |
| Battery Power | W | 5 s |
| Battery Current | A | 60 s |
| Battery Voltage | V | 60 s |
| Battery Temperature | °C | 60 s |
| DC Temperature | °C | 60 s |
| AC Temperature | °C | 60 s |
| Max Sell Power | W | writable |

---

## Troubleshooting

**Module not appearing in Home Assistant after first boot**
- Confirm the module joined your Wi-Fi (check your router's client list for a device starting with `deye-lite-`)
- If it's not connected, the fallback hotspot should be active; re-run the captive portal
- Try adding the integration manually via IP address

**All sensors show `unavailable`**
- Check RS485 wiring: A and B may be swapped; try reversing them
- Confirm the inverter's Modbus function is enabled (consult your inverter manual)
- Verify the module is powered

**SmartInject is not adjusting the export limit**
- Check that the `SmartInject` switch is on
- Verify `Grid Voltage` and `PV Power` are showing valid readings; SmartInject only runs when all required sensors have data
- If Max Sell Power shows a fixed value, the inverter may use a different register address for your model; check the compatibility notes below

**Fallback hotspot not appearing**
- Power-cycle the module
- The hotspot takes up to 30 seconds to appear after boot

**Web interface not reachable**
- Try the IP address instead of the `.local` hostname
- Some Android devices have issues with mDNS; use the IP from your router's DHCP table

---

## Compatibility

Tested on Deye single-phase hybrid inverters (SUN-xK-SG series). Register addresses follow the standard Deye/Sunsynk/Sol-Ark Modbus map and are expected to work on compatible inverters from those families, but this is not guaranteed for all models and firmware versions.

If your inverter uses a different register address for Max Sell Power than register 245, SmartInject will need to be adjusted accordingly.

---

## License

MIT. See [LICENSE](LICENSE).
