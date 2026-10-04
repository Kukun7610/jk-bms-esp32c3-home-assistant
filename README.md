# JK-BMS → Home Assistant with ESP32-C3 Super Mini (ESPHome)

Monitor and control your **JK BMS** (including the JK Inverter series *PB-XX*) in **Home Assistant** over Bluetooth, using a cheap **ESP32-C3 Super Mini** and **ESPHome**.

This project also adds something the BMS does not provide: **two State of Health (SoH) estimates**.

- A **simple** one, calculated from the BMS cycle counter.
- An **advanced** one that also takes **temperature, charge/discharge current and calendar age** into account.

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
10. [State of Health (SoH) explained](#10-state-of-health-soh-explained)
11. [Optional: dashboard cards](#11-optional-dashboard-cards)
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
| **Simple SoH** | **State of Health (Estimated)** and **Cycles Remaining To End Of Life** |
| **Advanced SoH** | **State of Health (Advanced Estimate)**, **Battery Aging Stress Factor**, **Equivalent Full Cycles (Tracked)** and a **Reset Advanced SoH Tracking** button |

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
3. Watch the log for devices whose name starts with **JK**, and copy the MAC address shown next to it. (If nothing appears, try `VERBOSE` instead of `DEBUG`.)
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
5. Find this section and put in **your own key** from step 6.2:

   ```yaml
   api:
     encryption:
       key: "XXX"   # <-- replace XXX with your own encryption key
   ```
6. Check the **protocol version** (section 6.4) and the **SoH settings** (sections 6.5 and 6.6).
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

### 6.5 Set your battery rating (simple SoH)

Also near the top:

```yaml
soh_rated_cycles: "5000"   # cycles the battery is rated for ...
soh_end_of_life: "80.0"    # ... until it reaches this SoH (%)
```

The defaults match the **Highstar IFpP40130220-100 (100 Ah)** cell datasheet (see [section 6.7](#67-values-from-the-highstar-100-ah-datasheet-16s-pack)). If you use a different cell, take these from **your** datasheet. Both SoH sensors use them.

### 6.6 Set the advanced SoH settings

Just below those two lines:

```yaml
soh_pack_capacity_ah: "100"        # total pack capacity in Ah
soh_ref_temp_c: "25.0"             # datasheet test temperature (deg C)
soh_ref_c_rate: "0.5"              # datasheet test C-rate
soh_temp_doubling_c: "10.0"        # wear doubles for every +10 deg C above reference
soh_c_rate_gamma: "0.5"            # extra wear per 1C above the reference C-rate
soh_calendar_loss_per_year: "1.0"  # % lost per year from age alone (0 = off)
```

The most important one is **`soh_pack_capacity_ah`**. Enter the **total pack capacity**. A **16S** pack made of 100 Ah cells in a single string (16S1P) is **100**. If you put two cells in parallel (16S2P) it would be **200**. See [section 10.3](#103-the-advanced-settings-explained) for what each setting means.

---

### 6.7 Values from the Highstar 100 Ah datasheet (16S pack)

This guide was written for a **16S pack of Highstar IFpP40130220-100 (100 Ah LFP prismatic) cells**. The defaults in the code come straight from that datasheet:

| Setting | Value | Where it comes from in the datasheet |
|---|---|---|
| `soh_rated_cycles` | `5000` | Cycle life **≥ 5000 cycles** (sheet 7, item 11) |
| `soh_end_of_life` | `80.0` | End of life is defined as capacity **below 80 %** of initial (same item) |
| `soh_ref_temp_c` | `25.0` | Cycle test is run at **23 ± 5 °C** (sheet 4) |
| `soh_ref_c_rate` | `0.5` | Cycle test uses **150 W per cell**, about 47 A, which is roughly **0.5C**. Rated capacity is also given at 0.5C (sheets 3 and 7) |
| `soh_pack_capacity_ah` | `100` | **Rated capacity 100 Ah**. A 16S1P pack stays at 100 Ah |

> **About "6000 cycles":** the datasheet only **guarantees at least 5000 cycles**. The reference curve on sheet 15 (marked *for reference only*) ends near 80% at roughly 5500 cycles. If a seller quoted 6000, the datasheet does not back that up, so `5000` is the safer value. With 5000 cycles the simple model loses **0.004 % per cycle**.

> **Test conditions matter:** the cycle life is for **100 % depth of discharge** at about 0.5C and room temperature. Hotter, harder or shallower use will give different results, which is exactly why the advanced SoH adjusts for temperature and current.

**What the datasheet does *not* give you:** it does not state how fast the battery ages in storage, and it does not say how much temperature or current shortens cycle life. So `soh_temp_doubling_c`, `soh_c_rate_gamma` and `soh_calendar_loss_per_year` remain **assumptions**.

**16S pack voltages (16 cells in series)**

| Item | Per cell (datasheet) | 16S pack |
|---|---|---|
| Nominal voltage | 3.2 V | **51.2 V** |
| Charge voltage (absolute maximum) | 3.65 V | **58.4 V** |
| Discharge cut-off | 2.5 V | **40.0 V** |
| Recommended long-term storage voltage | 3.2 to 3.4 V | 51.2 to 54.4 V |

In the BMS settings (the **number** entities in Home Assistant) set **cell count = 16** and **total battery capacity = 100**.

**Other datasheet limits worth knowing**

| Item | Datasheet value |
|---|---|
| Standard charge | 0.5C constant current (50 A) to 3.65 V, then constant voltage until the current drops to 0.05C |
| Max continuous charge power | 300 W per cell, roughly 1C (about 90 A) |
| Max discharge | 600 W per cell for 30 s at 100 % SoC, roughly a 2C short pulse (about 190 A). No continuous discharge limit is given in the sheet |
| Charge temperature | **0 to 45 °C**. Below 10 °C, charge at **0.2C (20 A) or less**. Below 0 °C, **do not charge** |
| Discharge temperature | **-20 to 60 °C** |
| Storage temperature | 15 to 35 °C |

These are the cell's limits, not recommended BMS thresholds. Most people set the BMS protections more conservatively than the absolute limits.

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
- **State of Health (Estimated)** and **State of Health (Advanced Estimate)** both show a value close to each other (see section 10)

> **Important:** while the ESP32-C3 is connected, the **JK phone app cannot connect** at the same time. To use the app, turn off the **enable bluetooth connection** switch in Home Assistant (or unplug the ESP32-C3).

---

## 10. State of Health (SoH) explained

JK-BMS does **not** report State of Health, so this project **calculates estimates**. You get two, and it is useful to compare them.

| | Simple SoH | Advanced SoH |
|---|---|---|
| Entity | `State of Health (Estimated)` | `State of Health (Advanced Estimate)` |
| Uses | BMS cycle counter only | Cycle count, **temperature**, **current (C-rate)** and **calendar age** |
| Reacts to hot / harsh use | No | Yes |
| Easy to verify by hand | Yes | Less so |
| Needs a saved value in flash | No | Yes (the running wear total) |

### 10.1 Simple SoH

Assumes a battery rated for **5000 cycles until it reaches 80% SoH** (the Highstar datasheet value):

```
loss per cycle = (100 - 80) / 5000 = 0.004 % per cycle

SoH = 100 - (charging cycles x loss per cycle)
```

| Charging cycles | Estimated SoH |
|---:|---:|
| 0 | 100.00 % |
| 500 | 98.00 % |
| 1000 | 96.00 % |
| 2500 | 90.00 % |
| 5000 | 80.00 % |

A second sensor, **Cycles Remaining To End Of Life**, shows the rated cycles minus the cycles already used.

### 10.2 Advanced SoH

The advanced estimate keeps a **running total of capacity lost**, and adds a little to it every **10 seconds**:

```
SoH = 100 - total loss
```

Each 10-second step adds two kinds of wear.

**Cycling wear** (from the energy moved in or out of the battery):

```
equivalent full cycles = amp-hours moved / (2 x pack capacity)

cycling loss = equivalent full cycles
               x loss per cycle (0.004 %)
               x temperature factor
               x C-rate factor
```

**Calendar wear** (the battery ages even when idle, faster when hot):

```
calendar loss = (calendar loss per year / hours per year)
                x hours elapsed
                x temperature factor
```

**The stress factors:**

| Factor | Rule | Examples |
|---|---|---|
| **Temperature** | Doubles for every +10 °C above 25 °C. Never below 1.0 (no credit for cold). Capped at 4.0. | 25 °C → 1.0, 35 °C → 2.0, 45 °C → 4.0 |
| **C-rate** | 1.0 up to 0.5C. Above that it grows by 0.5 per extra 1C. Capped at 3.0. | 0.5C (50 A) → 1.0, 1C (100 A) → 1.25 |

**At the datasheet conditions (25 °C, 0.5C) both factors are 1.0**, so the advanced model wears the battery at exactly the same rate as the simple model.

**Starting value:** the first time it runs, the advanced SoH starts from the **simple SoH** (based on the BMS cycle count), so a battery that already has 500 cycles starts at 98.00%, not 100%. From then on only **new wear** is added, with temperature and current stress.

**Example (from testing the code, 100 Ah pack, simple cycle count of 500):**

| Situation | Result |
|---|---|
| 1 hour of discharging at 50 A (0.5C), 25 °C | Stress factor **1.00**, wear equal to the datasheet rate |
| 1 hour at 100 A (1C), 40 °C | Stress factor **about 3.5**, so roughly 6.6× more wear for that hour than the first case |
| Sitting idle for 1 year at 25 °C | Loses **1.00 %** from calendar aging (with the default setting) |
| Sitting idle for 1 year at 35 °C | Loses **2.00 %** |

### 10.3 The advanced settings explained

| Setting | Default | What it means |
|---|---|---|
| `soh_pack_capacity_ah` | `100` | **Total pack capacity in Ah.** Cells in parallel add up (16S1P of 100 Ah cells = 100). Wrong value = wrong C-rate and cycle counting. |
| `soh_ref_temp_c` | `25.0` | The temperature your datasheet's cycle-life rating was measured at. |
| `soh_ref_c_rate` | `0.5` | The charge/discharge rate the rating was measured at. Check the datasheet. |
| `soh_temp_doubling_c` | `10.0` | Every this many °C above the reference doubles the wear. Common rule of thumb. |
| `soh_c_rate_gamma` | `0.5` | How quickly wear grows with current above the reference C-rate. |
| `soh_calendar_loss_per_year` | `1.0` | % of capacity lost per year from age alone at the reference temperature. Set `"0"` to turn calendar aging off. |

> **These are assumptions, not lab-fitted constants.** The temperature doubling, C-rate factor and calendar loss are typical rules of thumb for LFP, not values measured for your specific cell. Adjust them if your datasheet or manufacturer gives better numbers.

### 10.4 The extra advanced sensors and the reset button

| Entity | Meaning |
|---|---|
| **Battery Aging Stress Factor** | How fast the battery is wearing right now. **1.0** = datasheet conditions, **2.0** = twice as fast. A useful warning if your pack runs hot. |
| **Equivalent Full Cycles (Tracked)** | Full cycles counted by the ESP32 from energy throughput, since the advanced tracking started. It will not exactly match the BMS "charging cycles", because the BMS counts cycles its own way. |
| **Reset Advanced SoH Tracking** (button) | Restarts the advanced SoH from the BMS cycle count. Use it after **replacing the battery**, or if you changed the settings and want to start over. |

### 10.5 How the value is saved

The running wear total is stored in the ESP32's flash memory so it **survives reboots and power cuts**. To protect the flash from wearing out, it is written only **every 30 minutes**. A sudden power cut can therefore lose **at most the last 30 minutes** of wear. After a full flash erase the saved total is lost, and the sensor simply starts again from the BMS cycle count.

### 10.6 Limitations (please read)

- Both are **estimates**, not measurements. Treat them as a **lifetime indicator**.
- **Depth of discharge is not modelled.** Shallow cycling is usually gentler on LFP, so the advanced estimate may slightly overstate wear for lightly-cycled batteries.
- **Cold charging is not modelled.** The datasheet limits charging below 10 °C to 0.2C and forbids it below 0 °C, but gives no wear rate for it. Use the BMS charge under-temperature protection for this.
- **Calendar time only counts while the ESP32 is powered.** If it is unplugged for a week, that week is not counted.
- The **BMS counts a "cycle" its own way**, so check that the cycle count rises at a sensible rate against your real usage.
- The simple and advanced values should stay **fairly close** in normal use. If the advanced one drifts a long way from the simple one, check the pack capacity setting and the temperature probes first.
- A **more accurate measured SoH** is possible by comparing measured full-charge capacity to rated capacity, but that depends on the BMS capacity learning being well calibrated.

---

## 11. Optional: dashboard cards

In Home Assistant: **Dashboard → Edit → Add card → Manual**, then paste. Entity names can differ slightly, so check yours under **Settings → Devices → jk-bms**.

**Overview card:**

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
  - entity: sensor.jk_bms_state_of_health_advanced_estimate
  - entity: sensor.jk_bms_battery_aging_stress_factor
  - entity: sensor.jk_bms_cycles_remaining_to_end_of_life
  - entity: sensor.jk_bms_delta_cell_voltage
  - entity: binary_sensor.jk_bms_balancing
```

**Gauge for the advanced SoH:**

```yaml
type: gauge
entity: sensor.jk_bms_state_of_health_advanced_estimate
name: Battery Health (Advanced)
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

**History graph comparing both SoH estimates:**

```yaml
type: history-graph
title: SoH comparison
hours_to_show: 720
entities:
  - entity: sensor.jk_bms_state_of_health_estimated
  - entity: sensor.jk_bms_state_of_health_advanced_estimate
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
| **SoH sensors show no value** | They wait until the BMS has reported a cycle count. Give it a minute after the connection is up. |
| **Advanced SoH is far from the simple SoH** | Check `soh_pack_capacity_ah`, and look at the temperature probes and the **Battery Aging Stress Factor** sensor. |
| **Advanced SoH is wrong after changing the settings or the battery** | Press **Reset Advanced SoH Tracking** to re-start it from the BMS cycle count. |
| **Stress factor always 1.0 even when hot** | Check that the BMS temperature sensors 1 and 2 are reporting values. If they are missing, the code assumes the reference temperature. |
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
- This repository adds the **ESP32-C3 Super Mini configuration** and the **simple and advanced SoH estimates**.

---

*If this guide helped you, consider starring the repository.*
