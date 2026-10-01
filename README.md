# JK-BMS → Home Assistant with ESP32-C3 Super Mini (ESPHome)

Monitor and control your **JK BMS** (including the JK Inverter series *JK-PBXA16SXXP*) in **Home Assistant** over Bluetooth, using a **ESP32-C3 Super Mini** and **ESPHome**.

This project also adds something the BMS does not provide: an **Estimated State of Health (SoH)** sensor, calculated from the BMS cycle counter.

> **No ESPHome experience needed.** This guide starts from zero and walks through every step.

---

## Table of contents

1. [What you get](#1-what-you-get)
2. [What you need](#2-what-you-need)
3. [How it works (the big picture)](#3-how-it-works-the-big-picture)
4. [Step 1: Install ESPHome](#4-step-1-install-esphome)
5. [Step 2: Find your BMS Bluetooth MAC address](#5-step-2-find-your-bms-bluetooth-mac-address)
6. [Step 3: Create the device and paste the code](#6-step-3-create-the-device-and-paste-the-code)
7. [Step 4: Flash the ESP32-C3 for the first time](#7-step-4-flash-the-esp32-c3-for-the-first-time)
8. [Step 5: Add the device to Home Assistant](#8-step-5-add-the-device-to-home-assistant)
9. [Step 6: Check that it works](#9-step-6-check-that-it-works)
10. [Estimated State of Health (SoH)](#10-estimated-state-of-health-soh)
11. [Optional: a simple dashboard card](#11-optional-a-simple-dashboard-card)
12. [Troubleshooting](#12-troubleshooting)
13. [Updating later (wireless)](#13-updating-later-wireless)
14. [Credits](#14-credits)

---

## 1. What you get

After setup, Home Assistant will show your battery data live:

| Group | Examples |
|---|---|
| **Pack data** | Total voltage, current, power, state of charge (SoC), remaining capacity |
| **Cells** | Voltage and resistance of every cell (up to 24), min / max / delta / average |
| **Temperatures** | 4 temperature probes + MOSFET temperature |
| **Status** | Charging, discharging, balancing, heating, errors |
| **Counters** | Charging cycles, total runtime, power-on count |
| **Controls** | Charge switch, discharge switch, balancer switch |
| **Settings** | Over/under-voltage limits, current limits, temperature limits and more (adjustable from Home Assistant) |
| **Estimated SoH** | **State of Health (Estimated)** and **Cycles Remaining To End Of Life** |

---

## 2. What you need

### Hardware

- **JK BMS with Bluetooth** (JK Inverter series PB-XX or other JK BMS models)
- **ESP32-C3 Super Mini** board
- **USB-C cable that carries data** (some cables are charge-only and will not work for flashing)
- A **5 V USB power supply** to run the ESP32-C3 permanently (any phone charger works)
- A computer (Windows, Mac or Linux) for the first flash

> There is **no wiring** between the ESP32-C3 and the BMS. They talk over **Bluetooth**. You only need to power the ESP32-C3.

### Software / accounts

- A running **Home Assistant** (Raspberry Pi, mini PC, VM and so on)
- **WiFi** (2.4 GHz) reachable from where the ESP32-C3 will be placed
- **Chrome or Edge** browser (needed for USB flashing from the browser)

---

## 3. How it works (the big picture)

```
 JK BMS  ──Bluetooth──►  ESP32-C3 Super Mini  ──WiFi──►  Home Assistant
(battery)                (runs ESPHome code)             (dashboards, history)
```

- The **ESP32-C3** connects to the BMS over Bluetooth Low Energy (BLE).
- It reads the data and sends it to **Home Assistant** over WiFi.
- The code that tells the ESP32-C3 what to do is the file **`ESP32-C3_jk-bms-config.yaml`** in this repository.
- **ESPHome** is the tool that turns that YAML file into firmware for the ESP32-C3.

**Placement tip:** put the ESP32-C3 within a few metres of the BMS, ideally not inside a closed metal cabinet. Bluetooth is short range.

---

## 4. Step 1: Install ESPHome

Pick **one** of the two options.

### Option A: Home Assistant add-on (easiest, recommended)

Works on **Home Assistant OS** and **Supervised** installs.

1. In Home Assistant go to **Settings → Add-ons → Add-on Store**.
2. Search for **ESPHome Device Builder** and click it.
3. Click **Install**, then **Start**.
4. Turn on **Show in sidebar**.
5. Click **ESPHome** in the sidebar. You should see an empty device list.

> If you have *Home Assistant Container* or *Core* (no add-on store), use Option B.

### Option B: ESPHome Device Builder on your computer

1. Install **Python 3.10 or newer** from [python.org](https://www.python.org/downloads/). On Windows, tick **"Add Python to PATH"** during install.
2. Open a terminal (Command Prompt or PowerShell) and run:
   ```bash
   pip install esphome
   esphome dashboard config
   ```
3. Open **http://localhost:6052** in your browser.

---

## 5. Step 2: Find your BMS Bluetooth MAC address

The ESP32-C3 needs your BMS's **Bluetooth MAC address**. It looks like `C8:47:80:12:34:56`.

### Method 1: nRF Connect app (Android, easiest)

1. Install **nRF Connect for Mobile** (free) from the Play Store.
2. **Close the JK-BMS app** and make sure no other phone is connected to the BMS. The BMS allows only **one** Bluetooth connection at a time.
3. Open nRF Connect and tap **Scan**.
4. Find a device with a name like **`JK_B2A...`** / **`JK-...`** and note the **MAC address** shown under the name.

### Method 2: iPhone

iPhones hide real MAC addresses, so use Method 3 instead.

### Method 3: Use the ESP32-C3 itself

1. Complete Steps 3 and 4 below using a **temporary dummy MAC** such as `00:00:00:00:00:00`.
2. In the ESPHome logs, temporarily change the logger line to `esp32_ble_tracker: DEBUG`.
3. Watch the log for devices whose name starts with **JK**, and copy the MAC address shown next to it.
4. Replace the dummy MAC in `secrets.yaml` with the real one and flash again.

> Write the MAC address down. You will need it in the next step.

---

## 6. Step 3: Create the device and paste the code

### 6.1 Create the secrets (WiFi and MAC)

Secrets keep passwords out of the main code file.

1. In ESPHome, click **Secrets** (top right).
2. Add these three lines, using **your** values:

   ```yaml
   wifi_ssid: "YourWiFiName"
   wifi_password: "YourWiFiPassword"
   bms0_mac_address: "C8:47:80:12:34:56"
   ```
3. Click **Save**.

> Use your **2.4 GHz** WiFi network. The ESP32-C3 does not support 5 GHz.

### 6.2 Create a new device

1. Click **+ New Device** → **Continue**.
2. Give it the name **`jk-bms`**.
3. When asked for a device type, choose **ESP32-C3**.
4. ESPHome will show an **encryption key**. **Copy it and keep it somewhere safe**, then click **Skip** (we flash later).
5. A new device card named **jk-bms** appears.

### 6.3 Paste the code

1. On the **jk-bms** card click **Edit**.
2. **Delete everything** in the editor.
3. Open **`ESP32-C3_jk-bms-config.yaml`** from this repository, click **Raw** → select all → copy.
4. Paste it into the ESPHome editor.
5. Find this section near the top and put in **your own key** from step 6.2:

   ```yaml
   api:
     encryption:
       key: "XXX"   # <-- replace XXX with your own encryption key
   ```
6. Check the **protocol version** matches your BMS (see below).
7. Click **Save**.

### 6.4 Choose the right protocol version

Near the top of the file:

```yaml
protocol_version: JK02_32S
```

| Value | Use it if... |
|---|---|
| `JK02_32S` | **New** JK-BMS, hardware version **11.0 or newer** (also the usual choice for newer PB inverter BMS) |
| `JK02_24S` | Older JK-BMS, hardware version 6.0 to below 11.0 |
| `JK04` | Very old JK-BMS, hardware version 3.0 or lower |

Not sure? Start with `JK02_32S`. If no data appears, try `JK02_24S`.

### 6.5 Set your battery rating for the SoH sensor

Also near the top:

```yaml
soh_rated_cycles: "6000"   # cycles the battery is rated for ...
soh_end_of_life: "80.0"    # ... until it reaches this SoH (%)
```

Change these to match **your battery's datasheet**. See [section 10](#10-estimated-state-of-health-soh).

---

## 7. Step 4: Flash the ESP32-C3 for the first time

The first upload must be done with a **USB cable**. After that, updates are wireless.

### 7.1 Put the board in download mode

The ESP32-C3 Super Mini uses native USB, so you may need to enter boot mode manually:

1. **Hold** the **BOOT** button on the board.
2. While holding it, plug in the **USB-C cable** (or press and release **RESET**).
3. Release **BOOT**.

### 7.2 Flash

**If ESPHome runs on your own computer (Option B):**

1. Click **Install** on the **jk-bms** card.
2. Choose **Plug into this computer**.
3. Select the COM port of the ESP32-C3 and wait until it says *Done*.

**If ESPHome is a Home Assistant add-on (Option A):**

The browser cannot always access USB over a plain `http://` Home Assistant address, so use the manual method:

1. Click **Install** → **Manual download** → choose **Factory format**.
2. Wait for the compile to finish (the first build can take **5 to 15 minutes**) and the `.bin` file downloads.
3. Plug in the ESP32-C3 (in boot mode, as above).
4. Open **https://web.esphome.io** in **Chrome or Edge**.
5. Click **Connect**, select the ESP32-C3 port, then **Install** and choose the `.bin` file you downloaded.
6. Wait until it finishes.

> **No port shows up?** Try a different USB cable (it must carry data), a different USB port, and the boot-mode steps again.

---

## 8. Step 5: Add the device to Home Assistant

1. After flashing, the ESP32-C3 reboots and joins your WiFi.
2. In Home Assistant go to **Settings → Devices & Services**.
3. A **Discovered** card for **ESPHome / jk-bms** should appear. Click **Configure**.
4. Enter the **encryption key** from step 6.2 / 6.3 if asked.
5. Click **Submit**, then choose an **Area** (for example *Garage*).

If it is not discovered automatically: **Add Integration → ESPHome**, then enter the device's IP address or `jk-bms.local`.

---

## 9. Step 6: Check that it works

1. In ESPHome click **Logs** on the **jk-bms** card. You should see lines about the BLE connection and sensor values updating every few seconds.
2. In Home Assistant open **Settings → Devices & Services → ESPHome → jk-bms**. Your sensors should be listed with live values.

Look for these first:

- **online status** is **On**
- **total voltage** matches the display / JK app
- **state of charge** shows a sensible percentage
- **charging cycles** shows a number
- **State of Health (Estimated)** shows a value close to 100%

> **Important:** while the ESP32-C3 is connected, the **JK phone app cannot connect** at the same time. To use the app, turn off the **enable bluetooth connection** switch in Home Assistant (or unplug the ESP32-C3).

---

## 10. Estimated State of Health (SoH)

Your BMS does **not** report State of Health, so this project **calculates an estimate** from the **cycle counter** that the BMS provides.

### The model

The default values assume a battery rated for **6000 cycles until it reaches 80% SoH**:

```
loss per cycle = (100 - 80) / 6000 = 0.00333 % per cycle

SoH = 100 - (charging cycles × loss per cycle)
```

| Charging cycles | Estimated SoH |
|---:|---:|
| 0 | 100.00 % |
| 500 | 98.33 % |
| 1000 | 96.67 % |
| 3000 | 90.00 % |
| 6000 | 80.00 % |

### The two new sensors

| Sensor | Meaning |
|---|---|
| **State of Health (Estimated)** | Estimated battery health in %, clamped between 0 and 100 |
| **Cycles Remaining To End Of Life** | Rated cycles minus cycles already used |

### Changing it for your own battery

Edit the two lines at the top of the YAML file:

```yaml
soh_rated_cycles: "6000"   # e.g. "4000" for a battery rated 4000 cycles
soh_end_of_life: "80.0"    # e.g. "70.0" if the datasheet says 70 % at end of life
```

### Limitations

- It is an **estimate based only on cycle count**. It does not account for temperature, charge rate, or depth of discharge.
- The BMS has its **own way of counting a "cycle"**, so check that the counter rises at a sensible rate against your real usage.
- It is meant as a **lifetime indicator**, not a lab measurement.

---

## 11. Optional: a simple dashboard card

In Home Assistant: **Dashboard → Edit → Add card → Manual**, then paste (adjust the entity names to match yours, which you can find under **Settings → Devices → jk-bms**):

```yaml
type: entities
title: JK BMS
entities:
  - entity: sensor.jk_bms_state_of_charge
  - entity: sensor.jk_bms_total_voltage
  - entity: sensor.jk_bms_current
  - entity: sensor.jk_bms_power
  - entity: sensor.jk_bms_charging_cycles
  - entity: sensor.jk_bms_state_of_health_estimated
  - entity: sensor.jk_bms_cycles_remaining_to_end_of_life
  - entity: sensor.jk_bms_delta_cell_voltage
  - entity: binary_sensor.jk_bms_balancing
```

A gauge for the SoH:

```yaml
type: gauge
entity: sensor.jk_bms_state_of_health_estimated
name: Battery Health (Estimated)
min: 0
max: 100
needle: true
segments:
  - from: 0
    color: "#db4437"
  - from: 70
    color: "#ffa600"
  - from: 80
    color: "#43a047"
```

---

## 12. Troubleshooting

| Problem | What to try |
|---|---|
| **No COM port when flashing** | Use a **data** USB cable. Enter boot mode (hold BOOT, plug in, release). Try another USB port. |
| **Build fails** | Check the YAML indentation (spaces, not tabs). Make sure you copied the **whole** file. |
| **`secret not found` error** | Make sure `wifi_ssid`, `wifi_password` and `bms0_mac_address` are all in **Secrets**. |
| **Device does not connect to WiFi** | Use a **2.4 GHz** network and double-check the SSID and password. |
| **Device not discovered in Home Assistant** | Add it manually: **Add Integration → ESPHome**, then enter the IP address. |
| **Wrong encryption key** | Use the same key in the YAML `api:` section and in Home Assistant. |
| **`online status` is Off / no data** | Check the **MAC address**. Close the JK phone app. Move the ESP32-C3 closer to the BMS. Make sure Bluetooth is enabled on the BMS. |
| **Connected but values are zero / odd** | Try a different `protocol_version` (`JK02_24S` or `JK02_32S`). |
| **Values appear, then connection drops** | Weak Bluetooth signal. Move the board closer or away from metal. Check for a stable 5 V supply. |
| **SoH sensor shows no value** | It waits until the BMS has reported a cycle count. Give it a minute after the connection is up. |
| **Want to use the JK phone app again** | Turn off the **enable bluetooth connection** switch in Home Assistant first. |

Still stuck? Look at the **ESPHome logs**, because they usually say exactly what is wrong. Open an **Issue** in this repository and paste the relevant log lines (remove passwords first).

---

## 13. Updating later (wireless)

After the first USB flash you never need the cable again:

1. Edit the YAML in ESPHome.
2. Click **Install → Wirelessly**.

---

## 14. Credits

- The BMS communication is provided by the excellent **[syssi/esphome-jk-bms](https://github.com/syssi/esphome-jk-bms)** component, which this configuration downloads automatically.
- This repository adds the **ESP32-C3 Super Mini configuration** and the **estimated SoH** calculation.

---

*Hope this guide helped you*
