# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is a **KiCad hardware design project**, not software. There is no build, lint, or test
tooling — "correctness" here means schematic/PCB consistency (ERC/DRC), not compiling code.
KB-01 ("Komputer Basikal") is an open-source bike computer built around a Raspberry Pi
Compute Module 5 (CM5): NVMe storage, GNSS, an IMU/mag/baro/humidity sensor cluster for
ML-based dead reckoning, and a BQ25895-based power/charging system. See `README.md` for the
full component list, bus map (I2C1/I2C2/SPI0/UART0), power architecture, and `config.txt`
overlay requirements — read it before making electrical changes, since pin/bus assignments
there must stay in sync with the schematics.

## Repository layout

- `KB-01_kicad/KB-01/` — the active KiCad 10 project (the one to edit).
  - `KB-01.kicad_sch` — top sheet; instantiates the hierarchical sub-sheets below.
  - `battery_charger.kicad_sch`, `5v_boost_conv.kicad_sch`, `gnss.kicad_sch`,
    `sensors.kicad_sch`, `ui.kicad_sch`, `CM5_GPIO.kicad_sch`, `CM5_high_speed.kicad_sch`,
    `pcie-nvme.kicad_sch` — one sheet per subsystem, matching the sheet names in the top sheet.
  - `KB-01.kicad_pcb` — the board layout (single PCB, all sheets combined).
  - `Library/` — local footprint/symbol libraries referenced by `fp-lib-table` /
    `sym-lib-table` via `${KIPRJMOD}`. Keep footprint/symbol names in these tables in sync if
    you add or move library files.
  - `.history/` — **KiCad's automatic local-history snapshots, not a submodule.** It contains
    its own nested `.git` (KiCad created it, not this project), so `git status` on the parent
    repo shows it as a dirty gitlink (`new commits, modified content, untracked content`).
    This is expected and harmless — do not try to "fix" it by adding a `.gitmodules` entry,
    and do not `git add`/commit inside it. It can be deleted if history isn't needed; KiCad
    will recreate it.
  - `~*.lck` files are KiCad session lock files (present while a project file is open).
- `CM5_IO/` — the **reference/upstream CM5 IO board** (KiCad 9 files, from
  `RP-008099-DD-CM5-IO-Board-revision-2`, plus a zip of the same). KB-01's DSI display and
  PCIe/NVMe circuitry were copied from here (per `README.md`). Treat this as read-only
  reference material — port changes into `KB-01_kicad/KB-01/`, don't edit KB-01 by editing
  this copy.
- Layer stackup and per-layer routing intent are documented in `README.md` under
  "Layer Stackup"; the routing order and per-net constraints are in `ROUTING.md`.
  Controlled-impedance nets (PCIe, DSI0, GNSS RF, USB 2.0) belong on F.Cu/B.Cu only;
  In1/In4 are solid GND, In3 is power pours, In2 is low-speed signal.
- `Datasheet/` — component datasheets (BQ25792, BQ25895) for reference during design.
- `KB-01_kicad/KB-01/kb-01 pin assigment.xlsx` and `KB-01_kicad/CM5_Bike_Computer_v3.xlsx` —
  pin-assignment/planning spreadsheets kept alongside the KiCad project; check these when
  changing GPIO/bus assignments so the schematic and spreadsheet don't drift apart.

## Working with the KiCad files

- Edit through KiCad (10.0) itself when possible; these are structured S-expression files
  (`.kicad_sch`, `.kicad_pcb`, `.kicad_pro`) and hand-editing risks corrupting UUIDs,
  hierarchical sheet links, or netlist connectivity that KiCad maintains internally.
- If a text edit to a `.kicad_sch`/`.kicad_pcb` file is unavoidable, keep the S-expression
  formatting/indentation style already used in the file, and don't touch `(uuid ...)` values
  on existing objects.
- After any schematic change, the project should still pass KiCad's ERC (Electrical Rules
  Check) and, after layout changes, DRC (Design Rules Check) — mention this to the user
  rather than assuming they're clean, since these can only be run inside KiCad.
- `*.kicad_prl` (per-user local project settings) is listed in the root `.gitignore` but is
  already tracked in git history from before the ignore rule was added, so it still shows up
  as modified — this is expected, not a bug to fix.
- Generated/derived files that should never be committed: `fp-info-cache`, `*-backups/`,
  `*.lck`, `*.bak`/`*-bak*`, `_autosave-*` — see the root `.gitignore` for the full list.
