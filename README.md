# Komputer Basikal, KB-01

**KB-01** is a high-performance, open-source bike computer built around the Raspberry Pi Compute Module 5. Designed to function like a rugged PDA for cycling, it features NVMe storage, precise GNSS tracking, a full sensor cluster for ML-based dead reckoning, and a robust power management system with 4.5–7 hours of estimated runtime.

## 📋 Specifications

| Feature | Component / Detail |
| :--- | :--- |
| **Core** | Raspberry Pi CM5 (4GB RAM, quad Cortex-A76) |
| **Storage** | 250GB NVMe M.2 2230 (PCIe) |
| **Display** | Waveshare 3.5" LCD (MIPI DSI) |
| **GNSS** | ST Teseo-LIV4F with Taoglas FXP611.07.0092C (Passive Antenna) |
| **IMU** | Bosch BMI270 (Accel/Gyro, SPI) |
| **Magnetometer** | ST LIS2MDL (I2C) |
| **Barometer** | Bosch BMP390 (I2C) |
| **Temp/Humidity** | Sensirion SHT40 (I2C) |
| **Wheel Speed** | TI DRV5023FA (Continuous unipolar Hall switch, 30kHz BW) |
| **Battery** | 10,000mAh Li-ion 1160100 (3.7V / 37Wh) |
| **Charger** | TI BQ25895 (I2C-configurable, USB BC1.2 detection) |
| **Fuel Gauge** | Maxim MAX17048 (I2C) |
| **Boost Converter** | TI TPS61022 (5V output, 8A valley current limit) |
| **USB Mux** | TI TS3USB30E (2:1 USB 2.0, auto-switching via BQ25895 DSEL) |
| **User Input** | Rotary encoder (A/B + push) + 2 buttons |
| **Audio** | Passive electromagnetic buzzer (PWM-driven via 2N7002 N-FET) |
| **Connectivity** | USB 2.0 (device/peripheral mode for GPX file transfer + charging) |

## ⚡ Power System

The power architecture handles high current loads from the CM5 and NVMe while ensuring safe battery management.

```
USB-C → BQ25895 (VBUS) → SYS → TPS61022 (VIN) → 5V → CM5
                ↕
            Battery
```

* **Charger / Power Path:** [TI BQ25895](https://www.ti.com/product/BQ25895)
    * Charges the battery while simultaneously powering the system via SYS rail.
    * Integrated USB BC1.2 detection for automatic source identification (SDP/CDP/DCP).
    * Input Current Optimizer (ICO) negotiates maximum power from USB source.
    * Ship mode via BATFET for zero-quiescent shutdown.
* **Fuel Gauge:** [Maxim MAX17048](https://www.analog.com/en/products/max17048.html)
    * ModelGauge algorithm — no sense resistor required.
    * Provides accurate SoC% and voltage reporting over I2C.
* **Boost Converter:** [TI TPS61022](https://www.ti.com/product/TPS61022)
    * Provides a fixed 5V output from 3.0–4.5V battery input.
    * EN tied to VIN (always on when SYS is live); MODE tied to GND (PFM for light-load efficiency).
    * Feedback resistors: R1 = 732kΩ, R2 = 100kΩ (per TI reference design).
* **USB-C:**
    * CC1/CC2: 5.1kΩ pull-downs to GND (sink device).
    * D+/D−: Routed via TS3USB30E multiplexer — Channel 1 to BQ25895 (BC1.2 detection), Channel 2 to CM5 USB (data transfer).
    * Mux control: BQ25895 DSEL pin drives TS3USB30E SEL (automatic, no GPIO needed).
    * CM5 USB configured as device/peripheral via `dtoverlay=dwc2,dr_mode=peripheral`.
* **Battery Life:** ~4.5 hours (max load) to ~7 hours (typical load).

## 🔌 Bus Architecture

| Bus | GPIOs | Devices | Notes |
| :--- | :--- | :--- | :--- |
| **I2C1** | GPIO2/3 | BQ25895, MAX17048 | 1.8kΩ built-in pull-ups (do NOT add external) |
| **I2C2** | GPIO4/5 | LIS2MDL, BMP390, SHT40 | 4.7kΩ external pull-ups required |
| **SPI0** | GPIO8–11 | BMI270 | Mode 0, max 10MHz |
| **UART0** | GPIO14/15 | Teseo-LIV4F | 9600 baud default, TX/RX crossover |
| **DSI0** | DISP0 connector | Waveshare 3.5" | Copied from CM5 IO board |
| **PCIe** | PCIe connector | NVMe SSD | Copied from CM5 IO board |

## 📡 Sensor Cluster & ML Dead Reckoning

The sensor cluster enables ML-based location prediction when GNSS is unavailable (tunnels, dense tree cover, urban canyons).

* **Inputs:** BMI270 (acceleration/rotation), LIS2MDL (heading), BMP390 (altitude), DRV5023FA (wheel speed/distance)
* **Ground truth:** Teseo-LIV4F GNSS position during training
* **Approach:** Extended Kalman filter or RNN; wheel speed removes velocity drift ambiguity
* **Wheel Speed Sensor:** DRV5023FA is a continuous-sensing unipolar Hall switch (not sampled) with 30kHz bandwidth — it never misses a pulse regardless of wheel speed. Single south-facing magnet on spoke, open-drain output with 10kΩ pull-up.

## 🔊 Audio Alerts

A passive electromagnetic buzzer (e.g. MLT-8530) driven by PWM on GPIO7 through a 2N7002 N-channel MOSFET. Used for turn-by-turn navigation alerts, speed warnings, button feedback, and low battery warnings.

```
GPIO7 → 100Ω → gate ──┬── 2N7002 drain ──── Buzzer (−)
                       │                          │
                     10kΩ                     Buzzer (+) → 3.3V
                       │                          │
                      GND              1N4148 ──|◄──┘ (flyback diode)
```

## 🖥️ Software Architecture

* **Language:** C (with pthreads for multi-core)
* **GUI:** LVGL (renders directly to framebuffer, no X11/Wayland)
* **Kernel:** PREEMPT_RT patched Linux for deterministic interrupt handling
* **Core allocation:**
    * Core 0 — UI rendering + wheel speed ISR (highest priority)
    * Core 1 — Sensor acquisition (I2C/SPI reads)
    * Core 2 — GNSS processing (UART/NMEA parsing)
    * Core 3 — ML inference + data logging

## 🛠️ Hardware Design Notes

* **Space Optimization:** The boost converter is placed underneath the M.2 NVMe drive clearance to minimize the PCB footprint.
* **Antenna:** Uses a passive flexible polymer antenna (Taoglas FXP611) for GNSS.
* **Mounting:** Designed to accommodate standard 3.5" display mounting points.
* **GPIO0:** Left as spare — HAT EEPROM pin (I2C0/ID_SD), available for future expansion.

### Layer Stackup (6-layer, 1.6 mm, AISLER)

Stackup source: [AISLER 6-layer 1.6 mm 35 µm](https://community.aisler.net/t/6-layers-1-6mm-35-m-stackup/5458). Copper is 35 µm on all layers. Impedance-controlled nets stay on the outer layers only, referenced to the adjacent solid GND plane (same approach as the Raspberry Pi CM5 IO board).

```
Layer    Role                     Reference   Dielectric below
F.Cu     Signal — low-speed       In1 GND     2 × 2116 prepreg, 230 µm, Er 4.25
In1.Cu   GND plane — never split  —           FR4 core,          300 µm, Er 4.5
In2.Cu   Signal — GPIO / SD / I2C In1 GND     3 × 2116 prepreg, 345 µm, Er 4.25
In3.Cu   Power — split pours      —           FR4 core,          300 µm, Er 4.5
In4.Cu   GND plane — never split  —           2 × 2116 prepreg, 230 µm, Er 4.25
B.Cu     Signal — high-speed      In4 GND     —
```

* **B.Cu (high-speed side):** CM5 module, M.2 NVMe, DSI connector, Teseo GNSS and SMA are all on B.Cu, so PCIe, DSI0 and the GNSS RF trace route bottom-to-bottom with no layer changes. Net classes: `DP_90_Outer` (PCIe, USB 2.0), `DP_100_Outer` (DSI0), `SE_50_Outer` (GNSS RF).
* **F.Cu:** USB-C, charger, fuel gauge, sensors, UI. USB 2.0 D+/D− (J3 → TS3USB30E → BQ25895) stays on F.Cu.
* **In2.Cu:** route predominantly one axis (along the 120 mm dimension) — it faces In3 with no plane between them.
* **In3.Cu:** pours only, no traces: `+5V`, `Vsys`, `+3V3`, `VBUS`, `+1.8v`, `3.3V_GNSS`, `M.2_3v3`.
* **Stitching:** via fence between In1 and In4 along the RF trace and around the CM5 high-speed region. Fill unused In2 area with GND to balance copper against In3.

See [ROUTING.md](ROUTING.md) for the routing order and per-net constraints.

## ⚙️ config.txt

```ini
dtparam=i2c_arm=on
dtoverlay=i2c2-pi5,pins_4_5
dtparam=spi=on
enable_uart=1
dtoverlay=dwc2,dr_mode=peripheral
```

## ⚠️ Disclaimer

This project is currently in the **prototype/brainstorming phase**.

* Battery life estimates (4.5–7 hours) are theoretical and based on datasheet typical values.
* USB-PD is not supported with passive CC resistors (5V input only).
* The CM5 + NVMe combination draws significant power — consider NVMe power states for optimisation.

## 🔗 References & Credits

* Designed by [Syauqi Alias](https://syauqi-alias.github.io/)
* Read the full project log here: [Projects Brainstorming](https://syauqi-alias.github.io/jekyll/2023-06-10-idea.html#komputer-basikal-kb-01-bike-computer)

## 📄 License

[MIT License](LICENSE)
