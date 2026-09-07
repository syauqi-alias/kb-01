# KB-01 Routing Priority

This document is the order in which to route the KB-01 board, and the constraints for each group of nets.

It assumes the layer stackup in [README.md → Layer Stackup](README.md#layer-stackup-6-layer-16-mm-aisler) and the net classes defined in `KB-01_kicad/KB-01/KB-01.kicad_pro` (`SE_50_Outer`, `DP_90_Outer`, `DP_100_Outer`). Board coordinates below are KiCad board coordinates in mm.

**Principle:** route what cannot bend before what can. RF and differential pairs need straight, unbroken paths on an empty board. GPIO can snake around them later.

```
Tier 0  Planes + CM5 escape   foundation — nothing else is valid without these
Tier 1  GNSS RF               1 net, zero tolerance
Tier 2  PCIe                  3 pairs, tightest matching
Tier 3  DSI0                  5 pairs, clock-to-data matching
Tier 4  Switcher hot loops    tiny, local, but fix component placement
Tier 5  USB 2.0               3 pairs, one crosses sides
Tier 6  Power distribution    wide pours + via arrays
Tier 7  Buses                 SPI, I2C, UART, SD, SUSCLK
Tier 8  GPIO / UI / LEDs      anything goes
   —    Leftovers             do not route — delete from schematic
```

## Layer roles (summary)

```
F.Cu     Signal — low-speed       ref In1 GND
In1.Cu   GND plane — never split
In2.Cu   Signal — GPIO / SD / I2C ref In1 GND
In3.Cu   Power — split pours
In4.Cu   GND plane — never split
B.Cu     Signal — high-speed      ref In4 GND
```

**B.Cu is the high-speed side.** The CM5 module, M.2 socket J8, DSI connector J1, Teseo GNSS U6 and SMA J15 are all on B.Cu. PCIe, DSI0 and the GNSS RF trace route bottom-to-bottom with no layer change.

---

## Tier 0 — Planes and CM5 escape

Do this before the first trace.

- Pour **GND on In1.Cu and In4.Cu**, full board, **no splits**. Every impedance number in the net classes assumes a solid plane directly under the trace.
- Plan the **CM5 escape** (204 pads, B.Cu, roughly X 49–105, Y 97–134).
  - High-speed pairs (PCIe, DSI0, USB) exit straight out on B.Cu.
  - GPIO drops to In2 through vias.
  - Reserve clear B.Cu exit lanes for the high-speed pads. Do not place vias in those lanes.

## Tier 1 — GNSS RF

- Net: `Net-(D5-A1)` — add the net label **`GNSS_RF`** in `gnss.kicad_sch` so the name survives re-annotation.
- Path: **U6 RF_IN → D5 → J15 SMA**, all B.Cu, ~12 mm. U6 at (136.1, 50.2), D5 at (132.4, 43.2), J15 at (132.8, 38.6).
- Class: `SE_50_Outer`, **0.395 mm**, referenced to In4.
- Rules:
  - Straight line. No vias. No bends sharper than 45°.
  - No other copper on B.Cu within ~3× trace width (≈1.2 mm) either side.
  - **Via fence** both sides, B.Cu GND pour to In4, ~1.5 mm pitch.
  - Trace passes **through** the D5 ESD pad; no stub branching off.
- Why first: it is the only net where a 1 mm detour measurably costs receiver sensitivity.

## Tier 2 — PCIe

- Nets: `/CM5_high_speed/PCIE_TX_P/N`, `PCIE_RX_P/N`, `PCIE_CLK_P/N`.
- Path: CM5 (B.Cu) → **J8 M.2** at (92.3, 51.4), ~45 mm, B.Cu.
- Class: `DP_90_Outer`, **0.23 mm / 0.12 mm gap**.
- Rules:
  - **Intra-pair skew ≤ 0.1 mm.** No inter-pair matching needed for a single lane.
  - Zero vias if at all possible — both ends are on B.Cu. If a via is unavoidable, one per line, side by side, with a GND stitching via within 1 mm.
  - Pair-to-pair spacing ≥ 3× gap. Keep clear of the DSI pairs.
- Note: the RPi CM5 IO board uses 0.147 mm on a 90 µm dielectric. KB-01 is wider because AISLER's outer prepreg is 230 µm. That is expected — do not "fix" it.
- M.2 lanes 1–3 are intentionally unconnected. Only lane 0 is wired.

## Tier 3 — DSI0

- Nets: `/CM5_high_speed/DPHY0_C_P/N`, `DPHY0_D0..D3_P/N`.
- Path: CM5 (B.Cu) → **J1** at (70.05, 79.4), B.Cu.
- Class: `DP_100_Outer`, **0.18 mm / 0.13 mm gap**.
- Rules:
  - **Intra-pair ≤ 0.1 mm.**
  - **Data-to-clock lane match within ~0.5 mm.**
  - Stay on B.Cu, no vias.
- Why after PCIe: more pairs, slightly more tolerant — DSI bends around PCIe, not the other way round.

## Tier 4 — Switching converter hot loops

Only a few mm each, but they dictate cap and inductor placement. Route them while parts can still be nudged.

- **TPS61022 boost** — U4 at (88.1, 138.3), L2 at (91.5, 144.3), caps C16–C22, all B.Cu.
  - Loop: SW pin → L2 → VOUT cap → GND → PGND pin.
  - Keep on one layer, shortest possible. Cap GND pads drop straight into In4 with 2+ vias each.
- **BQ25895 charger** — U1 at (115.7, 145.2), L1 at (120.0, 136.8), input/output caps, all F.Cu.
  - Same treatment, referenced to In1.
- **Switch nodes** `Net-(U4-SW)` and `Net-(U10-LX)`: short, fat polygons. Never long traces. Never under the RF trace or the U9 oscillator.

## Tier 5 — USB 2.0

- Nets: `/Battery_charger/USB_D+/−`, `MUX_D+/−`, `CHG_D+/−`.
- Path: **J3 USB-C** (F.Cu, 103.6, 153.0) → **U2 TS3USB30E** (F.Cu, 103.3, 143.7) → **U1 BQ25895** (F.Cu) for BC1.2. All F.Cu, short.
- The mux's second channel goes to the **CM5 on B.Cu** — that one pair crosses sides.
  - Two signal vias side by side.
  - One GND stitching via beside them, so return current has a path from In1 to In4.
- Class: `DP_90_Outer`. **Intra-pair ≤ 0.5 mm** — USB 2.0 is forgiving.

## Tier 6 — Power distribution

Biggest current first.

```
BAT+  J4 (127, 150) → U1          F.Cu polygon, ~11 mm, ≥3 mm wide
VBUS  J3 → U1                     F.Cu polygon, ~13 mm, 2–3 mm wide
Vsys  U1 (F.Cu) → U4 (B.Cu)       F.Cu → In3 pour → B.Cu; via array, 6+ vias 0.8/0.4
+5V   U4 → CM5                    B.Cu polygon, ~5 mm — shortest heavy run on the board
Rails +3V3, +1.8v, 3.3V_GNSS,     In3 split pours, fed from CM5 / regulators,
      M.2_3v3                     delivered to F.Cu consumers via 2 vias each
```

- Vsys crossing sides is where a **via array** genuinely matters — a single via is a ~1 A bottleneck.
- Pour In3 last within this tier, after the outer polygons exist, so it is clear which rails still need an inner path.
- Track-width presets for current are in Board Setup → Pre-defined Sizes (0.5 mm ≈ 2 A … 3.0 mm ≈ 7 A, outer layer, 20 °C rise).

## Tier 7 — Buses (mostly In2.Cu)

- **SPI → BMI270** (U12, F.Cu, 103.6, 79.6). Up to 10 MHz — fastest bus on the board. Keep SCK short, no stubs.
- **I2C** → LIS2MDL, BMP581, SHT40, MAX17048, BQ25895. Star or daisy-chain; keep total SDA/SCL length sensible.
- **UART → Teseo** (CM5 B.Cu → U6 B.Cu). Can stay on B.Cu.
- **SD bus** `SD_CLK/CMD/DAT0–3` → J14 (136.5, 90.6). Route as a bundle, match within ~5 mm.
- **SUSCLK**: U9 32.768 kHz oscillator (B.Cu, 82.9, 60.6) → J8. Short. Keep the trace from heading toward L5 (74.4, 51.5).
- **In2 one-axis rule**: mostly vertical runs (along the 120 mm dimension). Jog horizontally on F.Cu or B.Cu. In2 faces In3 with no plane between them.

## Tier 8 — GPIO, UI, sense lines

- Encoder SW3, key switches SW4/SW5, buzzer LS1, power button SW2, LEDs, PPS.
- **Sense lines** — BQ25895 TS / ILIM, MAX17048 `Fuel_G`: short, away from switch nodes. Otherwise unconstrained.

## Do not route

These nets have no destination on this board. They were carried over from the CM5 IO board copy.

```
HDMI0_*   HDMI1_*   USB3-0-*   USB3-1-*   TRD0..3_*   DPHY1_*  (J13)
```

Delete them from `CM5_high_speed.kicad_sch` and `CM5_GPIO.kicad_sch`, and remove J13, **before Tier 0** — otherwise the CM5 escape plan reserves lanes for dead nets, and DRC flags them as unconnected forever.

## Pre-flight checklist

- [ ] Leftover nets and J13 removed from the schematic
- [ ] `GNSS_RF` net label added in `gnss.kicad_sch`
- [ ] GND pours on In1.Cu and In4.Cu, filled, no splits
- [ ] Board Setup → Net Classes → Assignments resolves PCIe → `DP_90_Outer`, DPHY0 → `DP_100_Outer`, RF → `SE_50_Outer`
- [ ] KiCad closed before any external edit to `.kicad_pro` / `.kicad_pcb` / `.kicad_sch`
- [ ] DRC shows no new errors after each tier
