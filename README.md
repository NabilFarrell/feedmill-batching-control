# Feedmill Batching Control System

Industrial feedmill batching system with PLC state machine, OPC UA communication, and Node-RED HMI dashboard.

## Overview

A complete batch processing system for animal feed production. The system controls 4 ingredient bins, a weigh hopper with load cell, a mixer, and a discharge gate. It supports 4 pre-configured recipes with automatic batch counting and target-based operation.

## Architecture

```
┌─────────────────────────────────────────────┐
│              CODESYS PLC                     │
│  ┌──────────────┐  ┌─────────────────────┐  │
│  │ Controller_ST │  │   Controller_LD     │  │
│  │ (State Mach.) │→ │ (Safety Interlocks) │  │
│  └──────────────┘  └─────────────────────┘  │
│         ↓                                    │
│  ┌──────────────┐                            │
│  │PlantModel_ST │                            │
│  │ (Physics)    │                            │
│  └──────────────┘                            │
└─────────────────┬───────────────────────────┘
                  │ OPC UA
┌─────────────────▼───────────────────────────┐
│          Node-RED Dashboard                  │
│  ┌────────┬──────────┬──────────┬─────────┐  │
│  │ Status │ Process  │ Controls │  Alarm  │  │
│  └────────┴──────────┴──────────┴─────────┘  │
└─────────────────────────────────────────────┘
```

## Features

- **6-State Machine:** IDLE → DOSING → MIXING → DISCHARGING → COMPLETE → FAULT
- **4 Recipes:** Poultry, Pig Grower, Pig Finisher, Custom
- **Safety Interlocks:** E-Stop latching, gate interlocks, mixer safety
- **Alarm System:** Empty Bin, Underweight, Emergency Stop
- **Manual Discharge:** Operator override for maintenance
- **Auto-Restart:** Continuous batching until target reached
- **Real-Time Dashboard:** Live weight, bin levels, batch status

## Repository Structure

```
├── README.md                    ← you are here
├── code/                        ← PLC code exports (PDF)
│   ├── Controller-ST.pdf
│   ├── Controller-LD.pdf
│   ├── PlantModel-ST.pdf
│   └── GVL.pdf
├── screenshots/                 ← dashboard + CODESYS screenshots
├── video/                       ← demo recordings
├── project-files/               ← Node-RED flow JSON
└── docs/                        ← specification document
```

## Demo

<!-- Add video embed or link here after uploading to YouTube/Loom -->
[Watch Demo Video →](video/)

## Screenshots

<!-- Add dashboard screenshots here -->
| IDLE | DOSING | COMPLETE | FAULT |
|------|--------|----------|-------|
| ![IDLE](screenshots/Dashboard%20Startup.png) | ![DOSING](screenshots/Dashboard%20Dosing.png) | ![COMPLETE](screenshots/Dashboard%20Complete.png) | ![FAULT](screenshots/Dashboard%20Emergency%20Stop.png) |

## Key Design Decisions

1. **Edge-to-Latch Pattern** — Operator inputs (Start, Stop, Alarm_Reset) use rising-edge detection with latching to prevent repeated triggers from held buttons
2. **E-Stop in LD** — Safety-critical latching implemented in ladder logic (not structured text) for deterministic scan-cycle behavior
3. **Mixer Stays ON During Discharge** — Prevents self-defeating loop where Mixer_Done collapses when Mixer_Activate drops at state transition
4. **Manual Discharge Toggle** — Toggle latch with guard conditions (only works in IDLE or FAULT state)

## What I Learned

- PLC state machine design with proper transition guards
- OPC UA integration between CODESYS and Node-RED
- Safety interlock design patterns (set/reset coils, gate mutual exclusion)
- Industrial HMI design with dark theme and color-coded status indicators
- Debugging simulation issues (AccessViolation, stuck states, self-defeating loops)

## Technologies

- **PLC:** CODESYS V3.5 SP22 Patch 3, CODESYS Control Win V3 x64
- **Communication:** OPC UA (Symbol Set configuration)
- **HMI:** Node-RED with node-red-dashboard, node-red-contrib-opcua
- **Language:** IEC 61131-3 Structured Text (ST), Ladder Diagram (LD)

## Author

Muhammad Nabil Farrell — Electrical Engineering Fresh Graduate

- GitHub: [github.com/NabilFarrell](https://github.com/NabilFarrell)
- LinkedIn: [linkedin.com/in/nabil-farrell](https://www.linkedin.com/in/nabil-farrell/)
- Email: nabilfarrellid@gmail.com

## License

This project is for portfolio/demonstration purposes.
