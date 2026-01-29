# Komputer Basikal, KB-01

**KB-01** is a high-performance, open-source bike computer built around the Raspberry Pi Compute Module. Designed to function like a rugged PDA for cycling, it features NVMe storage, precise GNSS tracking, and a robust power management system.

## 📋 Specifications

| Feature | Component / Detail |
| :--- | :--- |
| **Core** | Raspberry Pi CM5 (4GB RAM) |
| **Storage** | 250GB NVMe M.2 2230 |
| **Display** | 3.5” LCD |
| **GNSS** | ST Teseo-LIV3F with Taoglas FXP611.07.0092C (Passive Antenna) |
| **Sensors** | LSM6DS3 (Accel/Gyro), MMC5983MA (Magnetometer) |
| **Battery** | 6000mAh Li-ion (906090) |
| **Connectivity** | USB 2.0 Multiplexer (Data transfer + Charging) |

## ⚡ Power System

The power architecture is designed to handle high current loads from the Compute Module and NVMe drive while ensuring safe battery management.

* **BMS (Battery Management System):** [TI BQ25895](https://www.ti.com/product/BQ25895)
    * Supports charger pass-through (charges battery while powering the SBC).
    * Integrated USB detection and power path management.
* **PSU Boost Converter:** [TI TPS61022](https://www.ti.com/product/TPS61022)
    * Provides a fixed 5.1V output with up to 8A capacity.
    * Ensures sufficient headroom for the CM5 peak power draw.

## 🛠️ Hardware Design Notes

* **Space Optimization:** The step-down converter is placed underneath the M.2 NVMe drive clearance to minimize the PCB footprint.
* **Antenna:** Uses a passive flexible polymer antenna (Taoglas FXP611) for GNSS.
* **Mounting:** Designed to accommodate standard 3.5" display mounting points.

## ⚠️ Disclaimer

This project is currently in the **prototype/brainstorming phase**. 
* Battery life is estimated to be short due to the high power consumption of the CM5 and NVMe.

## 🔗 References & Credits

* Designed by [Syauqi Alias](https://syauqi-alias.github.io/)
* Read the full project log here: [Projects Brainstorming](https://syauqi-alias.github.io/jekyll/2023-06-10-idea.html#komputer-basikal-kb-01-bike-computer)

## 📄 License

[MIT License](LICENSE)