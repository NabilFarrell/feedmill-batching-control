# Feedmill Batching & Dosing Control System

Two-layer CODESYS control system for a feedmill batching line, with an Ignition Perspective HMI over OPC UA.

> [Bahasa Indonesia →](README_ID.md)

## Overview

A simulated feedmill batching station: **4 ingredient bins → weigh hopper (load cell) → mixer → discharge gate**. The system runs a 6-state batch cycle with recipes, safety interlocks, tolerance checking, and latched alarms, all exposed to the HMI over OPC UA.

**The defining design choice** is layer separation. The controller is written exactly as if it were running on real hardware — it only reads sensor values and writes command variables, and knows nothing about how the physics are faked. All sensor values and all flow integration live in a separate plant model. This is the same separation used in factory acceptance testing, and it means the control logic can be trusted without any knowledge of the simulation.

## Architecture

```
┌──────────────────────────────────────────────────────┐
│              CODESYS Control Win V3 (soft PLC)        │
│  Task: MainTask — IEC task, 100 ms cycle              │
│                                                      │
│  PLC_PRG                                            │
│   ├── Controller_ST   ← REAL logic                    │
│   │                     state machine, recipes,      │
│   │                     tolerance, alarms             │
│   ├── Controller_LD   ← interlocks + E-Stop latch    │
│   └── PlantModel_ST   ← FAKE physics                  │
│                         flow integration, gate        │
│                         travel delay, clamps          │
└────────────────────┬─────────────────────────────────┘
                     │ OPC UA  (opc.tcp://localhost:4840)
┌────────────────────▼─────────────────────────────────┐
│        Ignition Perspective 8.3 — HMI                │
│  ┌────────────┬────────────┬────────────────────────┐  │
│  │  Overview  │  Controls  │  History               │  │
│  │  (process  │  (operator │  (trend + batch log +  │  │
│  │   mimic)   │   inputs)  │   alarm journal)       │  │
│  └────────────┴────────────┴────────────────────────┘  │
└──────────────────────────────────────────────────────┘
```

## The Three Programs

| Program | Language | Owns |
|---|---|---|
| `Controller_ST` | Structured Text | 6-state machine, recipe selection, cumulative target logic, ±2% tolerance check, empty-bin check, alarm latching |
| `Controller_LD` | Ladder Diagram | Physical interlocks, E-Stop set/reset latch, gate mutual exclusion, actuator outputs |
| `PlantModel_ST` | Structured Text | Physics only — flow integration per bin, gate travel delay, discharge rate, clamping at zero |

The layers never mix. `Controller_ST` never touches the physics; `PlantModel_ST` never makes a control decision.

## State Machine

| State | Name | Behaviour |
|---|---|---|
| 0 | IDLE | All gates closed, mixer off. Accepts Start only when hopper < 5 kg and no alarm active |
| 1 | DOSING | Bins 1→4 in sequence. Open gate, integrate until cumulative target, close, wait for closed feedback |
| 2 | MIXING | Mixer runs for a fixed MIX_TIME |
| 3 | DISCHARGING | Mixer stays on, discharge gate opens, hopper drains to < 5 kg |
| 4 | COMPLETE | `Batch_Counter += 1`, `Total_Weight += recipe total`, 2 s hold, then auto-restart or IDLE |
| 5 | FAULT | All actuators off. Holds until the alarm is acknowledged and reset |

## Recipes

| Recipe | Bin 1 (Corn) | Bin 2 (Soybean Meal) | Bin 3 (Wheat Bran) | Bin 4 (Limestone) | Total |
|---|---|---|---|---|---|
| 1 — Poultry | 200 | 150 | 100 | 50 | **500 kg** |
| 2 — Cattle | 150 | 100 | 75 | 25 | **350 kg** |
| 3 — Supplement | 250 | 125 | 75 | 50 | **500 kg** |

`Recipe_Select` is latched at the Start rising edge — changing it mid-batch applies to the next batch only.

## HMI — Ignition Perspective

Three views, with a shared docked header providing navigation and system status.

**Overview** — process mimic with live bin levels, per-bin weights, valve state colour-coding, mixer fill, hopper weight vs target, and accumulated production. A persistent status box reports `OK` / `EMERGENCY STOP` / `FAULT`.

**Controls** — recipe selection, target batch count, auto-restart, start/stop, E-Stop, safety release, alarm reset, manual discharge.

**History** — load cell trend, batch history table, and the alarm journal.

HMI conventions applied:

- **ISA-101 colour discipline** — grey for idle equipment, colour reserved for state, red only for alarm
- **Native alarm management** — alarms are real Ignition alarm events with priority (High/Medium), acknowledgement state, and a persistent journal. `Alarm_EmptyBin` and `Alarm_Underweight` latch in the PLC and are acknowledged and cleared from the HMI
- **Per-bin fault identification** — the PLC exposes a single global `Alarm_EmptyBin` flag, so the HMI derives per-bin indication from the individual `BinLevel_n` tags against the empty threshold. This lets the operator see *which* bin ran dry, which the PLC flag alone cannot convey
- **Page-based navigation** with a docked header region, and nav buttons that highlight the active view

## HMI Migration: Node-RED → Ignition Perspective

The first version used a Node-RED dashboard as a fast prototyping layer. It was replaced with Ignition Perspective for one substantive reason:

**Node-RED cannot do alarm management.** The original dashboard rendered alarms as coloured text and a table built by hand. Ignition provides a real alarm system — priority, acknowledgement lifecycle, shelve/cleared states, and a journal — which is what industrial operators and alarm-management standards (ISA-18.2) actually expect. Everything else about the HMI also moved: ISA-101 state colour coding, per-bin fault identification, page-based navigation, and a persistent alarm banner.

The Node-RED version is retained under `v1-node-red/` and `screenshots/v1-node-red/` as the v1 prototyping layer.

## Features

- **6-state batch cycle** with transition guards and safe-state handling
- **3 recipes** with cumulative target logic across 4 bins
- **Safety interlocks** — E-Stop latching, gate mutual exclusion, mixer-gate interlock
- **Latched alarm system** — empty bin, underweight (±2% tolerance), with acknowledgement and reset
- **E-Stop software latch** simulating a hardware safety relay, with separate release and reset actions
- **Manual discharge override** for maintenance, guarded to IDLE and FAULT only
- **Auto-restart** for continuous batching until a target count
- **Batch history and alarm journal** with time, priority, and state

## Repository Structure

```
├── README.md                    ← this file
├── README_ID.md                 ← Bahasa Indonesia
├── .gitignore                   ← CODESYS build artifacts + machine settings excluded
├── code/                        ← PLC code exports (PDF) + CODESYS project source
│   ├── Controller-ST.pdf
│   ├── Controller-LD.pdf
│   ├── PlantModel-ST.pdf
│   ├── GVL.pdf
│   └── Feedmill Control.project ← CODESYS project (import to run the PLC)
├── docs/
│   ├── mock_project_1_feedmill_batching_spec.md   ← full specification
│   └── hardware-reference.md     ← variable → real hardware mapping
├── hmi/                         ← Ignition Perspective project archive
│   └── designer project export.zip
├── screenshots/
│   ├── hmi/                     ← client + designer captures of the current HMI
│   │   ├── client-*.jpg
│   │   └── designer-*.jpg
│   ├── codesys/                 ← project tree, OPC UA symbol set
│   └── v1-node-red/             ← archived Node-RED dashboard
├── video/
│   ├── feedmill-batching-demo.mp4   ← full HMI demo
│   └── v1-node-red/             ← archived v1 demos
└── v1-node-red/                 ← Node-RED flow JSON (v1 HMI layer)
    └── flows_codesys_opcua.json
```

## Demo

**Current — Ignition Perspective HMI**

| Scenario | Video |
|---|---|
| Full run: idle → dosing → mixing → discharging → complete, plus E-Stop and alarm handling | [feedmill-batching-demo.mp4](video/feedmill-batching-demo.mp4) |

**v1 — Node-RED HMI layer** (archived)

| Scenario | Video |
|---|---|
| Empty bin alarm, error status, release | [Idle → Control → Start → Empty Bin → Release Error Status](video/v1-node-red/Idle-Control-Start-Empty%20Bin-Release%20Error%20Status.mp4) |
| E-Stop activation and safety release | [Idle → Control → Start → Emergency → Release Emergency](video/v1-node-red/Idle-Control-Start-Emergency-Release%20Emergency.mp4) |

## Key Design Decisions

1. **Two-layer architecture** — Controller logic written as if on real hardware, plant model owns all physics. Same separation as FAT testing; the controller never learns it is being simulated
2. **Edge-to-latch pattern** — operator inputs (Start, Stop, Alarm_Reset) use rising-edge detection with latching to prevent repeated triggers from held buttons
3. **E-Stop in ladder, not ST** — safety latching implemented in LD for deterministic scan-cycle behaviour, with separate release and reset so restart is always a deliberate multi-step action
4. **Mixer stays ON during discharge** — prevents a self-defeating loop where `Mixer_Done` collapses when `Mixer_Activate` drops at the state transition
5. **Manual discharge as a toggle with guards** — only permitted in IDLE or FAULT, and self-stops when the hopper empties
6. **HMI derives per-bin indication, PLC owns the alarm** — the HMI is not making control decisions, only presenting `BinLevel_n` against the same threshold the PLC uses
7. **Node-RED → Ignition for alarm management** — see the migration section above

## What I Learned

- PLC state machine design with proper transition guards and safe-state handling
- Why layer separation matters: it makes the control logic independently verifiable
- OPC UA integration between CODESYS and Ignition — symbol set configuration, certificate generation, tag subscriptions
- Tag Groups (scan classes) and why the default 1000 ms rate is a latency trap for live mimic displays
- Safety interlock patterns — set/reset coils, gate mutual exclusion, deliberate reset sequencing
- Ignition alarm management — priority, acknowledgement lifecycle, and why a global PLC flag is not enough for an operator
- ISA-101 HMI colour discipline and how to encode state without decorative colour
- Perspective page architecture — Pages with mounted views vs. view swapping, and why docked regions are sized in `Size` while flex children use `position.basis`
- Debugging simulation issues — `AccessViolation`, stuck states, and self-defeating loops

## Technologies

- **PLC:** CODESYS V3.5 SP22 Patch 3, CODESYS Control Win V3 x64
- **Languages:** IEC 61131-3 Structured Text (ST) and Ladder Diagram (LD)
- **Communication:** OPC UA server, IEC Symbol Set configuration
- **HMI:** Ignition 8.3 Perspective (Maker Edition), OPC UA driver
- **Standards referenced:** ISA-101 (HMI design), ISA-18.2 (alarm management), IEC 61131-3
- **v1 prototyping:** Node-RED, node-red-dashboard, node-red-contrib-opcua

## Author

Muhammad Nabil Farrell — Electrical Engineering Fresh Graduate, aspiring Automation Engineer

- GitHub: [github.com/NabilFarrell](https://github.com/NabilFarrell)
- LinkedIn: [linkedin.com/in/nabil-farrell](https://www.linkedin.com/in/nabil-farrell/)
- Email: nabilfarrellid@gmail.com

## License

This project is for portfolio and demonstration purposes.
