# JK-BMS → Home Assistant with ESP32-C3 Super Mini (ESPHome)

Monitor and control your **JK BMS** (including the JK Inverter series *PB-XX*) in **Home Assistant** over Bluetooth, using a cheap **ESP32-C3 Super Mini** and **ESPHome**.

This project adds something the BMS does not provide: **three State of Health (SoH) values**.

- **Simple SoH:** calculated from the BMS cycle counter.
- **Advanced SoH:** also takes temperature, current, cold charging, state of charge and calendar age into account, and is **corrected by a real capacity measurement**.
- **Measured SoH:** the last capacity you really measured, divided by the rated capacity.

> 📘 **Want to set it up?** Follow the **[Installation Guide](INSTALLATION.md)**.
> This page explains **how the SoH is calculated**.

**Files in this repository**

| File | Purpose |
|---|---|
| `ESP32-C3_jk-bms-config.yaml` | The ESPHome configuration (paste this into ESPHome) |
| `IMG.png` | The Home assistant Dashboard Entity reference |
| [`INSTALLATION.md`](INSTALLATION.md) | Beginner step-by-step setup guide |
| `README.md` | This page: how the SoH calculation works |

---

## Table of contents

1. [State of Health (SoH) explained](#1-state-of-health-soh-explained)
2. [Optional: dashboard cards](#2-optional-dashboard-cards)
3. [Troubleshooting (SoH)](#3-troubleshooting-soh)
4. [Credits](#4-credits)

---

## 1. State of Health (SoH) explained

Your BMS does **not** report State of Health, so this project **calculates estimates**. You get two, and it is useful to compare them.

| | Simple SoH | Advanced SoH |
|---|---|---|
| Entity | `State of Health (Estimated)` | `State of Health (Advanced Estimate)` |
| Uses | BMS cycle counter only | Cycle count, **temperature**, **current (C-rate)**, **cold charging**, **state of charge** and **calendar age** |
| Reacts to hot / harsh use | No | Yes |
| Reacts to cold charging and long time at full charge | No | Yes |
| Corrected by a real capacity measurement | No | **Yes** (closed loop, section 1.3) |
| Easy to verify by hand | Yes | Less so |
| Needs a saved value in flash | No | Yes (the running wear total) |

You also get **`State of Health (Measured)`**, which is simply the last capacity you really measured divided by the rated capacity. It is the only one of the three that is a measurement rather than an estimate.

### 1.1 Simple SoH

Assumes a battery rated for **5000 cycles until it reaches 80% SoH** (the Highstar datasheet value, see [INSTALLATION.md, section 6.7](INSTALLATION.md#67-values-from-the-highstar-100-ah-datasheet-16s-pack)):

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

### 1.2 Advanced SoH

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
               x heat factor
               x C-rate factor
               x cold-charge factor   (only while charging in the cold)
```

**Calendar wear** (the battery ages even when idle):

```
calendar loss = (calendar loss per year / hours per year)
                x hours elapsed
                x heat factor
                x state-of-charge factor
```

**The stress factors** (all are 1.0 under normal, datasheet-like conditions):

| Factor | Applies to | Rule | Examples |
|---|---|---|---|
| **Heat** | Cycling and calendar | Doubles for every +10 °C above 25 °C. Never below 1.0. Capped at 4.0. | 25 °C → 1.0, 35 °C → 2.0, 45 °C → 4.0 |
| **C-rate** | Cycling | 1.0 up to 0.5C. Above that it grows by 0.5 per extra 1C. Capped at 3.0. | 0.5C (50 A) → 1.0, 1C (100 A) → 1.25 |
| **Cold charge** | Cycling, **only while charging** | 1.0 at 10 °C and above. Below that it grows exponentially: `exp(0.15 x degrees below 10 °C)`. Uses the **coldest** temperature probe. Capped at 20. | 10 °C → 1.0, 5 °C → 2.1, 0 °C → 4.5, -10 °C → 20 |
| **State of charge** | Calendar only | 1.0 up to 70 % SoC, then rises in a straight line to 1.5 at 100 % SoC. | 50 % → 1.0, 85 % → 1.25, 100 % → 1.5 |

**Starting value:** the first time it runs, the advanced SoH starts from the **simple SoH** (based on the BMS cycle count), so a battery that already has 500 cycles starts at 98.00%, not 100%. From then on only **new wear** is added.

**Examples (from testing the code, 100 Ah pack):**

| Situation | Result |
|---|---|
| 1 hour of discharging at 50 A (0.5C), 25 °C | Stress factor **1.00**, wear equal to the datasheet rate |
| 1 hour at 100 A (1C), 40 °C | Stress factor **about 3.5**, so roughly 6.6× more wear for that hour |
| 1 hour **charging** at 20 A, 5 °C instead of 25 °C | Cycling wear **2.1×** higher |
| The same at 0 °C | Cycling wear **4.5×** higher |
| **Discharging** in the cold | **No** extra penalty (the datasheet limits charging, not discharging, at low temperature) |
| Sitting idle for 1 year at 25 °C, SoC 20 to 70 % | Loses **1.00 %** |
| Sitting idle for 1 year at 25 °C, SoC 85 % | Loses **1.25 %** |
| Sitting idle for 1 year at 25 °C, SoC 100 % | Loses **1.50 %** |

> **The cold-charge penalty is a safety net, not protection.** It only *records* the damage. Your datasheet says charging below 0 °C is not allowed, so also set the BMS **charge under-temperature protection**.

### 1.3 Closed-loop recalibration (fixing the drift)

A running total that only adds small amounts will slowly **drift** away from the truth, because any error in the assumed numbers keeps accumulating. To stop that, the project **measures the real capacity** and corrects the total. There are two ways, and you can use both.

#### Automatic capacity test

The ESP32 watches for a clean "full to empty" discharge and counts the amp-hours that come out. It uses **cell voltages**:

1. **Armed:** the highest cell reaches **3.42 V or more**. The status shows *"Armed: pack is full, waiting for a discharge"* and **stays that way** while the highest cell is above 3.40 V. Small charge or discharge currents while the pack is full (for example an inverter switching back and forth) do **not** start or stop anything.
2. **Counting:** only when the highest cell **falls to 3.40 V** does the count begin. Every amp-hour that leaves the pack is added up. Charging is allowed to a small degree: up to **0.2 Ah** of charge is simply subtracted from the total and does **not** abort the test.
3. **Aborted:** only if **more than 0.2 Ah is charged** during the count, or the BMS goes offline. The test is thrown away, and it re-arms the next time the highest cell reaches 3.42 V.
4. **Finished:** the lowest cell reaches **3.00 V**. The amp-hours delivered are the measured capacity.
5. **Checked:** the result is only accepted if the test was clean:

| Check | Default |
|---|---|
| Average temperature during the test | 15 to 40 °C |
| Average discharge rate | 0.7C or less |
| Amp-hours delivered | At least 50 % of rated, and not more than 115 % |

6. **Applied:** the running wear total is moved towards the measurement. With the default **blend of 0.5** it moves **halfway**, which limits the effect of measurement noise. Set the blend to **1.0** to overwrite it completely, or **0.0** to only watch the measured value without changing the SoH.

**Example (simulated 90 Ah pack, rated 100 Ah, advanced SoH starting at 98.00 %):** a full test from 3.40 V down to 3.00 V measured **90.0 Ah**, and the status showed *"Accepted: 90.0 Ah measured, advanced SoH corrected to 94.0 %"*.

Why 94.0 % and not 90 %? Because of the **blend of 0.5**: the SoH moves only halfway from 98.0 % towards the measured 90 %. Each further clean test moves it halfway again, so it settles close to the measurement. With `soh_cal_blend: "1.0"` it would jump straight to 90.0 %. **State of Health (Measured)** always shows the plain measurement (90.0 % here).

**Capacity Test Status** shows what is happening:

| Message | Meaning |
|---|---|
| Waiting for a full charge | Nothing is happening yet |
| Armed: pack is full, waiting for a discharge | The highest cell has been at 3.42 V or more and has not yet fallen to 3.40 V |
| Counting discharge: 37.2 Ah delivered so far | A test is running |
| Accepted: 90.0 Ah measured, advanced SoH corrected to 94.0 % | The measurement was used |
| Rejected: ... | The test was not clean; the reason is in the message and nothing was changed |
| Aborted: ... | More than 0.2 Ah was charged during the count, or the BMS went offline, so the test was thrown away |

**Be realistic about the automatic test.** Many solar and inverter systems **never** discharge to empty. If yours doesn't, it will never finish a test, and the manual method below is the way to calibrate.

**Important: the 3.00 V end point reads low.** The datasheet capacity is measured down to **2.5 V**. Stopping at 3.00 V leaves a small amount of capacity unmeasured, so the automatic test will read **lower than the true capacity**. Until you correct for that, each accepted test pulls the SoH down slightly too far. Two ways to handle it:

- Run **one careful manual test** down to your BMS cut-off, note the Ah, and compare it with what the automatic test reports. Then set `soh_cal_capacity_correction` to *(Ah to the BMS cut-off) ÷ (Ah the automatic test reported)*. I can't tell you the size of the gap for your cell, because it depends on the cell, temperature and discharge rate.
- Or set `soh_cal_blend` to `0.0` for a while. The **State of Health (Measured)** and **Last Measured Capacity** sensors still update, but the advanced SoH is left alone until you trust the numbers.

#### Manual calibration

If you can measure the capacity yourself (or after one carefully run test):

1. Charge the pack fully, until the BMS shows 100 %. Let it rest for 30 minutes.
2. Discharge it at about **0.5C** (about 50 A) at **15 to 35 °C**, with **no charging** in between, until the lowest cell reaches your cut-off.
3. Note the **amp-hours that came out** (from a shunt-based meter or your inverter or BMS counter).
4. In Home Assistant, enter that number in **Measured Capacity Input**.
5. Press **Apply Measured Capacity**.

The advanced SoH is **set directly** to measured ÷ rated (capped at 100 %, since new cells are often a little above their rating).

> Capacity depends on temperature and discharge rate, and the datasheet rating is for about 0.5C at 23 ± 5 °C. That is why the checks above exist. Don't calibrate from a test done in the cold or at high current.

### 1.4 The extra entities and buttons

| Entity | Meaning |
|---|---|
| **Battery Aging Stress Factor** | How fast the battery is wearing right now from heat, current and cold charging. **1.0** = datasheet conditions, **2.0** = twice as fast. |
| **Equivalent Full Cycles (Tracked)** | Full cycles counted by the ESP32 from energy throughput. It will not exactly match the BMS "charging cycles". |
| **State of Health (Measured)** | Last measured capacity ÷ rated capacity. The only real measurement of the three. |
| **Last Measured Capacity** | The last capacity (Ah) that was measured, automatically or entered by hand. |
| **Capacity Test Status** | What the automatic test is doing (see the table above). |
| **Measured Capacity Input** | A number box where you type a capacity you measured yourself. |
| **Apply Measured Capacity** (button) | Applies the typed number to the advanced SoH. |
| **Reset Advanced SoH Tracking** (button) | Restarts the advanced SoH from the BMS cycle count. Use it after **replacing the battery**. |

### 1.5 All advanced settings

| Setting | Default | What it means |
|---|---|---|
| `soh_pack_capacity_ah` | `100` | **Total pack capacity in Ah** (16S1P of 100 Ah cells = 100). Wrong value = wrong C-rate, cycle count and calibration. |
| `soh_ref_temp_c` | `25.0` | Temperature the datasheet cycle life was measured at (23 ± 5 °C). |
| `soh_ref_c_rate` | `0.5` | Rate the datasheet cycle life was measured at (150 W per cell, about 0.5C). |
| `soh_temp_doubling_c` | `10.0` | Every this many °C above the reference doubles the wear. |
| `soh_c_rate_gamma` | `0.5` | How quickly wear grows with current above the reference C-rate. |
| `soh_calendar_loss_per_year` | `1.0` | % lost per year from age alone, at the reference temperature and **mid SoC** (30 to 70 %). `"0"` turns calendar aging off. |
| `soh_cold_charge_below_c` | `10.0` | Below this temperature, **charging** adds extra wear. (Datasheet: 0.2C maximum below 10 °C.) |
| `soh_cold_charge_k` | `0.15` | Steepness of the cold-charge penalty: `exp(k x degrees below the limit)`. |
| `soh_cold_charge_max_factor` | `20.0` | Upper limit for the cold-charge multiplier. |
| `soh_soc_high_start` | `70.0` | SoC (%) up to which calendar wear is normal. |
| `soh_soc_high_factor` | `1.5` | Calendar wear multiplier at 100 % SoC. |
| `soh_cal_full_cell_v` | `3.42` | The highest cell at or above this voltage means the pack is full and the test is **armed**. Lower it if your charger stops below this and the test never arms. |
| `soh_cal_start_cell_v` | `3.40` | While armed, counting starts only when the highest cell falls to this voltage. |
| `soh_cal_max_charge_ah` | `0.2` | The test is aborted only if more than this many Ah are charged during the count. |
| `soh_cal_empty_cell_v` | `3.00` | The test ends when the lowest cell reaches this voltage. The datasheet cut-off is 2.5 V, so a higher value reads a little low (see section 1.3). |
| `soh_cal_capacity_correction` | `1.00` | Multiplies the measured Ah. Use above 1.00 to compensate for stopping at 3.00 V instead of 2.5 V. |
| `soh_cal_blend` | `0.5` | How far each accepted test moves the SoH: 1.0 = overwrite, 0.5 = halfway. |
| `soh_cal_min_temp_c` / `soh_cal_max_temp_c` | `15.0` / `40.0` | Tests outside this average temperature are rejected. |
| `soh_cal_max_c_rate` | `0.7` | Tests discharged faster than this are rejected. |
| `soh_dir_threshold_w` | `5.0` | Power (W) above which the pack counts as charging or discharging. |

> **The wear multipliers (heat doubling, C-rate, cold-charge, state-of-charge and the calendar rate) are assumptions, not lab-fitted constants.** Your datasheet does not give them. This is exactly why the capacity calibration exists: the measured capacity keeps the result honest even if these assumptions are a little off.

### 1.6 How the value is saved

The running wear total and the last measured capacity are stored in the ESP32's flash memory so they **survive reboots and power cuts**. To protect the flash from wearing out, they are written only **every 30 minutes**. A sudden power cut can lose **at most the last 30 minutes** of changes. An automatic test that is running during a reboot is simply thrown away. After a full flash erase the saved values are lost, and the sensor starts again from the BMS cycle count.

### 1.7 Limitations (please read)

- The simple and advanced values are **estimates**. Only **State of Health (Measured)** is a measurement, and only when you have run a clean test.
- **Depth of discharge is not modelled.** The wear is counted by amp-hours moved, so 100 small 1 % cycles count the same as one 100 % cycle. Real deep cycling is probably a bit harder on the cells, but the datasheet rating is for **100 % depth of discharge**, so for deep cycles this is the right match. Detecting each charge/discharge swing reliably needs an accurate BMS state of charge and would add a lot of complexity, so it was left out.
- **Calendar time only counts while the ESP32 is powered.** If it is unplugged for a week, that week is not counted.
- The 5000-cycle datasheet rating may already include some ageing from time, so adding calendar wear on top may count a little twice. This makes the estimate slightly pessimistic.
- The **BMS counts a "cycle" its own way**. Check that its cycle count rises at a sensible rate against your real use.
- The **automatic test needs a discharge from about 3.40 V down to 3.00 V** on the lowest cell with almost no charging in between, which may be rare. Its result also reads slightly low because it stops at 3.00 V.
- The simple and advanced values should stay **fairly close** in normal use. If they drift far apart, check the pack capacity setting and the temperature probes first.

---

---

## 2. Optional: dashboard cards

In Home Assistant: **Dashboard → Edit → Add card → Manual**, then paste. Entity names can differ slightly, so check yours under **Settings → Devices → jk-bms**.

![HA overview](https://raw.githubusercontent.com/Kukun7610/jk-bms-esp32c3-home-assistant/refs/heads/main/IMG.png)

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
  - entity: sensor.jk_bms_state_of_health_measured
  - entity: sensor.jk_bms_last_measured_capacity
  - entity: sensor.jk_bms_capacity_test_status
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

---

## 3. Troubleshooting (SoH)

| Problem | What to try |
|---|---|
| **SoH sensors show no value** | They wait until the BMS has reported a cycle count. Give it a minute after the connection is up. |
| **Advanced SoH is far from the simple SoH** | Check `soh_pack_capacity_ah`, and look at the temperature probes and the **Battery Aging Stress Factor** sensor. |
| **Advanced SoH is wrong after changing the settings or the battery** | Press **Reset Advanced SoH Tracking** to re-start it from the BMS cycle count. |
| **Capacity Test Status stays on "Waiting for a full charge"** | The highest cell must reach **3.42 V**. If your charger stops lower, reduce `soh_cal_full_cell_v` (and keep `soh_cal_start_cell_v` below it). |
| **Capacity Test Status stays on "Armed"** | That is normal until the highest cell falls to 3.40 V. The count starts then. |
| **Capacity Test Status says "Rejected"** | The test was not clean (temperature, discharge too fast, or it did not start from full). Nothing was changed; the message says why. |
| **Measured capacity looks too low** | Stopping at 3.00 V reads a little low. Raise `soh_cal_capacity_correction`, or lower `soh_cal_empty_cell_v`. |
| **Stress factor always 1.0 even when hot** | Check that the BMS temperature sensors 1 and 2 are reporting values. If they are missing, the code assumes the reference temperature. |

Problems with **installation, flashing, WiFi or the Bluetooth connection** are covered in **[INSTALLATION.md](INSTALLATION.md#10-troubleshooting-installation-and-connection)**.

---

## 4. Credits

- The BMS communication is provided by the excellent **[syssi/esphome-jk-bms](https://github.com/syssi/esphome-jk-bms)** component, which this configuration downloads automatically.
- This repository adds the **ESP32-C3 Super Mini configuration** and the **simple and advanced SoH estimates**.

---

*If this guide helped you, consider starring the repository.*
