# Setup Guide

This guide walks through the full installation and configuration of the Agile Energy Display, from sourcing hardware through to a working display.

---

## Contents

1. [Hardware](#1-hardware)
2. [ESPHome installation](#2-esphome-installation)
3. [Finding your meter identifiers](#3-finding-your-meter-identifiers)
4. [Home Assistant templates](#4-home-assistant-templates)
5. [ESPHome configuration](#5-esphome-configuration)
6. [Flashing the device](#6-flashing-the-device)
7. [Adding to Home Assistant](#7-adding-to-home-assistant)
8. [Verifying everything works](#8-verifying-everything-works)
9. [Multiple devices](#9-multiple-devices)

---

## 1. Hardware

### Display board

This project is designed for the **ESP32-4848S040C_1**, commonly sold on AliExpress as the **"ESP32 4.0 inch capacitive touch"** display board. Search for `ESP32-4848S040C` to find it. It is made by Sunton and available from several sellers.

Key specs:

| Feature | Detail |
|---|---|
| Model | ESP32-4848S040C_1 |
| SoC | ESP32-S3 |
| Display | 480×480 ST7701S RGB panel |
| Touch | GT911 capacitive |

These boards typically come with or without an enclosure. The project supports mounting with the USB port either at the bottom or the top — see the [rotation note](#display-rotation) if needed.

### What you'll need

- The ESP32-S3 display board
- USB-C cable (for initial flashing)
- A computer with Python installed (for ESPHome)
- A running Home Assistant instance with the Octopus Energy integration already configured

---

## 2. ESPHome installation

If you don't already have ESPHome, the easiest route is to install the **ESPHome add-on** directly within Home Assistant:

1. In Home Assistant, go to **Settings → Add-ons → Add-on Store**
2. Search for **ESPHome** and install it
3. Start the add-on and enable **Show in sidebar**

Alternatively, install ESPHome as a Python package on your computer:

```bash
pip install esphome
```

The ESPHome dashboard add-on is the recommended approach for most users as it handles compilation, flashing, and OTA updates from a browser interface.

---

## 3. Finding your meter identifiers

The Octopus Energy integration creates entity IDs that embed your meter's serial number and MPAN (Meter Point Administration Number). You need both values before configuring anything.

### Method 1 — via the integration device page

1. Go to **Settings → Devices & Services**
2. Find the **Octopus Energy** integration and click it
3. Click on your electricity meter device
4. Look at any sensor entity — the entity ID will follow this pattern:

   ```
   sensor.octopus_energy_electricity_SERIAL_MPAN_current_rate
   ```

### Method 2 — via Developer Tools

1. Go to **Developer Tools → States**
2. In the filter box, type `octopus_energy_electricity`
3. Find an entity like `sensor.octopus_energy_electricity_23e5102151_1412483603003_current_rate`

From this example:
- **Serial**: `23e5102151` (the segment after `electricity_`)
- **MPAN**: `1412483603003` (the segment before `_current_rate`)

Write these down — you'll use them in both the HA templates and the ESPHome config.

### Confirming the event entities exist

The templates rely on `event` entities provided by the Octopus Energy integration, not just sensors. Confirm these exist in Developer Tools → States:

```
event.octopus_energy_electricity_SERIAL_MPAN_current_day_rates
event.octopus_energy_electricity_SERIAL_MPAN_next_day_rates
```

If these don't appear, check that your Octopus Energy integration is fully configured and that your account has Agile tariff data available. These entities are created automatically by the integration — you don't need to do anything extra to enable them.

---

## 4. Home Assistant templates

The display relies on a set of template sensors in Home Assistant that pre-process the raw rate data into formats the ESPHome device can consume efficiently.

### Adding the templates

1. Open `homeassistant/templates.yaml` from this repository
2. Replace every instance of `SERIAL` and `MPAN` with your values from step 3

   For example, change:
   ```yaml
   state_attr('event.octopus_energy_electricity_SERIAL_MPAN_current_day_rates', 'rates')
   ```
   to:
   ```yaml
   state_attr('event.octopus_energy_electricity_23e5102151_1412483603003_current_day_rates', 'rates')
   ```

   There are multiple occurrences across the file — use your editor's find-and-replace to swap them all in one pass.

3. Add the templates to your Home Assistant configuration. There are two common approaches:

   **If you already have a `templates.yaml`** referenced from `configuration.yaml`:
   Paste the contents of the file into your existing `templates.yaml`.

   **If you don't have one yet**, copy the file to your HA config directory and add this to `configuration.yaml`:
   ```yaml
   template: !include templates.yaml
   ```

4. Restart Home Assistant (or reload template entities via **Developer Tools → YAML → Template entities**)

### Verifying the templates

After restarting, go to **Developer Tools → States** and check these entities exist and have valid (non-`unknown`) states:

| Entity | Expected state |
|---|---|
| `sensor.agile_display_payload` | A timestamp string |
| `sensor.agile_today_average_rate` | A number (pence) |
| `sensor.agile_current_slot_text` | e.g. `14:00-14:30` |
| `sensor.agile_today_high_text` | e.g. `High: 28.50p @ 17:30` |
| `sensor.agile_tomorrow_best_2h_text` | e.g. `02:00-04:00 @ 5.20p` or `Waiting...` |
| `sensor.agile_tomorrow_peak_window_text` | e.g. `16:00-19:00 @ 24.30p` or `Waiting...` |
| `binary_sensor.agile_display_night_mode` | `on` or `off` |

Also check the attributes of `sensor.agile_display_payload` — it should have `today_prices_csv`, `current_slot_index`, `best_remaining_text`, `tomorrow_prices_csv`, and `tomorrow_ready` attributes, all with values.

If any sensors show `unknown` or `unavailable`, the most likely cause is an incorrect serial or MPAN — double-check the entity ID spellings carefully, as the values are case-sensitive.

---

## 5. ESPHome configuration

Open `esphome/agile-energy-display.yaml` and edit the `substitutions` block at the top of the file. This is the **only section you need to change** for a standard installation.

```yaml
substitutions:
  device_name: agile-energy-display        # used as the ESPHome device hostname
  friendly_name: Agile Energy Display      # shown in HA and ESPHome dashboard
  api_key: "REPLACE_WITH_YOUR_API_KEY"     # see below
  ota_password: "REPLACE_WITH_YOUR_OTA_PASSWORD"
  ap_password: "REPLACE_WITH_YOUR_AP_PASSWORD"
  octopus_serial: "YOUR_METER_SERIAL"      # from step 3
  octopus_mpan: "YOUR_MPAN"               # from step 3
```

### Generating secure values

For `api_key`, generate a random base64 string. You can do this in the ESPHome dashboard (it generates one automatically when you create a new device), or from the command line:

```bash
openssl rand -base64 32
```

For `ota_password` and `ap_password`, use any strong random string. A password manager or this command works well:

```bash
openssl rand -hex 16
```

### WiFi credentials

The config uses ESPHome's `secrets.yaml` mechanism for WiFi credentials. Create or edit `secrets.yaml` in the same directory as your ESPHome config (or in the ESPHome config directory if using the add-on):

```yaml
wifi_ssid: "Your WiFi network name"
wifi_password: "Your WiFi password"
```

This file is intentionally not committed to version control — it's listed in `.gitignore`.

---

## 6. Flashing the device

### First flash (USB)

The first time you flash, the device needs to be connected via USB-C.

**Via ESPHome dashboard (add-on):**

1. Open the ESPHome dashboard in Home Assistant
2. Click **+ New Device** and follow the wizard, or click **Edit** on an existing entry if you've already added the YAML manually
3. Click **Install → Plug into this computer**
4. Follow the browser-based flashing tool (requires Chrome or Edge)

**Via CLI:**

```bash
esphome run esphome/agile-energy-display.yaml
```

ESPHome will compile the firmware (this takes a few minutes the first time) and then prompt you to select the serial port.

If the device isn't detected, you need to manually enter download mode. With the board already connected via USB:

1. Hold the **BOOT** button
2. Press and release the **RESET** button
3. Release **BOOT**

The device should now be visible as a serial port and ready to flash.

### Subsequent updates (OTA)

Once the device is on your network, all future updates can be done wirelessly:

```bash
esphome run esphome/agile-energy-display.yaml
```

ESPHome will detect the device on the network and offer OTA as an option. The display backlight will turn off during the OTA update and come back on when complete.

---

## 7. Adding to Home Assistant

1. Once the device is running and connected to WiFi, Home Assistant should detect it automatically
2. A notification will appear in **Settings → Devices & Services** — click **Configure**
3. Enter the `api_key` you set in the substitutions
4. The device will appear as a new integration with a backlight light entity, relay switches, and a status binary sensor

---

## 8. Verifying everything works

After adoption, the display should start showing live data within a minute or so as the sensors subscribe and receive their first values.

### What to expect on first boot

1. The display backlight comes on
2. Placeholder dashes (`--.--p`, `--.- kWh`) are visible for a few seconds
3. Values populate as each sensor receives data from HA
4. The price chart renders once `today_prices_csv` and `current_slot_index` are received

### If values don't populate

**Check the ESPHome logs** — in the ESPHome dashboard, click **Logs** on the device. Look for lines referencing the sensor entity IDs. If you see warnings about unavailable entities, the entity names in your substitutions don't match what's in HA.

**Check HA logs** — go to **Settings → System → Logs** and filter for `octopus`. Any errors from the Octopus Energy integration will appear here.

**Common issues:**

| Symptom | Likely cause |
|---|---|
| All values stuck on dashes | Device connected to HA but no sensor data — check template sensors in HA |
| `sensor.agile_display_payload` is `unknown` | Wrong serial/MPAN in templates.yaml |
| Chart shows but no "now" marker | `current_slot_index` attribute is empty or zero — check the `agile_display_payload` attributes |
| Tomorrow chart shows "Rates not published yet" | Normal before ~16:00–20:00 — Octopus hasn't released next-day rates yet |
| Display stays dim after touch | `binary_sensor.agile_display_night_mode` may be stuck `on` — check the template |

---

## 9. Multiple devices

If you want to run the display at more than one location (for example, two properties with separate Octopus meters):

### Same HA instance, same meter

Simply flash a second device with the same YAML. Change `device_name` and `friendly_name` in substitutions to give it a unique identity, and generate fresh `api_key` and `ota_password` values. The HA template sensors are shared — no changes needed there.

### Different HA instances, different meters

Each HA instance needs its own copy of `templates.yaml` with the relevant serial and MPAN substituted. The template entity names (`sensor.agile_display_payload` etc.) are the same on both instances, which is intentional — the ESPHome device connects to whichever HA instance it's adopted into and subscribes to the same entity names regardless of location.

Each ESPHome device gets its own YAML file with:
- Unique `device_name`, `api_key`, `ota_password`
- The serial and MPAN for the meter at that property

---

## Display rotation

If your device is mounted with the USB port facing downward, uncomment the rotation line in the `display:` section of the ESPHome YAML:

```yaml
display:
  - platform: st7701s
    # rotation: 270    # uncomment for placement with down-facing USB socket
```

---

## Adjusting brightness and night mode

### Brightness levels

In the ESPHome YAML, the `globals` section contains:

```yaml
- id: normal_brightness
  initial_value: "1.0"    # full brightness (0.0–1.0)

- id: dim_brightness
  initial_value: "0.35"   # night mode / after wake timeout

- id: wake_timeout
  initial_value: "20"     # seconds before dimming after a touch
```

Adjust these to taste and reflash.

### Night mode schedule

Edit the `agile_display_night_mode` binary sensor in `homeassistant/templates.yaml`:

```yaml
state: >
  {% set h = now().hour %}
  {{ h >= 21 or h < 7 }}
```

Change the hours to suit your household. Reload template entities in HA after saving — no reflash required.