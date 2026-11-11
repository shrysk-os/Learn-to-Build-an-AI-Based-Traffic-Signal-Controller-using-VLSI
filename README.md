# AI-Based Traffic Signal Controller Using VLSI

A parameterized SystemVerilog implementation of **two coordinated 4-way traffic junctions** featuring adaptive green timing, pedestrian arbitration, emergency override, and precise green-wave synchronization.

## Features

- **Parameterized Junction Controller:** Reusable Moore FSM supporting NS Green, NS Yellow, All-Red, EW Green, EW Yellow, Pedestrian, and parked `NS_WAIT` states. Two instances share the same RTL with different parameters.
- **Generic Timer:** Standalone `generic_timer` with `COUNTER_WIDTH`, `start`, `count_target`, and `done` interfaces. Reused for green-wave offset generation.
- **Pedestrian Arbitration:** Round-robin `ped_arbiter` with held grants and starvation-free request/grant handling.
- **Emergency Override:** Both junctions reach All-Red within one clock cycle of emergency sampling (requirement: ≤ 2 cycles), followed by restart through a valid safe phase.
- **Green-Wave Synchronization:** Junction B's NS Green starts exactly 4 clock cycles after Junction A's NS Green, independent of green and pedestrian durations.
- **Adaptive Green Timing:**  
  `green = clamp(GREEN_TIME + density × GREEN_EXT_STEP, MIN_GREEN_TIME, MAX_GREEN_TIME)`  
  Traffic density is sampled once at green-phase entry.
- **RTL Design Practices:** Single clock domain, active-high synchronous reset, named port connections, and no magic numbers in FSM logic.

## Repository Structure

```text
rtl/
├── generic_timer.sv
├── junction_controller.sv
├── ped_arbiter.sv
└── amtgc_top.sv

tb/
├── tb_monitors.sv
├── tb_generic_timer.sv
├── tb_junction_controller.sv
├── tb_amtgc_top.sv
└── tb_amtgc_full.sv

sim/
├── run_all.bat
├── wave_*.do
├── logs/
└── screenshots/

docs/
├── Technical_Report.md
├── Task1-4_AsBuilt_Documentation.md
├── diagrams/
├── ASSUMPTIONS.md
├── BUG_LOG.md
├── Task5_Coverage_Statement.md
├── SCREENSHOT_MANIFEST.md
├── Self_Attestation_Checklist.md
├── Demo_Video_Script.md
└── RTL_CHECKSUMS.txt
```

## Tools & Requirements

- ModelSim Intel FPGA Edition 2020.1
- QuestaSim 2021.1 (also tested)
- Windows environment for the provided batch script

No additional dependencies are required.

## How to Run

Run commands from the project root directory (the folder containing `rtl/`, `tb/`, and `sim/`).

### Option 1: Run All Tests (Windows)

Ensure ModelSim is available on your system `PATH`.

```bat
sim\run_all.bat
```

### Option 2: Run Individual Testbenches

Execute the following commands in the ModelSim Transcript.

#### Initialize Simulation

```tcl
vlib work
vmap work work
```

#### Task 2 – Generic Timer

```tcl
vlog -sv rtl/generic_timer.sv tb/tb_generic_timer.sv
vsim -voptargs=+acc work.tb_generic_timer
do sim/wave_timer.do
run -all
quit -sim
```

#### Task 2/4 – Junction Controller

```tcl
vlog -sv rtl/generic_timer.sv rtl/junction_controller.sv tb/tb_monitors.sv tb/tb_junction_controller.sv
vsim -voptargs=+acc work.tb_junction_controller
do sim/wave_junction.do
run -all
quit -sim
```

#### Task 3/4 – System Integration

```tcl
vlog -sv rtl/generic_timer.sv rtl/junction_controller.sv rtl/ped_arbiter.sv rtl/amtgc_top.sv tb/tb_monitors.sv tb/tb_amtgc_top.sv
vsim -voptargs=+acc work.tb_amtgc_top
do sim/wave_system.do
run -all
quit -sim
```

#### Task 5 – Unified Self-Checking Verification

```tcl
vlog -sv rtl/generic_timer.sv rtl/junction_controller.sv rtl/ped_arbiter.sv rtl/amtgc_top.sv tb/tb_monitors.sv tb/tb_generic_timer.sv tb/tb_amtgc_full.sv
vsim -voptargs=+acc work.tb_amtgc_full
do sim/wave_full.do
run -all
```

**Expected Result:**

```text
AMTGC FULL VERIFICATION: ALL PASS
```

The unified testbench reports every coverage item as `COVERED`. Any failed check terminates simulation with a `** Fatal:` message.

## Verification Configuration

| Parameter | Junction A | Junction B |
|---|---:|---:|
| Base Green / Step / Min / Max | 4 / 2 / 3 / 12 | 2 / 1 / 3 / 8 |
| Yellow Duration | 2 | 2 |
| All-Red Duration | 2 | 3 |
| Pedestrian Duration | 3 | 2 |

- Green-wave offset target: **4 clock cycles**
- `COUNTER_WIDTH`: **8**

## Verification Results

| Test | Result |
|---|---|
| Generic Timer | 12/12 checks passed |
| Timer widths | 4, 8, and 16 bits |
| Green-wave offset | Exactly 40 ns (4 cycles) |
| Unified plan scenarios | 20 |
| Additional tests | 9 |
| Total coverage scenarios | 29/29 |
| Pedestrian service | Junction A: 684 / Junction B: 657 |
| Green-wave measurements | 835, all exactly 4 cycles |
| Simultaneous arbitration rounds | 60 (A/B alternating 30/30) |
| Constrained-random simulation | 20,000 cycles |

### Mutation Testing

| Mutation | Detection |
|---|---|
| All-Red skipped | Conflicting green without All-Red gap |
| Emergency disabled | Not All-Red one clock after assertion |
| Fixed-priority arbiter | `ARBITER FAIRNESS FAIL` |
| Minimum-green clamp removed | Expected 3 cycles, observed 2 |

Mutation tests were caught successfully, and the original RTL passed after restoration.

Detailed logs: `sim/logs/task5_mutation_tests.log`

## Documentation

| File | Description |
|---|---|
| `docs/Technical_Report.md` | Technical report with 8 sections and appendices |
| `docs/Task1-4_AsBuilt_Documentation.md` | Per-task design, results, and evidence |
| `docs/diagrams/` | Block diagram, FSM diagram, Task 1 plan diagram |
| `docs/ASSUMPTIONS.md` | Design assumptions and conventions |
| `docs/BUG_LOG.md` | Defects, root causes, and fixes (BUG-01 to BUG-07) |
| `docs/Task5_Coverage_Statement.md` | Test coverage, mutation results, and limitations |
| `docs/SCREENSHOT_MANIFEST.md` | Screenshot and simulation log index |
| `docs/Self_Attestation_Checklist.md` | Delivery checklist |
| `docs/Demo_Video_Script.md` | Demonstration script |
| `docs/RTL_CHECKSUMS.txt` | SHA-256 checksums of frozen RTL |

## Training Project Lineage

This design extends the original training project:

`traffic_timer` → `generic_timer`  
`traffic_fsm_final` → 4-way `junction_controller`  
`density_logic` → clamped adaptive green timing  
Single pedestrian latch → shared round-robin arbiter  
Manual waveform testing → self-checking verification

## Frozen RTL & Reproducibility

`docs/RTL_CHECKSUMS.txt` contains SHA-256 hashes of the RTL validated during the final Task 5 run (October 8, 2026).

Verify the RTL on Windows:

```bat
certutil -hashfile rtl\junction_controller.sv SHA256
```

The delivered RTL must match the recorded checksums to preserve verification reproducibility.

## Author & Disclosure
SHREYAS K,Elevance Skills VLSI internship. AI Transparency Notice: Parts of the conceptual explanations and analogies in this documentation were generated with the assistance of [AI Tool, e.g., ChatGPT]. All AI-generated content has been human-reviewed, edited, and verified for technical accuracy.
